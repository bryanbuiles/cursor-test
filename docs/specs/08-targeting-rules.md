# Spec 08 — Targeting rules

## Objetivo

UI para rollout porcentual y overrides por empresa, por ambiente, con el texto de precedencia. Cubre **RF-06, RF-07** y la parte UI de **RF-10** (el operador ve que el override es independiente del %).

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume spec **01–07**.

**API (spec 04) + cookie (spec 05):**

- `PUT /v1/flags/:key/environments/:env` `{ "enabled_global": boolean, "rollout_percent": 0-100 }`
- `PUT /v1/flags/:key/environments/:env/overrides/:empresaId` `{ "enabled": boolean }`
- `DELETE /v1/flags/:key/environments/:env/overrides/:empresaId`

**GET detalle** incluye `environments[].overrides[]` con `{ empresa_id, enabled }`.

**Precedencia (mostrar en UI, no calcular en el cliente para producción):**

1. Si `enabled_global` es false en ese ambiente → todas las empresas off (overrides y % no se evalúan en spec 09).
2. Override por `empresa_id` on/off; esa empresa **no** entra al %.
3. Rollout porcentual 0–100 sobre empresas sin override.

**Hash (solo informativo aquí; implementación spec 09):** FNV-1a 32-bit de `empresa_id + ":" + flag_key`, `bucket = hash % 100`, enabled si `bucket < rollout_percent`.

**Dónde vive la UI:** en `/flags/[key]/edit` (o `/flags/[key]` detalle) sección por ambiente.

## Alcance

### In scope

- Input entero 0–100 para `%` por ambiente; rechazar fuera de rango (UI + API 400).
- Lista de overrides: añadir (`empresa_id` + on/off), editar enabled, eliminar.
- Copy visible de precedencia (ambiente off → override → %).
- Nota: cambiar el % no debería mostrarse como “cambia a acme” si acme tiene override (texto de ayuda RF-10).
- Validar `empresa_id` no vacío; un override por empresa y ambiente (la API upsert).

### Out of scope

- Endpoint evaluate y simulador (09–10).
- Catálogo de empresas (solo ids libres).
- A/B, variantes, hash por usuario/request.

## Tareas

1. En la pantalla de la flag, bloque por `dev` / `staging` / `prod`.
2. Campo `%` + guardar (PUT environment conservando `enabled_global`).
3. Formulario override: `empresa_id`, select/checkbox enabled, submit PUT.
4. Botón eliminar → DELETE.
5. Mostrar error API si percent inválido.
6. Texto de ayuda de precedencia y de overrides vs %.

## Criterios de aceptación

1. Poner prod en 75 y recargar: se ve 75. Valor 101 no se persiste. **RF-06**.
2. Crear override `acme` on y `globex` off en prod; recargar lista. **RF-07**.
3. Eliminar override `globex`; deja de aparecer; `acme` permanece.
4. Texto en pantalla explica que una empresa con override no entra al rollout.
5. `enabled_global: false` se sigue pudiendo setear (spec 07) y el copy dice que eso apaga todo el ambiente.
6. Sin sesión, no se puede mutar (401 / redirect login).

## Notas técnicas

- No evaluar el hash en el browser para decidir on/off de clientes reales.
- `empresa_id` es string estable (`acme`, `globex` en seed).
- PUT env siempre envía el par `enabled_global` + `rollout_percent`.
- Writes siguen generando `audit_events` en API (`flag.environment.update`, `flag.override.upsert`, `flag.override.delete`).
