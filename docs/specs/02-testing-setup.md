# Spec 02 — Testing setup

## Objetivo

Dejar Vitest operable en `apps/api`, `packages/domain` y `packages/db`, con un `pnpm test` en la raíz que ejecute esos suites. Incluir un test de humo de `GET /health` y un test dummy de domain.

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume que **spec 01** ya está hecha.

**Artefactos que deben existir:**

- Monorepo pnpm: `apps/web`, `apps/api`, `packages/db` (`@ff/db`), `packages/domain` (`@ff/domain`).
- API Hono en puerto **8787** con `GET /health` → `{ "ok": true }`.
- Web Next.js en **3000**.

**Cómo arrancar (recap):** `pnpm install`; API `pnpm --filter api dev`; web `pnpm --filter web dev`.

No se testea aún el dominio de flags. El test de domain es un placeholder sobre el export existente (p.ej. `domainPackage === "@ff/domain"`).

Trazabilidad: infraestructura; no cubre RFs del PRD.

## Alcance

### In scope

- Vitest en `apps/api`, `packages/domain`, `packages/db`.
- Script `test` en cada uno de esos `package.json` y script `test` en la raíz (`pnpm -r --filter api --filter @ff/domain --filter @ff/db test` o equivalente).
- Test de API: importar la app Hono (sin levantar puerto, usando `app.request` o similar) y asertar `GET /health` → 200 `{ ok: true }`.
- Test dummy en `@ff/domain` y uno trivial en `@ff/db` (p.ej. el placeholder export).
- `vitest.config.ts` (o en `vite.config.ts`) por paquete, environment `node`.

### Out of scope

- Playwright / tests E2E de browser.
- Tests de schema, CRUD, login, evaluador.
- Cobertura mínima obligatoria (nyc/c8) o CI GitHub Actions (opcional, no requerido).
- Tests en `apps/web` (opcional; si se añaden, no bloquean esta spec).

## Tareas

1. Añadir `vitest` (y tipos si hace falta) como devDependency en `apps/api`, `packages/domain`, `packages/db`.
2. Configurar Vitest en cada uno (`include: ["src/**/*.test.ts"]` o `**/*.test.ts`).
3. Extraer o exportar la instancia Hono (`export { app }` desde p.ej. `apps/api/src/app.ts`) para testear sin `listen`.
4. Crear `apps/api/src/health.test.ts` (o equivalente) que haga `app.request("/health")`.
5. Crear `packages/domain/src/index.test.ts` y `packages/db/src/index.test.ts` con una aserción trivial.
6. Cablear `pnpm test` en la raíz.
7. Documentar en README: `pnpm test`.

## Criterios de aceptación

1. `pnpm test` en la raíz termina con **exit code 0**.
2. El suite de API incluye un test que falla si `/health` no devuelve 200 y `{ ok: true }`.
3. Existen al menos un archivo `*.test.ts` en `@ff/domain` y uno en `@ff/db`.
4. No se introducen tests de feature flags, SQLite real, ni Playwright.
5. Los tests de API **no** requieren que el proceso esté escuchando en 8787 (app in-process).

## Notas técnicas

- **Puertos:** 8787 API, 3000 web (no usados por los tests de esta spec si se usa `app.request`).
- **Comando canónico:** `pnpm test`.
- Filtro útil: `pnpm --filter api test`.
- No usar `--watch` en el script `test` de CI/raíz; `vitest run`.
- El evaluador y FNV-1a se testean en spec 09, no aquí.
