# Spec 11 — History and review

## Objetivo

Listar la auditoría básica (quién / qué / cuándo) para revisar cambios de flags y targeting. Cubre **RF-14** lectura y CA-9.

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume spec **01–10**: las escrituras a `audit_events` ya ocurren en la API (create, patch, env, overrides) con `actor` igual al usuario demo (`demo`).

**Tabla `audit_events` (spec 03):**

| Columna | Tipo |
| --- | --- |
| `id` | PK |
| `at` | timestamp |
| `actor` | text |
| `flag_key` | text |
| `environment` | text nullable |
| `action` | text |
| `summary` | text |

**Acciones conocidas:** `flag.create` | `flag.update` | `flag.environment.update` | `flag.override.upsert` | `flag.override.delete` | `seed`.

**Auth:** `GET /v1/audit` es de **operador** → requiere cookie `ff_session` (como `/v1/flags`). Evaluate sigue público.

**Puertos:** 3000 / 8787. Login `demo`/`demo`.

## Alcance

### In scope

- `GET /v1/audit?flag_key=&limit=`  
  - `flag_key` opcional (filtra).  
  - `limit` default 50, max 200.  
  - Orden `at` DESC.
- Página `/history` autenticada: tabla `at`, `actor`, `flag_key`, `environment`, `action`, `summary`. Filtro por flag (query o select).
- Link en nav: Historial.
- Test API: tras crear una flag, GET audit incluye `flag.create`.

### Out of scope

- Diff rico / rollback / “revert this change”.
- Paginación cursor compleja (offset simple opcional).
- Export CSV, alertas.
- OAuth/roles.
- Reescribir el modelo de audit.

## Tareas

1. Handler `GET /v1/audit` + middleware de sesión.
2. Response:

```json
{
  "events": [
    {
      "id": 1,
      "at": "<iso>",
      "actor": "demo",
      "flag_key": "checkout-v2",
      "environment": "prod",
      "action": "flag.environment.update",
      "summary": "enabled_global=true rollout_percent=50"
    }
  ]
}
```

3. Página `/history` que consume el GET (credentials).
4. Filtro `?flag_key=` desde un select de flags o input.
5. Confirmar que un toggle de spec 07/08 aparece como último evento tras recargar historial.
6. Test: 401 sin cookie; 200 con sesión.

## Criterios de aceptación

1. Con sesión, `/history` muestra al menos el evento de seed o los de uso reciente. CA-9.
2. Crear o cambiar una flag y recargar historial: aparece el evento nuevo con actor `demo`, `flag_key`, timestamp. **RF-14**.
3. Filtrar por `checkout-v2` oculta eventos de otras keys.
4. Sin sesión: `GET /v1/audit` → 401 y `/history` → `/login`.
5. No hay UI de rollback ni de permisos.
6. `pnpm --filter api test` incluye el caso de audit autenticado.

## Notas técnicas

- No evaluar flags en esta página.
- `environment` null para acciones globales (`flag.create`, `flag.update`).
- `summary` es texto libre corto generado en el write path (spec 04); esta spec solo lee.
- Un solo writer SQLite; lecturas de audit en el mismo proceso API.
