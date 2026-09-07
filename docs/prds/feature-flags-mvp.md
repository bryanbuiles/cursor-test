# PRD: Feature Flags internas (MVP)

**Estado:** decisiones de producto bloqueadas  
**Auth:** usuario demo (sin OAuth, roles ni permisos)  
**Persistencia:** SQLite local  
**Targeting:** ambiente, empresa, rollout porcentual

---

## 1) Contexto y problema

Hoy, activar o desactivar una funcionalidad implica un deploy. Eso retrasa experimentos, rollouts graduales y apagados de emergencia, y mezcla cambios de producto con releases de código.

Se necesita una herramienta **interna** para controlar flags en runtime: por **ambiente**, por **empresa** (tenant) y por **porcentaje de empresas**, sin redeploy.

---

## 2) Objetivo

Permitir a un operador interno (usuario demo) crear y cambiar flags, y a los servicios internos **evaluar** si una feature está on u off para una empresa en un ambiente, con efecto en menos de 1 minuto y persistencia en SQLite local.

**Éxito del MVP:** un flag se crea, se configura targeting y el evaluador responde de forma estable y predecible, verificable con casos de prueba, sin deploy de la app consumidora.

---

## 3) Público objetivo y usuarios

| Actor | Relación con el producto |
| --- | --- |
| Operador interno | Persona autenticada con el **usuario demo**. Crea, edita y activa/desactiva flags (incluido prod). |
| Servicio interno (backend) | Consumidor. Llama al evaluador server-side. |
| Frontend interno | No evalúa reglas. Consulta un endpoint interno que usa el evaluador. |

No hay clientes finales, self-serve ni roles. Cualquiera con el usuario demo tiene las mismas capacidades.

---

## 4) Alcance

### In scope

- Flags booleanos (on/off).
- Targeting por **ambiente** (`dev` / `staging` / `prod`), **override por empresa** y **rollout porcentual** (por ambiente, sobre empresas).
- CRUD de flags y de reglas de targeting.
- Evaluador server-side + endpoint interno de evaluación.
- Persistencia **SQLite local**.
- Login con **un usuario demo** fijo.
- Auditoría básica (quién/qué/cuándo, con el usuario demo como actor).
- Cache corta (30–60s) para aplicar cambios sin deploy.
- Fail-closed por defecto; excepción para flags marcados **críticos**.

### Out of scope

- OAuth, SSO, invitaciones, roles y permisos avanzados.
- Variantes A/B, experimentos, payloads de configuración.
- Kill switches de infraestructura.
- SDKs públicos o UI para clientes.
- Rollback automático.
- Base de datos remota.
- Hash por request o por usuario final (el % es de empresas).

---

## 5) Conceptos de dominio

### Flag

Definición global de una feature. Identificada por `flag_key` estable. Es booleana. Tiene estado **por ambiente** (p. ej. el flag puede estar globalmente off en `prod` y on en `dev`).

Atributos mínimos: `flag_key`, nombre, descripción opcional, indicador **crítico** (sí/no), estado por ambiente, reglas de targeting por ambiente, timestamps.

### Targeting rule

Regla que, en un **ambiente**, decide el valor del flag para una empresa. Tipos, en orden de precedencia:

1. **Ambiente off global:** si el flag está off en ese ambiente, el resultado es **off** para todas las empresas (no se evalúan overrides ni %).
2. **Override por empresa:** `empresa_id` forzada **on** u **off**. Esa empresa **no** entra al rollout porcentual.
3. **Rollout porcentual:** entero 0–100. Aplica solo a empresas **sin** override. El universo es empresas, no requests.

### Evaluador

Componente server-side que, dados `flag_key`, `ambiente` y `empresa_id`, devuelve `{ enabled: boolean }` aplicando la precedencia anterior.

- Rollout estable: `hash(empresa_id + flag_key)` determinista; la misma empresa no parpadea.
- Si no puede evaluar (error de persistencia, flag inexistente, datos corruptos): **fail-closed** (`enabled: false`), salvo flag **crítico**, que usa su último valor conocido en cache o, si no hay, **fail-open** no aplica: ver RF-12.
- Fuente de verdad: el evaluador. El front no replica reglas.

### Empresa y ambiente

- **Empresa:** tenant interno con `empresa_id` estable.
- **Ambientes:** únicamente `dev`, `staging`, `prod`. El % y los overrides son **por ambiente**.

---

## 6) Requerimientos funcionales

**RF-01.** El sistema permite iniciar sesión únicamente con un usuario demo predefinido (credenciales fijas). Un login válido abre sesión; uno inválido se rechaza y no expone datos de flags.

**RF-02.** Un operador autenticado puede crear una flag con `flag_key` único, nombre y opcionalmente descripción e indicador crítico. Si `flag_key` ya existe, la creación falla con error claro y no se duplica.

**RF-03.** Un operador autenticado puede listar flags y ver, por cada una, estado por ambiente y resumen de targeting (overrides y % vigente).

**RF-04.** Un operador autenticado puede editar nombre, descripción e indicador crítico de una flag existente. `flag_key` no es editable tras crearla.

**RF-05.** Un operador autenticado puede poner una flag **on** u **off** a nivel de ambiente (`dev` | `staging` | `prod`) sin redeploy de la aplicación consumidora.

