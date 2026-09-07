# Spec 10 — Simulator

## Objetivo

Página interna autenticada para **simular** una evaluación: el operador envía `flag_key` + ambiente + `empresa_id` y ve `enabled` y `reason` llamando a `POST /v1/evaluate`. No está en el PRD; es herramienta de verificación de CA-2..5 vía UI.

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume spec **01–09**.

**Evaluate (sin cookie en la API):**

`POST http://localhost:8787/v1/evaluate`

```json
{ "flag_key": "checkout-v2", "environment": "prod", "empresa_id": "acme" }
```

```json
{ "enabled": true, "reason": "company_override" }
```

`reason`: `environment_off` | `company_override` | `rollout` | `fail_closed` | `critical_cache`.

**La página SÍ exige sesión** (solo operadores). El browser llama a evaluate; no hace falta cookie para ese POST, pero la ruta `/simulator` redirige a `/login` si `GET /v1/auth/me` es 401.

**Precedencia a mostrar como ayuda (igual spec 09):**

1. `enabled_global` false → off (`environment_off`)
2. override empresa
3. rollout FNV-1a `empresa_id + ":" + flag_key`, `bucket % 100`, on si `bucket < percent`

**Seed útil:** `checkout-v2` prod 50% + overrides acme/globex; `payments-kill` critical.

**Puertos:** web 3000, API 8787, `credentials: "include"` para me/flags; evaluate puede ir sin credenciales.

## Alcance

### In scope

- Ruta `/simulator` protegida.
- Formulario: flag (select de `GET /v1/flags` o input de key), select ambiente `dev|staging|prod`, input `empresa_id`.
- Submit → `POST /v1/evaluate` → mostrar `enabled` (sí/no) y `reason`.
- Link en el header/nav desde el dashboard.
- Casos documentados en la propia página (texto): override acme, globex, ambiente off, %.

### Out of scope

- SDK, llamadas desde apps de clientes.
- Historial de simulaciones persistido.
- Reimplementar el hash en el cliente como fuente de verdad (opcional mostrar bucket solo como debug; no sustituir a la API).
- OAuth.

## Tareas

1. Página `/simulator` + nav.
2. Cargar lista de flags para el select (requiere cookie).
3. Submit evaluate y panel de resultado.
4. Manejar `fail_closed` de forma visible (no como crash).
5. Copy breve de precedencia.

## Criterios de aceptación

1. Logueado, simular `checkout-v2` / `prod` / `acme` → enabled true, reason `company_override` (con seed).
2. Misma flag/env `globex` → enabled false, `company_override`.
3. Poner la flag off en prod (`enabled_global: false`) y simular acme → `enabled` false, `environment_off`.
4. Empresa **sin** override, % 0 → false `rollout`; % 100 → true `rollout`.
5. Flag inventada → `enabled` false, `fail_closed`.
6. Sin sesión, `/simulator` → `/login`.
7. Recargar el mismo formulario dos veces da el mismo `enabled` (estabilidad).

## Notas técnicas

- El simulador **no** es el evaluador: solo un cliente del endpoint.
- No persistir simulaciones en SQLite.
- Header/nav: Dashboard, (flags), Simulador; Historial vendrá en spec 11.
