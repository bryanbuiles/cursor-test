# Spec 07 — Create and edit flag

## Objetivo

Permitir crear una flag y editar metadatos + on/off por ambiente desde la web, sin cambiar `flag_key`. Cubre **RF-02, RF-04, RF-05** en UI.

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume spec **01–06** (dashboard listo, login cookie, API CRUD).

**Puertos:** web `3000`, API `8787`. `credentials: "include"`.

**Contratos (spec 04, no modificar):**

- `POST /v1/flags` `{ "flag_key", "name", "description?", "critical" }` → 201 Flag; duplicado **409**.
- `PATCH /v1/flags/:key` `{ "name?", "description?", "critical?" }` → 200. `flag_key` inmutable.
- `PUT /v1/flags/:key/environments/:env` `{ "enabled_global", "rollout_percent" }` → 200.
- Defaults al crear: tres ambientes `enabled_global: false`, `rollout_percent: 0`.

Al hacer toggle on/off, enviar el `rollout_percent` **actual** (GET detalle antes o conservarlo en estado) para no resetear el % a 0 por accidente.

**Rutas web:** `/flags/new`, `/flags/[key]/edit`. Dashboard: botones “Nueva flag” y link editar por fila.

**Auth:** igual que dashboard; sin sesión → `/login`.

## Alcance

### In scope

- Formulario crear: `flag_key` (slug, p.ej. `^[a-z0-9]+(?:-[a-z0-9]+)*$` recomendado), `name`, `description`, checkbox `critical`.
- Formulario editar: `flag_key` visible **disabled**; `name`, `description`, `critical`; tres toggles `enabled_global` para `dev`/`staging`/`prod`.
- Mostrar error de duplicado (`flag_key` existente) en el formulario.
- Tras crear, redirect a `/` o a editar.
- Links desde el listado (spec 06).

### Out of scope

- Inputs de `rollout_percent` y CRUD de overrides (spec 08). En edit, al PUT env hay que **preservar** el percent existente.
- Evaluador, simulador, historial.
- Borrar flags (no está en el PRD MVP).

## Tareas

1. Página `/flags/new` + submit POST.
2. Página `/flags/[key]/edit` + GET detalle + PATCH metadatos.
3. Toggles de ambiente: PUT environment con `enabled_global` nuevo y `rollout_percent` vigente.
4. Validación client mínima (key vacía, name vacío) además de errores API.
5. Actualizar dashboard con CTA crear y link editar.
6. No hace falta test Playwright; verificar manualmente o con tests de componentes si ya hay Vitest en web (no obligatorio).

## Criterios de aceptación

1. Crear `promo-banner` con nombre y `critical` false; aparece en el dashboard. **RF-02**.
2. Repetir el mismo `flag_key` muestra error claro y no hay segunda fila.
3. Editar nombre y `critical` persiste tras recargar. El campo key no es editable. **RF-04**.
4. En prod, pasar `enabled_global` de off a on (o viceversa) se ve en el dashboard sin redeploy. **RF-05**.
5. El `rollout_percent` de un ambiente no cambia a 0 solo por toglear enabled (a menos que ya fuera 0).
6. Sin sesión, `/flags/new` redirige a login.

## Notas técnicas

- `flag_key` inmutable: no enviar `flag_key` en PATCH; input disabled.
- PUT environment requiere **ambos** campos `enabled_global` y `rollout_percent` (contrato spec 04).
- Auditoría la escribe la API; la UI no inserta `audit_events`.
