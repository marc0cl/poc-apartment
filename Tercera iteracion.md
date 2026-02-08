## 1) Supuestos (si faltan datos)

* S1: El repo **aún no está “armado”** o está en etapa muy inicial (no vi `package.json`, `go.mod`, estructura de carpetas, etc.). Por eso el plan propone PRs que **primero fijan DX/CI/contratos**, y recién después feature work.
* S2: “Todo local” = `web + api + worker + db` corriendo en tu máquina, **sin exposición pública** (por ahora).
* S3: En esta fase local se permite “personal / non-commercial” para probar (MapTiler free, etc.). El cambio a “premium/comercial” se hace **antes de publicar a público real**.
* S4: Respetamos las decisiones cerradas v1 (multi-tenant, UF+CLP siempre, leads, ubicación exacta, etc.) como ley del proyecto. 
* S5: Vamos a minimizar decisiones “implícitas” usando **contract-first (OpenAPI)** y migraciones SQL forward-only para que Codex/Claude-Code implementen sin inventar.  

---

## 2) Objetivo y alcance

* **Objetivo:** Definir la “Tercera Iteración” como un **plan local-first** con PRs ultra explícitos, que produzca un producto viable versionado en GitHub, y que deje **puntos de extensión** claros para una migración “premium” (GCP, suscripciones, CDN, etc.).
* **Fuera de alcance (por ahora):**

  * Deploy público/infra final (Cloud Run/Cloud SQL/CDN/etc.). Solo dejamos “hooks” y perfiles de config.
  * Optimización extrema (CDN, invalidaciones sofisticadas).
* **Criterios de éxito:**

  * Cada PR es **pequeño, verificable y reversible**.
  * El repositorio tiene **gates** (CI + templates + contratos) que reducen al mínimo las alucinaciones de IA. 
  * La migración a “premium” es una secuencia de PRs **acotados**, no un rewrite.

---

## 3) Contexto técnico observado (si aplica)

* Ya existe documentación que fija: decisiones cerradas v1, arquitectura, API v1, y modelo de datos/migraciones. 
* Hay un protocolo explícito anti-alucinación y un formato obligatorio para respuestas/diseño (lo usamos como “norma”).  
* Hay plantilla ADR para decisiones importantes (la usaremos para formalizar el “local-first premium-ready”). 

---

## 4) Diseño propuesto

### 4.1 Arquitectura

**Idea central:** “Local-first” no significa “hackear todo”; significa tener **perfiles de runtime** y **ports/adapters** desde el día 1.

**Componentes lógicos (monorepo recomendado por la propia arquitectura):**

* `web` (Next/React) público + admin bajo `/admin`
* `api` (Go) REST `/v1` contract-first
* `worker` (Go) jobs: UF, media, notificaciones
* `db` (Postgres + PostGIS) via docker local
* (opcional) `redis` local para rate-limit/colas simples

Esto calza con la arquitectura definida. 

**Perfiles de ejecución (config, no forks de código):**

* `PROFILE=local`

  * Storage: `localfs` (carpetas) o “signed upload local” (URL a un endpoint local)
  * Mail: `dev` (log/archivo), sin enviar emails reales
  * Maps: `none` o “placeholder” (link a maps)
  * FX/UF: `stub` (valor en DB/seed) + job manual/cron local
* `PROFILE=premium`

  * Storage: `gcs` (signed URLs reales + buckets)
  * Mail: `resend`
  * Maps: `maptiler` (plan comercial) / `google` (si se decide)
  * FX/UF: `cmf` (con API key) + fallback

**Regla para evitar deuda:** *el código de negocio nunca “pregunta por el env”*; el env solo selecciona implementaciones vía interfaces.

---

### 4.2 Modelo de datos

En local usamos el **mismo esquema** que en premium (solo cambia infraestructura). Eso evita migraciones dolorosas.

Base mínima (alineada a lo ya definido):

* Multi-tenant: `organizations`, `users` con RBAC por org. 
* Propiedades/publicación: `properties`, `listings` (draft/published/archived), slugs. 
* Dual price: `fx_rates`, `system_settings.uf_version`, y derivados UF/CLP en `listings`. 
* Leads: `leads` + pipeline. 
* Media: `media_assets`, `listing_media` (orden + cover). 

**Premium hooks en DB (aunque no los uses aún):**

* `media_assets.storage_provider` (`localfs|gcs|s3`) para switch limpio.
* `organization_domains` (white-label) como tabla “dormida” hasta premium. 

