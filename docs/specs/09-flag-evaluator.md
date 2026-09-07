# Spec 09 — Flag evaluator

## Objetivo

Implementar el evaluador puro en `@ff/domain`, el endpoint interno `POST /v1/evaluate` y cache 30–60s. Cubre **RF-08 a RF-13** y CA-2 a CA-7.

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume spec **01–08** (schema, API flags con sesión, seed).

**Este endpoint NO exige cookie** (consumo interno de servicios). `/v1/flags*` sigue exigiendo sesión.

**Tablas:** `flags`, `flag_environments`, `company_overrides`.

**Fail-closed:** flag inexistente, body inválido, error de DB → `{ "enabled": false, "reason": "fail_closed" }`. Flag `critical`: si hay valor en cache de memoria para esa terna, devolverlo con `reason: "critical_cache"`; si no hay cache → `false` + log (no fail-open).

**Hash canónico (no cambiar sin migración; fuera de MVP):**

1. `input = empresa_id + ":" + flag_key` (UTF-8).
2. FNV-1a **32-bit** (offset `2166136261`, prime `16777619`).
3. `bucket = hash % 100` (hash tratado como uint32).
4. En rollout: `enabled = bucket < rollout_percent`.
5. `% = 0` → siempre false; `% = 100` → siempre true (para empresas sin override y con ambiente on).

**Precedencia RF-08:**

1. Flag no existe → fail_closed.
2. Ambiente `enabled_global === false` → `enabled: false`, `reason: "environment_off"` (no mirar override ni %).
3. Existe override para `empresa_id` → ese boolean, `reason: "company_override"`.
4. Else rollout, `reason: "rollout"`.

Empresas con override **no** usan el hash (**RF-10**).

**Cache:** memoria de proceso, TTL 30–60s (elegir p.ej. 30s), key `(flag_key, environment, empresa_id)` → `{ enabled, reason, storedAt }`. Invalidar **todo** el cache de esa flag (o global) en escrituras de flags/env/overrides para cumplir SLA &lt; 60s de forma agresiva; si solo TTL, el peor caso debe ser ≤ 60s (**RF-13**).

## Alcance

### In scope

- `evaluateFlag(input, snapshot)` **puro** en `@ff/domain` (sin I/O). Tests unitarios de precedencia, estabilidad, 0/100, override vs %.
- Cargar snapshot desde SQLite en la API y llamar domain.
- `POST /v1/evaluate` público (sin sesión).
- Cache + fail-closed + log en stderr/console en fallos y en critical sin cache.
- Test de distribución aproximada: 200 `empresa_id` distintos, % 50, flag ambiente on sin overrides: proporción on entre ~35% y ~65% (no exigir 50.0 exacto).
- Test: misma terna dos veces → mismo `enabled` (**RF-09**).

### Out of scope

- UI simulador (spec 10).
- SDK público.
- Cambiar el algoritmo de hash.
- Auth en evaluate.

## Tareas

1. `packages/domain/src/hash.ts` + `evaluateFlag.ts` + tipos `EvaluateInput`, `EvaluateResult`, `FlagSnapshot`.
2. Tests Vitest en domain (tabla de casos).
3. Servicio API: leer DB → snapshot → evaluate → cache.
4. Invalidar cache en handlers de spec 04 (create/patch/put/delete).
5. `POST /v1/evaluate` validar body: `flag_key` string, `environment` enum, `empresa_id` string no vacío; si no, 200 fail_closed **o** 400 + body fail_closed. **Preferir 200 `{ enabled: false, reason: "fail_closed" }`** para que el consumidor no trate 400 como “on”.
6. Tests API: seed scenarios CA-2..7.

## Criterios de aceptación

1. Ambiente `dev` on y `prod` off, misma empresa, misma flag: evaluate respeta ambiente. CA-2. **RF-08**.
2. Override acme on y globex off en prod, % 0 o 100: acme true, globex false. CA-3. **RF-10**.
3. Sin overrides, % 0 todas false; % 100 todas true; % 50 estable al reevaluar. CA-4. **RF-09**.
4. Empresa con override no cambia al pasar % de 10 a 90. CA-5.
5. Tras cambiar % u override, evaluate refleja el cambio en **&lt; 60s** (con invalidación, inmediato). CA-6. **RF-13**.
6. Flag inexistente o DB caída (simular en test) → `enabled: false`. CA-7. **RF-12**.
7. `POST /v1/evaluate` **sin** cookie funciona. **RF-11**.
8. `pnpm --filter @ff/domain test` y tests nuevos de API pasan.

## Notas técnicas

**POST `/v1/evaluate`**

Request:

```json
{
  "flag_key": "checkout-v2",
  "environment": "prod",
  "empresa_id": "acme"
}
```

Response **siempre** (salvo 5xx inesperado):

```json
{
  "enabled": true,
  "reason": "company_override"
}
```

`reason` ∈ `environment_off` | `company_override` | `rollout` | `fail_closed` | `critical_cache`.

**FlagSnapshot** (para la función pura): `{ flag_key, critical, enabled_global, rollout_percent, override: boolean | null }` donde `override` es `null` si no hay fila.

La web no llama a evaluate hasta spec 10. Servicios internos solo envían la terna, no reglas.
