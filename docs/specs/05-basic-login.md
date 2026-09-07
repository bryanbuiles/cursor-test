# Spec 05 — Basic login

## Objetivo

Añadir login de **un usuario demo** (cookie httpOnly) y proteger las rutas de operador `/v1/flags*`. Cubre **RF-01**, CA-1 y CA-10 (sin OAuth ni roles).

## Contexto y dependencias

Un agente debe poder implementar esta spec **leyendo solo este archivo**. Se asume spec **01–04**: API CRUD de flags en `/v1/flags`, SQLite, Vitest, web Next.js en `:3000`, API `:8787`.

**Contrato de flags:** el de spec 04 (POST/GET/PATCH flags, PUT environments, PUT/DELETE overrides). No se cambia el JSON.

**Credenciales demo (fijas):**

- Usuario: env `DEMO_USER` default `"demo"`.
- Password: env `DEMO_PASSWORD` default `"demo"`.

**Cookie:** nombre `ff_session`, `HttpOnly`, `Path=/`, `SameSite=Lax`. En local `Secure` no es obligatorio (http). Valor: token opaco firmado o random persistido en memoria (Map). TTL sugerido 8h.

Tras login, `audit_events.actor` debe ser el username demo (sigue siendo `"demo"`).

**CORS:** permitir credentials (`Access-Control-Allow-Credentials: true`) y origen explícito `http://localhost:3000` (no `*`).

**Web:** `fetch` a la API con `credentials: "include"`.

## Alcance

### In scope

- `POST /v1/auth/login` `{ "username", "password" }` → Set-Cookie; 401 si no coincide.
- `POST /v1/auth/logout` → borra cookie.
- `GET /v1/auth/me` → `{ "username": "demo" }` si hay sesión; **401** si no.
- Middleware: todas las rutas que empiezan por `/v1/flags` requieren sesión válida → **401** `{ "error": "unauthorized" }`.
- Página Next.js `/login` (formulario usuario/password). Login OK redirige a `/`.
- Sin sesión, la web no debe mostrar datos de flags (redirigir a `/login` si se visita `/` — middleware Next o check client).
- Tests API: sin cookie `GET /v1/flags` es 401; con login, 200. Login inválido 401.

### Out of scope

- OAuth, SSO, invitaciones, roles, permisos, refresh JWT complejo, 2FA.
- Proteger `POST /v1/evaluate` (aún no existe; spec 09 lo deja **sin** cookie a propósito).
- `GET /v1/audit` (spec 11): cuando exista, también será de operador (sesión); no implementar el GET aquí.
- Evaluador, simulador, historial UI.

## Tareas

1. Variables `DEMO_USER` / `DEMO_PASSWORD` leídas al arrancar.
2. Implementar login/logout/me y store de sesiones (memoria está bien para MVP).
3. Aplicar middleware **solo** a `/v1/flags` y subrutas. `/health` y `/v1/auth/*` (login) públicos. Logout/me: me requiere sesión; logout puede ser 204 aun sin sesión.
4. Actualizar tests de spec 04: o bien hacen login en `beforeEach`, o montan app de test con bypass **prohibido en prod**. Preferir login real en tests de integración.
5. Página `/login` en `apps/web`. `NEXT_PUBLIC_API_URL=http://localhost:8787`.
6. CORS credentials.
7. No crear tablas de users.

## Criterios de aceptación

1. `POST /v1/auth/login` con `demo`/`demo` → 200 y cookie `ff_session`. **RF-01**.
2. Login con password incorrecto → 401, sin cookie de sesión usable, `GET /v1/flags` sigue 401.
3. Sin cookie, `GET /v1/flags` → **401**. Con cookie válida → **200**. CA-1.
4. No existen pantallas ni endpoints de OAuth, roles o permisos. CA-10.
5. En el browser: ir a `http://localhost:3000/login`, autenticarse, y una petición a flags incluye la cookie (Network).
6. `pnpm --filter api test` pasa (tests actualizados con sesión).

## Notas técnicas

**POST `/v1/auth/login`**

Request: `{ "username": "demo", "password": "demo" }`  
Response 200: `{ "username": "demo" }` + `Set-Cookie: ff_session=...`.

**GET `/v1/auth/me`** 200: `{ "username": "demo" }`.

**POST `/v1/auth/logout`** 204.

Cookie no es JWT de un IdP. Comparación de password: igualdad directa con env (MVP interno).

Web no evalúa flags; solo llama a la API (PRD).