---

### 4.3 API / Contratos

**Regla local-first para no alucinar:**

1. **OpenAPI v1 es la fuente de verdad**,
2. el código se genera o se valida contra el contrato,
3. la UI usa cliente tipado generado.  

**MVP local (subset) recomendado para partir:**

* Público:

  * `GET /v1/listings`
  * `GET /v1/listings/{id_or_slug}`
  * `POST /v1/leads`
  * `GET /v1/indicators/uf` (aunque sea stub al inicio)
* Admin:

  * `POST /v1/auth/login`, `POST /v1/auth/refresh`, `POST /v1/auth/logout`, `GET /v1/me`
  * CRUD listings (`/v1/admin/listings...`) + publish/unpublish/archive
  * `GET /v1/admin/leads`, `PATCH /v1/admin/leads/{id}`

Esto está alineado con la especificación ya escrita. 

---

### 4.4 Concurrencia / Paralelismo / Jobs

Local-first:

* `worker` puede correr como proceso separado, pero los jobs pueden iniciar **manual** (comando) hasta estabilizar.
* Media: en local puede ser “noop” o generar 1–2 variantes básicas; el diseño deja el hook para derivados completos premium. 
* UF: al inicio `fx_rates` se seed-ea; luego se agrega job y providers (stub → cmf). 

---

### 4.5 Caché

En local: lo mínimo (para no complicar).

* Definir headers cacheables en GET públicos (aunque no haya CDN aún), para que premium sea “activar, no reescribir”. 
* ETag/`uf_version` (incluso si al inicio uf_version es fijo) para compatibilidad futura. 

---

### 4.6 Observabilidad

Local:

* request-id + logs estructurados desde día 1, sin vendor lock-in. 
  Premium:
* OpenTelemetry para traces/metrics cuando toque (hook, no obligación en MVP local).

---

## 5) Plan incremental (PRs)

### Convención de PRs para IA (clave para “no alucinar”)

Cada PR debe incluir en su descripción (o en `docs/prs/NNN-*.md`) esta estructura:

```md
# Objetivo
# Alcance (IN)
# No-alcance (OUT)
# Entregables (archivos/rutas)
# Criterios de aceptación (checklist verificable)
# Plan de tests (comandos + casos)
# Riesgos / Rollback
# Notas para premium (hooks que se dejan)
```

Esto fuerza a Codex/Claude-Code a ejecutar “modo implementación” sin inventar. Se apoya en el Protocolo IA. 

---

### PR1 — “Iteración 3: Local-first premium-ready” (documentación + reglas)

**IN**

* Crear doc: `docs/real-estate/08-local-first-y-migracion-premium.md`:

  * perfiles `local` vs `premium`
  * tabla de providers (Storage/Mail/Maps/FX)
  * estrategia de migración (PRs premium)
* Crear `docs/ai/PR_SPEC_TEMPLATE.md` (plantilla anterior)
* Crear `docs/ai/AI_QUALITY_GATE.md` (checklist “Done”) alineado al quality gate del repo (puede basarse en el bloque que ya está en Response Template). 

**OUT**

* Cualquier feature del producto.

**Aceptación**

* Docs claros: cualquier humano puede tomar un PR y saber qué hacer sin decidir cosas nuevas.

**Premium hooks**

* Dejar explícito el set de interfaces y env vars (ver PR2).

---

### PR2 — Bootstrapping local (DX) + estructura monorepo mínima

**IN**

* Estructura de carpetas (si el repo está vacío): `apps/web`, `apps/api`, `apps/worker`, `db/migrations`, `docs/...`
* `docker-compose.yml` (mínimo: Postgres + PostGIS)
* `.env.example`
* `README.md` con “cómo correr local”
* Comandos estables (recomendado): `Makefile` o `Taskfile` con targets:

  * `dev`, `test`, `lint`, `fmt`, `db:up`, `db:migrate`, `db:reset`

**OUT**

* Implementación real de endpoints.

**Aceptación**

* Cualquier dev puede levantar DB local y correr skeleton de apps sin tocar configuración manual.

**Premium hooks**

* Targets de build/test serán usados por GitHub Actions (CI no invoca comandos ad-hoc).

---

### PR3 — GitHub Actions (CI base) + templates de PR

**IN**

* `.github/workflows/ci.yml`:

  * correr `lint` y `test` para Go + Web usando los comandos del PR2
  * cachear deps (Go y Node) si aplica
