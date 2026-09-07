# Spec 06 — Dashboard list

## Objetivo

Pintar en Next.js el listado de flags para el operador autenticado: estado por ambiente y resumen de targeting (% y número de overrides). Cubre **RF-03**.

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume spec **01–05**.

**API:** `GET /v1/flags` requiere cookie `ff_session`. Login `demo`/`demo` en `POST http://localhost:8787/v1/auth/login`.

**Puertos:** web `3000`, API `8787`. `NEXT_PUBLIC_API_URL=http://localhost:8787`. Fetch con `credentials: "include"`.

**Shape de listado (compatible spec 04):**

```json
{
  "flags": [
    {
      "flag_key": "checkout-v2",
      "name": "Checkout v2",
      "description": "",
      "critical": false,
      "created_at": "<iso>",
      "updated_at": "<iso>",
      "environments": [
        {
          "environment": "dev",
          "enabled_global": true,
          "rollout_percent": 100,
          "overrides_count": 0
        }
      ]
    }
  ]
}
```

Si el GET detalle usa `overrides[]` y no `overrides_count`, la UI calcula `overrides.length`. Ambos son válidos.

**Arranque:** migrate, seed, `pnpm --filter api dev`, `pnpm --filter web dev`. Usuario demo.

No hay páginas de crear/editar todavía (spec 07); el dashboard puede mostrar keys como texto, no como formulario.

## Alcance

### In scope

- Ruta `/` (dashboard) **protegida**: sin sesión → redirect `/login`.
- Tabla o lista: `flag_key`, `name`, badge `critical`, por cada ambiente `dev`/`staging`/`prod`: on/off (`enabled_global`), `rollout_percent`, conteo de overrides.
- Empty state si `flags` es `[]`.
- Error state si la API falla (401 recae en login).
- Navegación mínima: link a login no hace falta si ya hay sesión; opcional logout llamando `POST /v1/auth/logout`.

### Out of scope

- Formularios crear/editar (07).
- UI de % y overrides editables (08).
- Simulador (10), historial (11).
- Evaluar flags en el cliente (prohibido por PRD).

## Tareas

1. Helper `apiFetch(path)` en web: base URL + credentials.
2. En `/`, `GET /v1/auth/me`; si 401 redirect `/login`.
3. `GET /v1/flags` y renderizar tabla. Orden sugerido: `flag_key` asc.
4. Empty state: texto “No hay flags” (o equivalente).
5. Estilos Tailwind coherentes, layout simple (header con nombre de la app).
6. Logout opcional pero recomendado (limpia cookie y vuelve a login).

## Criterios de aceptación

1. Con seed de spec 03 y sesión demo, `/` muestra al menos `checkout-v2` y `payments-kill`. **RF-03**.
2. Para `checkout-v2` en `prod` se ve % **50** y **2** overrides (acme, globex).
3. Se distingue visualmente `enabled_global` true vs false por ambiente.
4. Flags `critical: true` se marcan (badge o texto).
5. Sin cookie, visitar `/` acaba en `/login` y no se listan flags.
6. Lista vacía (DB sin flags) muestra empty state, no un crash.

## Notas técnicas

- La web **no** replica reglas de precedencia; solo muestra datos de la API.
- No usar server-side fetch a la API sin reenviar cookie (si se usa Server Components, reenviar `Cookie` header). Client fetch con credentials es aceptable.
- No añadir `POST /v1/flags` desde esta pantalla.
