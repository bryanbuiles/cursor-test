# Spec 04 — API feature flags

## Objetivo

Exponer en Hono el CRUD de flags, estado por ambiente (on/off + %), overrides por empresa y escritura de auditoría. Sin login todavía (middleware en spec 05). Cubre **RF-02 a RF-07** a nivel API y la **escritura** de **RF-14**.

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume spec **01–03**.

**Artefactos:** API Hono exportada como `app`, SQLite `packages/db/data/app.sqlite`, tablas `flags`, `flag_environments`, `company_overrides`, `audit_events`, seed opcional.

**Puertos:** API `8787`, web `3000`. CORS `http://localhost:3000`.

**Arranque:** migrate + seed (spec 03); `pnpm --filter api dev`. Tests: `pnpm --filter api test` con sqlite **temporal**, no el de dev.

**Auth:** no hay. Actor de auditoría: literal `"demo"` (placeholder hasta spec 05).

**No implementar** `POST /v1/evaluate` (spec 09).

## Alcance

### In scope

- Rutas y JSON definidos en Notas técnicas.
- Validación: `flag_key` único; `environment` ∈ {`dev`,`staging`,`prod`}; `rollout_percent` entero 0–100; un override por (`flag`, `env`, `empresa_id`).
- Al **crear** una flag, crear las 3 filas `flag_environments` con default `enabled_global: false`, `rollout_percent: 0` (seguro / fail-closed).
- `flag_key` **no** se puede cambiar en PATCH.
- Cada escritura (create, patch, put env, put/delete override) inserta `audit_events`.
- Tests de integración Hono + DB temp: create duplicado → error; percent 101 → 400; override upsert; delete override.

### Out of scope

- Cookie, `/v1/auth/*`, 401.
- Evaluador, cache, hash FNV-1a.
- UI Next.js.
- `GET /v1/audit` (lectura para UI: spec 11). Sí se **escribe** audit aquí.
- OAuth, roles.

## Tareas

1. Conectar `apps/api` a `@ff/db` (`getDb()`). En tests, inyectar path sqlite temp + migrate.
2. Implementar handlers y montar bajo `/v1`.
3. Códigos: 200/201 OK; 400 validación; 404 flag o override inexistente (delete); 409 `flag_key` duplicado.
4. Helper `writeAudit({ actor: "demo", flag_key, environment, action, summary })`.
5. Tests nombrados sugeridos: `creates flag with three environments`, `rejects duplicate flag_key`, `rejects percent out of range`, `upserts company override`, `deletes override`.

## Criterios de aceptación

1. `POST /v1/flags` con body válido crea la flag y 3 ambientes; GET posterior los devuelve. **RF-02**.
2. Segundo POST con el mismo `flag_key` → **409** (o 400 documentado) y no duplica filas.
3. `PATCH /v1/flags/:key` cambia `name`, `description`, `critical`; ignorar o rechazar cambios a `flag_key`. **RF-04**.
4. `PUT .../environments/prod` con `{ "enabled_global": true, "rollout_percent": 50 }` persiste. Percent `-1` o `101` → **400**. **RF-05, RF-06**.
5. `PUT .../overrides/acme` con `{ "enabled": true }` upsert; un solo row por terna. `DELETE` lo elimina. **RF-07**.
6. Tras cada escritura hay una fila nueva en `audit_events` con `actor: "demo"`. **RF-14 write**.
7. `pnpm --filter api test` pasa incluyendo los casos de validación.
8. No hay `/v1/evaluate` ni `/v1/auth`.

## Notas técnicas

### Contratos JSON (canónicos; no cambiar en specs 05–11)

**EnvironmentState**

```json
{
  "environment": "prod",
  "enabled_global": true,
  "rollout_percent": 50,
  "overrides": [
    { "empresa_id": "acme", "enabled": true }
  ]
}
```

En **listados**, `overrides` puede omitirse y usarse `overrides_count: number`.

**Flag (detalle GET)**

```json
{
  "flag_key": "checkout-v2",
  "name": "Checkout v2",
  "description": "",
  "critical": false,
  "created_at": "<iso>",
  "updated_at": "<iso>",
  "environments": [
    { "environment": "dev", "enabled_global": true, "rollout_percent": 100, "overrides": [] },
    { "environment": "staging", "enabled_global": true, "rollout_percent": 100, "overrides": [] },
    { "environment": "prod", "enabled_global": true, "rollout_percent": 50, "overrides": [
      { "empresa_id": "acme", "enabled": true },
      { "empresa_id": "globex", "enabled": false }
    ]}
  ]
}
```

**POST `/v1/flags`** request: `{ "flag_key": "string", "name": "string", "description": "string?", "critical": false }`  
Response: Flag detalle **201**.

**GET `/v1/flags`** response: `{ "flags": [ Flag... ] }` (listado puede usar `overrides_count` por env).

**GET `/v1/flags/:key`** → Flag detalle **200** o **404**.

**PATCH `/v1/flags/:key`** request: `{ "name"?: string, "description"?: string, "critical"?: boolean }` → Flag **200**.

**PUT `/v1/flags/:key/environments/:env`**  
`:env` ∈ `dev|staging|prod`.  
Request: `{ "enabled_global": boolean, "rollout_percent": number }` → EnvironmentState **200**.

**PUT `/v1/flags/:key/environments/:env/overrides/:empresaId`**  
Request: `{ "enabled": boolean }` → `{ "empresa_id": string, "enabled": boolean }` **200**.

**DELETE** misma ruta de override → **204** o **200** `{ "ok": true }`.

### Acciones de audit (sugeridas)

`flag.create` | `flag.update` | `flag.environment.update` | `flag.override.upsert` | `flag.override.delete`

### Defaults al crear

`enabled_global: false`, `rollout_percent: 0` en los tres ambientes.