* `.github/pull_request_template.md` usando el PR spec
* (Opcional) `.github/ISSUE_TEMPLATE/*` para tareas

**OUT**

* Deploy.

**Aceptación**

* Un PR sin tests/lint pasa local pero **no** entra a main si CI falla.
* Branch protection (config manual en GitHub): require CI + 1 review.

**Premium hooks**

* CI será extendido para “generate checks” (OpenAPI/sqlc) en PR4/PR5.

---

### PR4 — OpenAPI v1 “mínimo viable” + generación tipada

**IN**

* `openapi/v1.yaml` (subset MVP local: listings, listing detail, leads, auth admin, admin listings, admin leads, uf indicator)
* Script `gen:openapi` (o target en Makefile) para:

  * generar tipos servidor Go (o stubs)
  * generar cliente TS para `web`
* CI: step que falla si el código generado no está en sync.

**OUT**

* Implementar toda la lógica; aquí solo contrato + plumbing.

**Aceptación**

* Front y back compilan usando tipos generados (aunque handlers devuelvan mock/501).

**Premium hooks**

* Cada nuevo endpoint se hace en OpenAPI primero (no se permiten endpoints “inventados”). 

---

### PR5 — Migraciones DB v1 + seed mínimo

**IN**

* `db/migrations/0001_init.sql` (base multi-tenant + listings + leads + media + enums)
* `db/migrations/0002_triggers.sql` (updated_at, search tsv, etc. si aplica)
* `db/migrations/0003_fx_rates_and_dual_prices.sql` (UF/CLP) 
* Seed script (o migración seed) que crea:

  * 1 organization
  * 1 admin
  * 1–3 listings publicados con precio UF base y derivados
  * 1 fx_rate UF_CLP “actual”
  * `uf_version=0`

**OUT**

* Optimizaciones complejas.

**Aceptación**

* `db:reset` deja data navegable para UI.

**Premium hooks**

* Esquema ya soporta `storage_provider` y `organization_domains` aunque no se use.

---

### PR6 — API skeleton: middleware + Problem Details + DB plumbing

**IN**

* Servidor Go con:

  * request-id
  * manejo consistente de errores `application/problem+json`
  * conexión DB + pool
* Implementar `GET /v1/health` y “stubs controlados” para el resto (501 con Problem Details y `type` claro)

**OUT**

* Auth y reglas de negocio.

**Aceptación**

* Respuestas de error consistentes y no “panic”.

---

### PR7 — Auth admin + RBAC por organización

**IN**

* Implementar `/v1/auth/login`, `/v1/auth/refresh`, `/v1/auth/logout`, `/v1/me`
* RBAC middleware (ADMIN/AGENT/ANALYST) por org. 
* Política de tokens (según doc): access token corto + refresh token (cookie HttpOnly o en body, pero definido y documentado). 

**OUT**

* Auth público visitante.

**Aceptación**

* Un usuario de otra org no puede acceder a recursos ajenos (tests de autorización mínimos).

**Premium hooks**

* Dejar interface para proveedor de identidad si en premium quieren OIDC.

---

### PR8 — Admin listings CRUD (draft/publish) + validaciones duras

**IN**

* Endpoints admin para crear/editar/listar, publish/unpublish/archive
* Validaciones de publicación (checklist) alineadas a la spec. 
* Generación de slug estable

**OUT**

* Media upload.

**Aceptación**

* Se puede publicar un listing seed + crear uno nuevo desde admin vía API.

**Premium hooks**

* ETag/If-Match para updates (concurrencia) si ya está en spec. 

---

### PR9 — Público listings list/detail + paginación por cursor

**IN**

* `GET /v1/listings` con filtros mínimos (operation, type, price_uf range, bedrooms/bathrooms, sort) según spec. 
* Cursor pagination opaca
* `GET /v1/listings/{id_or_slug}`

**OUT**

* Search avanzada (full-text/geo).

**Aceptación**

* Listado funciona estable con datos seed y no hace N+1 obvio.

**Premium hooks**

* Cache headers + ETag listos para CDN posterior. 

---

### PR10 — Leads (público) + panel admin de leads

**IN**

* `POST /v1/leads`:

  * validación listing publicado
  * exigir email o phone
  * anti-spam mínimo: honeypot + rate limit (local: memoria; premium: Redis) 