**RF-06.** Un operador autenticado puede, por ambiente, definir un rollout porcentual entero entre 0 y 100 inclusive. Valores fuera de rango se rechazan.

**RF-07.** Un operador autenticado puede, por ambiente, crear, actualizar y eliminar un override on/off para un `empresa_id`. Un `empresa_id` tiene como máximo un override por flag y ambiente.

**RF-08.** El evaluador, con `flag_key` + `ambiente` + `empresa_id`, aplica esta precedencia y nada más: (1) si el ambiente está off global → `enabled: false`; (2) si hay override de esa empresa → ese valor; (3) si no, el rollout porcentual; (4) si % = 0 o no hay empresas en el bucket → `enabled: false`.

**RF-09.** Dada la misma terna (`flag_key`, `ambiente`, `empresa_id`) y las mismas reglas, el evaluador devuelve siempre el mismo `enabled` (hash determinista `empresa_id + flag_key`). No usa request id ni usuario final.

**RF-10.** Empresas con override no participan del cálculo porcentual. Cambiar el % no altera el valor de una empresa con override.

**RF-11.** Existe un endpoint interno de evaluación que el backend (y, si aplica, el front vía backend) usa para obtener `enabled`. El cliente no envía reglas; solo la terna de evaluación.

**RF-12.** Si la evaluación falla (SQLite no disponible, flag inexistente, payload inválido), el evaluador responde fail-closed: `enabled: false`, salvo flags marcadas críticas, que responden con el último valor cacheado si existe; si no hay cache, `enabled: false` y se registra el incidente en auditoría/log.

**RF-13.** Un cambio de estado o targeting persistido en SQLite es visible en evaluaciones nuevas en **menos de 60 segundos** (cache 30–60s o invalidación).

**RF-14.** Cada creación, edición, cambio de estado, override o % queda registrado en auditoría básica: timestamp, actor (usuario demo), `flag_key`, ambiente si aplica, y resumen del cambio. La auditoría es consultable en el MVP (lista o equivalente).

**RF-15.** Todas las entidades anteriores se persisten en SQLite local. Reiniciar el proceso no pierde flags, reglas ni auditoría.

---

## 7) Requerimientos no funcionales

- **Latencia de evaluación:** p95 < 50 ms en local con cache caliente (orientativo de MVP; medible con un test o script).
- **Consistencia:** cambios aplicados en < 1 minuto.
- **Disponibilidad del evaluador:** si SQLite o el proceso fallan, fail-closed (RF-12); no degradar a “on” silencioso.
- **Persistencia:** un único archivo SQLite local; sin réplica distante en v1.
- **Seguridad:** sin OAuth ni ACL. La sesión demo es suficiente para el MVP; no se expone el endpoint de evaluación a internet pública como requisito de este PRD (uso interno).
- **Observabilidad mínima:** log de errores de evaluación y de escrituras de auditoría.
- **Estabilidad de rollout:** el hash no cambia de algoritmo sin migración versionada (fuera de MVP cambiar el hash).

---

## 8) Criterios de aceptación del MVP

1. Login demo funciona; credenciales incorrectas no permiten operar flags.
2. Puedo crear una flag, ponerla off en `prod` y on en `dev`; el evaluador respeta el ambiente.
3. Override empresa A = on y empresa B = off en `prod` produce esos valores aunque el % sea 0 o 100.
4. Sin overrides, % = 0 → todas off; % = 100 → todas on; % = 50 → el mismo conjunto de empresas queda on de forma estable al reevaluar.
5. Empresa con override no cambia al mover el % de 10 a 90.
6. Tras editar un % o un override, una evaluación 60s después refleja el cambio (sin deploy).
7. Flag inexistente o SQLite caído → `enabled: false` (fail-closed).
8. Reiniciar la app: flags y reglas siguen en SQLite.
9. Auditoría muestra al menos el último cambio de estado y de targeting.
10. No existen pantallas ni APIs de OAuth, roles o permisos.

---

## 9) Riesgos y supuestos

**Supuestos**

- `empresa_id` lo provee el sistema consumidor; este MVP no administra un catálogo rico de empresas (puede persistir solo ids vistos en overrides).
- Ambientes son exactamente `dev`, `staging`, `prod`.
- Un solo proceso / una sola instancia es aceptable con SQLite (sin escritura concurrente multi-nodo).
- El usuario demo es aceptable porque es herramienta interna de curso/prototipo, no producción multi-equipo.
- “Crítico” no implica fail-open ciego: sin cache, también fail-closed (RF-12).

**Riesgos**

- SQLite + varios procesos writers → bloqueos o datos stale; el MVP asume un writer.
- Cache de 30–60s puede dejar una empresa en el valor viejo durante el SLA; es aceptado.
- Hash no criptográfico mal elegido puede sesgar el %; se debe validar con un test de distribución aproximada.
- Usuario demo único: cualquier persona con la credencial puede cambiar prod; aceptado por alcance.
- Fail-closed puede apagar features críticas si SQLite falla; mitigación: flag crítica + cache, no fail-open sin dato.

---

*Fin del PRD MVP. Cambios a este documento son deltas explícitos sobre las decisiones bloqueadas.*
