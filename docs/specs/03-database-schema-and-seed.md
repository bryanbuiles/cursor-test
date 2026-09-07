# Spec 03 — Database schema and seed

## Objetivo

Definir el schema Drizzle sobre SQLite/libSQL local, migraciones y un seed reproducible (flags, ambientes, overrides, un evento de auditoría). Cumple **RF-15** y el criterio de aceptación MVP de persistir tras reinicio (CA-8 a nivel de datos).

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume **spec 01 y 02** hechas: monorepo pnpm, `@ff/db`, `@ff/domain`, Vitest, API `/health`.

**No hay HTTP de flags todavía** (spec 04). Esta spec solo persiste.

**Archivo SQLite (fijo):** `packages/db/data/app.sqlite`  
**Env:** `DATABASE_URL=file:./data/app.sqlite` relativo a `packages/db`, o path absoluto equivalente. Un solo writer.

**Ambientes (enum de producto):** `dev` | `staging` | `prod`.

**Cómo arrancar (recap):** `pnpm install`; scripts nuevos `db:migrate` y `db:seed` (ver tareas).

Trazabilidad: **RF-15**, CA-8 (persistencia). No cubre RF-01..14 de API/UI.

## Alcance

### In scope

- Drizzle ORM + driver SQLite local (`better-sqlite3` **o** `@libsql/client` en modo archivo). Elegir uno y usarlo de forma consistente.
- Tablas: `flags`, `flag_environments`, `company_overrides`, `audit_events` (nombres y columnas exactos abajo).
- Migraciones versionadas (Drizzle kit).
- Seed idempotente o “wipe + seed” documentado.
- Script para abrir DB y comprobar que el seed está.
- Test Vitest en `@ff/db` que aplica schema en un sqlite **temporal** y verifica constraints básicas (flag_key unique).

### Out of scope

- Rutas HTTP `/v1/*`.
- Login.
- Función `evaluateFlag`.
- UI.
- Réplica remota, Turso cloud, Postgres.

## Tareas

1. Dependencias en `@ff/db`: `drizzle-orm`, drizzle-kit, driver sqlite/libsql.
2. Schema en `packages/db/src/schema.ts` (o `src/schema/index.ts`) con:

**`flags`**

| Columna | Tipo | Notas |
| --- | --- | --- |
| `id` | integer PK autoincrement | |
| `flag_key` | text unique not null | inmutable a nivel producto |
| `name` | text not null | |
| `description` | text | nullable o `''` |
| `critical` | integer/boolean not null default false | |
| `created_at` | text/integer not null | ISO o unix; documentar |
| `updated_at` | text/integer not null | |

**`flag_environments`**

| Columna | Tipo | Notas |
| --- | --- | --- |
| `id` | integer PK | |
| `flag_id` | FK → flags.id | on delete cascade |
| `environment` | text not null | solo `dev`,`staging`,`prod` |
| `enabled_global` | boolean not null | si false, evaluador (spec 09) corta a off |
| `rollout_percent` | integer not null | 0–100; check si el driver lo permite |
| unique(`flag_id`, `environment`) | | |

**`company_overrides`**

| Columna | Tipo | Notas |
| --- | --- | --- |
| `id` | integer PK | |
| `flag_id` | FK → flags.id | cascade |
| `environment` | text not null | mismo enum |
| `empresa_id` | text not null | id de tenant; no hay catálogo rico |
| `enabled` | boolean not null | |
| unique(`flag_id`, `environment`, `empresa_id`) | | |

**`audit_events`**

| Columna | Tipo | Notas |
| --- | --- | --- |
| `id` | integer PK | |
| `at` | timestamp not null | |
| `actor` | text not null | seed: `demo` |
| `flag_key` | text not null | |
| `environment` | text | nullable |
| `action` | text not null | p.ej. `flag.create` |
| `summary` | text not null | |

3. `packages/db/src/client.ts`: abre el archivo `data/app.sqlite` (crear directorio `data/` si no existe).
4. `drizzle.config.ts` y `pnpm --filter @ff/db db:migrate` (nombre exacto a documentar en README).
5. Seed `packages/db/src/seed.ts`:

   - Flag `checkout-v2` (`critical: false`): `dev` enabled 100%, `staging` enabled 100%, `prod` `enabled_global: true`, `rollout_percent: 50`. Overrides en **prod**: `acme` → enabled true, `globex` → enabled false.
   - Flag `payments-kill` (`critical: true`): los tres ambientes `enabled_global: true`, `rollout_percent: 0` (sin overrides).
   - Al crear cada flag, insertar las **tres** filas de `flag_environments`.
   - Un `audit_events` de ejemplo: actor `demo`, action `seed`, summary `initial seed`.

6. Exportar schema y `getDb()` desde `@ff/db`.
7. Test: sqlite en tmp, migrate, insertar dos flags con el mismo `flag_key` → el segundo falla.
8. `.gitignore` ya debe ignorar `packages/db/data/` (si spec 01 no lo hizo, añadirlo).

## Criterios de aceptación

1. `pnpm --filter @ff/db db:migrate` crea/actualiza `packages/db/data/app.sqlite` sin error.
2. `pnpm --filter @ff/db db:seed` deja exactamente **2** flags con keys `checkout-v2` y `payments-kill`, **6** filas de `flag_environments` (2×3), **2** overrides en prod para `checkout-v2` (`acme` on, `globex` off), y **≥1** fila en `audit_events`.
3. Borrar el proceso Node y volver a abrir el mismo archivo sqlite conserva esas filas (CA-8 a nivel DB).
4. `pnpm --filter @ff/db test` pasa, incluyendo unicidad de `flag_key`.
5. No existen rutas HTTP nuevas de flags.

## Notas técnicas

- **DATABASE_URL** (ejemplo): `file:/home/.../packages/db/data/app.sqlite` o relativo documentado.
- No hay tabla `companies`; solo `empresa_id` en overrides.
- `rollout_percent` es de **empresas**, no de requests (el hash se implementa en spec 09).
- Actor de auditoría en seed: `demo` (el login real es spec 05).
- Tests de `@ff/db` **no** deben escribir en `app.sqlite` de desarrollo; usar archivo temp.