* Admin:

  * `GET /v1/admin/leads`
  * `PATCH /v1/admin/leads/{id}` (status/assign/note)

**OUT**

* Envío de email real (se deja para premium PR).

**Aceptación**

* Flujo end-to-end: usuario crea lead → aparece en admin.

**Premium hooks**

* Interface `Mailer` ya definida (aunque sea dev/noop en local).

---

### PR11 — Web público (mínimo) + Admin UI mínimo

**IN**

* Público:

  * `/arriendo` (listado)
  * `/arriendo/[slug]` (detalle)
  * form de lead
* Admin:

  * `/admin/login`
  * `/admin/listings`
  * `/admin/listings/[id]/edit`
  * `/admin/leads`

**OUT**

* UI “bonita”; priorizar funcional + tipado.

**Aceptación**

* Navegación completa usando el API real, con cliente tipado generado desde OpenAPI.

---

### PR12 — Media local (upload) “premium-compatible”

**IN**

* Implementar endpoints de media según spec (upload-init, upload-complete, reorder, delete). 
* `StorageProvider` con implementación `localfs`:

  * guarda en `.data/storage/...`
  * genera `upload.url` que apunta a un endpoint local (simula signed URL)
* UI admin: subir fotos y asignar cover

**OUT**

* Derivados completos (webp 400/800/1200/1600) si complica; puede quedar como noop + TODO premium.

**Aceptación**

* Un listing publicado muestra imágenes (aunque sean originales).

**Premium hooks**

* `StorageProvider` soporta `gcs` sin cambiar UI/contrato.

---

### PR13 — UF job + recálculo de precios (local stub → premium provider)

**IN**

* `GET /v1/indicators/uf`
* `fx_rates` + `uf_version` actualizables
* Job `fx_update_uf_daily` que:

  * en `local`: toma UF desde seed/config (stub)
  * incrementa `uf_version`
  * recalcula `listings.price_uf_amount` / `price_clp_amount` 

**OUT**

* Integración CMF real (premium).

**Aceptación**

* Cambiar UF (stub) cambia CLP en listados/detalle y cambia `fx.version`.

**Premium hooks**

* Interface `FxProvider` con implementaciones `stub|cmf|fallback`.

---

## PRs de migración “premium” (cuando decidas publicar)

Estos PRs se diseñan para ser **switch de providers**, no reescrituras:

### P1 — Storage GCS (signed URLs reales) + buckets draft/public

* Implementación `StorageProvider=gcs`
* Separación draft/public como indica la arquitectura 
* Mantener localfs para dev

### P2 — Mailer Resend (real) + plantillas

* Implementación `Mailer=resend`
* Mantener `Mailer=dev` local

### P3 — Maps provider comercial (MapTiler paid o alternativa)

* `MAP_PROVIDER=maptiler` + key
* Mantener `MAP_PROVIDER=none` local

### P4 — Caching/CDN + invalidaciones (webhook interno)

* Activar cache headers/ETag y hooks de purge (ya preparados) 

### P5 — Deploy en GCP (si lo eligen) + observabilidad

* Contenerización + ejecución en Cloud Run/VM/lo que definan
* OTel/metrics/traces si corresponde

---

## 6) Tests

Mínimo por PR (para que CI sea un “freno anti-alucinación”):

* **Unit (Go):** validaciones, RBAC, precio UF/CLP, cursor encoding/decoding.
* **Integration (Go + Postgres):** queries de listado con filtros + paginación.
* **E2E smoke (Web):** listar → abrir detalle → enviar lead → aparece en admin.

Esto es coherente con el “quality gate” del Protocolo IA. 

---

## 7) Riesgos y mitigaciones

* **Riesgo:** que “local-first” derive en código especial-casing y deuda.

  * **Mitigación:** perfiles + interfaces (ports/adapters) y contrato OpenAPI como verdad. 
* **Riesgo:** que IA implemente endpoints/DB “de memoria”.

  * **Mitigación:** PR4 (OpenAPI) + PR5 (migraciones) + CI que valida generación/sync. 
* **Riesgo:** que se postergue demasiado “premium” y luego la migración sea grande.

  * **Mitigación:** P1–P5 están diseñados como swaps de providers, no “refactor masivo”.

---

## 8) Implementación (si se solicitó código)

No pediste código todavía. Si quieres, el siguiente paso es que trabajemos **PR1** (doc + templates + quality gate) con contenido exacto de archivos y rutas.  
