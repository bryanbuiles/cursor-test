# Spec 01 — Monorepo setup

## Objetivo

Crear el esqueleto del monorepo con pnpm workspaces: `apps/web` (Next.js + Tailwind), `apps/api` (Hono), `packages/db` y `packages/domain` (exportables, aún sin lógica de flags). Arranque verificable: health de API y página web vacía.

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. No hay specs previas. No se implementa el producto de flags todavía.

Fuente de producto (trazabilidad, no sustituye a esta spec): `docs/prds/feature-flags-mvp.md`.

**Workspace raíz:** un único repo. Package manager: **pnpm**.

**Puertos (fijos de aquí en adelante):**

| App | Path | Puerto | Rol |
| --- | --- | --- | --- |
| Web | `apps/web` | `3000` | Next.js App Router + Tailwind |
| API | `apps/api` | `8787` | Hono + Node |

**CORS:** la API debe permitir origen `http://localhost:3000`.

**Packages:**

- `packages/db`: paquete `@ff/db`. En esta spec solo `package.json`, `tsconfig.json` y `src/index.ts` que exporta un placeholder (`export const dbPackage = "@ff/db"`). Sin Drizzle.
- `packages/domain`: paquete `@ff/domain`. Placeholder (`export const domainPackage = "@ff/domain"`). Sin evaluador.

**TypeScript** en todos los paquetes. `strict: true`.

## Alcance

### In scope

- `pnpm-workspace.yaml` con `apps/*` y `packages/*`.
- `package.json` raíz con scripts `dev`, `dev:web`, `dev:api`.
- Next.js (App Router) + Tailwind CSS en `apps/web`.
- Hono en `apps/api` con `GET /health` → `{ "ok": true }`.
- Packages `@ff/db` y `@ff/domain` compilables/exportables.
- `.gitignore` que ignore `node_modules`, `.next`, `dist` y `*.sqlite` / `packages/db/data/`.
- README mínimo en la raíz: cómo `pnpm install` y `pnpm dev`.

### Out of scope

- Drizzle, migraciones, SQLite, seed.
- Vitest.
- Auth, cookies, OAuth, roles.
- CRUD de flags, evaluador, UI de dashboard.
- Turbo (opcional; no es requisito). Si se añade, no sustituye los scripts pnpm.

## Tareas

1. Inicializar pnpm en la raíz: `pnpm-workspace.yaml` y `package.json` (`private: true`, `packageManager` pnpm).
2. Crear `apps/web` con Next.js App Router, TypeScript y Tailwind. Página `/` con un título estático, p.ej. “Feature Flags” (sin listar flags).
3. Configurar `apps/web` para llamar a la API en `http://localhost:8787` vía env `NEXT_PUBLIC_API_URL` (default ese valor). En esta spec no hace falta fetch real.
4. Crear `apps/api` con Hono, script `dev` que escucha en `0.0.0.0:8787`. Implementar `GET /health` y CORS para `http://localhost:3000`.
5. Crear `packages/db` (`name: "@ff/db"`) y `packages/domain` (`name: "@ff/domain"`) con `exports` a `src/index.ts`. La API y/o la web pueden depender de ellos (aunque aún no los usen).
6. Alinear `tsconfig` (paths o project references simples). Un `pnpm --filter api exec tsc --noEmit` y equivalente en web/domain/db no debe fallar.
7. Scripts raíz: `dev` arranca web y api en paralelo (p.ej. `pnpm -r --parallel --filter web --filter api dev` o dos procesos documentados).
8. `.gitignore` como en alcance.
9. README con los comandos de arranque.

## Criterios de aceptación

1. `pnpm install` termina sin error en un checkout limpio.
2. `pnpm --filter api dev` (o el script documentado) deja `GET http://localhost:8787/health` respondiendo **200** y body JSON `{ "ok": true }`.
3. `pnpm --filter web dev` sirve `http://localhost:3000` con status 200 y el título de la app visible.
4. Existen exactamente estos paquetes de trabajo: `apps/web`, `apps/api`, `packages/db`, `packages/domain`.
5. No hay schema SQL, tests Vitest, ni rutas `/v1/flags` o `/login`.
6. `.gitignore` cubre artefactos de Next y archivos sqlite.

## Notas técnicas

- **API_PORT:** `8787`. **WEB_PORT:** `3000`.
- **NEXT_PUBLIC_API_URL:** `http://localhost:8787`.
- Node 20+ recomendado.
- Nombre interno de paquetes: `@ff/db`, `@ff/domain`, apps pueden llamarse `web` y `api` en `package.json`.
- Esta spec no persiste datos. El archivo sqlite se introducirá en spec 03 en `packages/db/data/app.sqlite`.
