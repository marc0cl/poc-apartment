## 1) Supuestos (si faltan datos)

* S1: No tengo visibilidad del **estado actual del código** (solo documentación). Estimo como si hubiese que construir el MVP “desde cero” o desde un esqueleto mínimo.
* S2: “MVP público” = **catálogo público + detalle + formulario de contacto (leads) + backoffice mínimo** para crear/publicar avisos (sin CRM/WhatsApp aún). Esto calza con las decisiones ya cerradas del proyecto (público sin login; favoritos/búsquedas guardadas quedan para después). 
* S3: Tráfico inicial bajo/moderado (primeros 1–3 meses): 5k–50k pageviews/mes, <2k publicaciones. (Si es más, suben costos de mapas/CDN/DB y hay que ajustar índices/caché).
* S4: Austeridad aceptada: **1 entorno “prod” + staging liviano**, observabilidad mínima, y hardening gradual.
* S5: Costos expresados en **USD** (por estabilidad); conversión a CLP depende del tipo de cambio.

---

## 2) Objetivo y alcance

* **Objetivo:** publicar un MVP usable por el público para buscar/ver propiedades y contactar; y que la corredora pueda publicar avisos sin intervención técnica.
* **Fuera de alcance (por ahora):**

  * Login de visitantes + favoritos + búsquedas guardadas (decisión del proyecto: requieren login, pero no son requisito para navegación pública). 
  * WhatsApp/CRM, automatizaciones de marketing.
  * Multi-región, HA, WAF avanzado, analítica avanzada.
* **Criterios de éxito:**

  * Publicar un aviso con fotos en <10 min desde backoffice.
  * Listado + detalle cargan “rápido” en móvil (SEO/SSR/ISR según arquitectura propuesta). 
  * Precio siempre visible en **UF arriba + CLP abajo**, con UF actualizada automáticamente. 
  * Endpoint público de leads protegido (rate-limit + anti-bot).

---

## 3) Contexto técnico observado (si aplica)

* Decisiones del MVP: multi-tenant (org/corredora), operaciones arriendo/venta, UF+CLP, ubicación exacta, media en buckets + CDN/derivados, leads, público sin login. 
* Arquitectura objetivo documentada: Next.js (SSR/ISR) + API Go + Postgres/PostGIS + worker para jobs (UF/imagenes) + object storage + CDN. 
* Normas del repo: anti-alucinación, contract-first, paginación por cursor, errores tipo Problem Details, migraciones forward-only.  

---

## 4) Diseño propuesto

### 4.1 Arquitectura

**Base (alineada con el repo):**

* **Frontend** (público + backoffice): Next.js (SSR/ISR para SEO y performance). 
* **API**: Go (REST /v1, Problem Details, RBAC por tenant). 
* **DB**: Postgres + PostGIS (búsqueda por zona/geo si se necesita). 
* **Worker/Jobs**: UF diaria + derivados de imágenes. 
* **Object storage + CDN** para fotos/planos.

**Dónde aplicar austeridad (sin romper el diseño):**

* Mantener “multi-tenant” en esquema, pero arrancar con **1 corredora** (seed) y RBAC básico.
* Backoffice mínimo: CRUD de avisos + carga de fotos + ver leads.
* Mapas: arrancar con **link “ver en Google Maps”** y/o mapa estático; dejar mapa interactivo como “upgrade” si cuesta. (Abajo detallo costos).

---

### 4.2 Modelo de datos (mínimo MVP)

Entidades mínimas (ya alineadas con lo documentado):

* `organizations`, `users`
* `listings` (operación, tipo, dirección + lat/lng, precio_uf, precio_clp_cacheado, estado publish/draft)
* `listing_media` (urls/keys, orden, tipo foto/plano)
* `leads` (listing_id, org_id, contacto, mensaje, metadata anti-spam)
* `uf_rates` (fecha, valor_clp) 

Índices clave:

* `listings(org_id, status, published_at desc)`
* filtros frecuentes: `operation`, `property_type`, `bedrooms`, `bathrooms`, `price_uf`, `geo` (si se filtra por mapa).
* `leads(org_id, created_at desc)`

---

### 4.3 API / Contratos (mínimo MVP)

**Público**

* `GET /v1/listings?cursor=&limit=&operation=&type=&bedrooms=&price_uf_min=&price_uf_max=...`
* `GET /v1/listings/{id}`
* `POST /v1/leads` (rate-limit + anti-bot)

**Backoffice (autenticado)**

* `POST /v1/auth/login`
* `GET /v1/orgs/{orgId}/listings`
* `POST /v1/orgs/{orgId}/listings`
* `PATCH /v1/orgs/{orgId}/listings/{id}` (incluye publish/unpublish)
* `POST /v1/orgs/{orgId}/listings/{id}/media:sign-upload` (signed URL)
* `GET /v1/orgs/{orgId}/leads`

Esto es consistente con la guía contract-first y con los endpoints sugeridos en la documentación. 

---

### 4.4 Concurrencia / Paralelismo / Jobs

* **UF**: 1 job diario (I/O bound). Fuente recomendada:

  * **CMF**: requiere `apikey` en cada request; es una fuente institucional para UF (API CMF Bancos v3). ([api.cmfchile.cl][1])
  * Fallback (si falla CMF): `mindicador.cl` (open source). ([mindicador.cl][2])
* **Imágenes**: job async (CPU+I/O): generar thumbnails/webp y guardar en bucket/CDN. Limitar concurrencia (p.ej. 2–4 workers) para no saturar CPU.

---

### 4.5 Caché

* Público read-heavy ⇒ cachear:

  * CDN para imágenes.
  * `GET /listings` y `GET /listings/{id}` con `ETag` + TTL corto (ej. 30–120s) + invalidación al publicar/editar. (Esto está en línea con el diseño del repo).

---

### 4.6 Observabilidad

Mínim-to-end.

* logs estructurados (sin secretos).
* métricas: latencia p95, rate de errores 4xx/5xx, tasa de creación de leads, cola de jobs.

---

### 4.7 Estimaciones de costos (infra + terceros)

#### Costos “sí o sí” (casi siempre)

* **Email** (notificaciones de leads / OTP futuro):

  * Resend: plan Free $0/mes con 3.000 emails/mes; Pro $20/mes con 50.000 emails/mes. ([Resend][3])
* **Anti-bot** para forms:

  * Cloudflare Turnstile: se posiciona como CAPTCHA “free”; en docs mencionan límites de widgets para free (20 widgets por cuenta). Suficiente para MVP. ([cloudflare.com][4])
* **Object storage** (fotos/planos):

  * Cloudflare R2: $0.015/GB-mes, egress gratis; free tier 10GB-mes + 1M ops clase A + 10M ops clase B. ([Cloudflare Docs][5])

#### Mapas (donde más conviene ser austeros)

* No es buena idea depender de los tiles públicos de OpenStreetMap en producción si hay riesgo de uso intensivo; la política indica que “uso intensivo… es prohibido sin permiso”, y recomiendan alternativas. ([wiki.openstreetmap.org][6])
* Alternativa barata “pro”:

  * MapTiler **Flex**: $25/mes (uso comercial) + cuota incluida (p.ej. 25k sesiones/mes y 500k requests/mes). ([maptiler.com][7])
* Alternativa cara:

  * Google Maps Platform muestra un plan “Starter” $100/mes (según su página de pricing) + trial credit $300 para nuevos clientes. ([Google Maps Platform][8])

---

## 4.8 3 escenarios de hosting (mensual) para un MVP austero

> Nota: estos son **rangos realistas** para lanzar; el costo real depende del tráfico y del peso de imágenes.

### Escenario A — “Ultra austero / 1 VPS”

**Ideal si el equipo puede operar un poco (backups, updates, monitoreo).**

* Compute: 1 droplet 4GB/2vCPU en DigitalOcean: **$24/mes**. ([DigitalOcean][9])
* Storage imágenes: R2 (ej. 50GB): aprox **$0.75/mes** (menos free tier). ([Cloudflare Docs][5])
* Email: Resend Free: **$0/mes**. ([Resend][3])
* Mapas: MapTiler Flex: **$25/mes**. ([maptiler.com][7])
  **Total típico:** ~**$50–$70/mes** (según backups/monitoring).
  Austeridad extra: si al inicio usas solo “link a Google Maps” y no mapa embebido, puedes postergar MapTiler y bajar ~ $25/mes.

### Escenario B — “Low-ops PaaS (Vercel + Render)”

**Ideal si quieren reducir operación manual.**

* Frontend en Vercel Pro: **$20/mes** (Hobby es “personal, non-commercial”; para negocio normalmente Pro). ([Vercel][10])
* API/Worker en Render:

  * instancias desde **$7/mes** (Starter), **$25/mes** (Standard 2GB/1CPU). ([Render][11])
* Postgres Render: desde **$7/mes** (Starter). ([Render][11])
* R2 + Resend + MapTiler: similar a escenario A. ([Cloudflare Docs][5])
  **Total típico:** ~**$85–$150/mes**.

### Escenario C — “Cloudflare-first (Workers/Pages)”

Solo si deciden mover partes del stack a Workers (no siempre encaja con Go).

* Workers Paid plan: mínimo **$5/mes** por cuenta (según docs). ([Cloudflare Docs][12])
  **Recomendación:** dejarlo como opción futura; el MVP más simple es A o B.

---

## 5) Plan incremental (PRs) — hitos pequeños “AI-ready”

Cada hito pensado para: **1 PR chico**, testeable, y posible de ejecutar con Codex/IA + revisión humana.

> Estimación en **días-persona** (8h) con apoyo de IA. Sin IA, multiplicar ~1.3–1.8x.

### PR1 — Esqueleto + contratos + entorno local (1–2 días)

* Docker compose (app + db), env vars, CI mínimo.
* OpenAPI /v1 “vacío” + Problem Details base.
* Healthcheck.
* **Entrega:** levantar stack local en 1 comando.

### PR2 — Migraciones Postgres (org/user/listing/uf/leads) + seed (2–3 días)

* Tablas + índices mínimos.
* Seed de 1 org + 1 admin.
* **Entrega:** DB lista para CRUD.

### PR3 — Auth backoffice + RBAC por org (2–3 días)

* Login básico (email+password) y middleware RBAC.
* **Entrega:** endpoints privados protegidos.

### PR4 — CRUD de listings (draft/publish) (3–4 días)

* Validaciones de negocio (operación, tipo, precio UF, dirección, lat/lng).
* **Entrega:** publicar/despublicar.

### PR5 — API pública de listados + paginación por cursor + filtros (3–4 días)

* Query eficiente + índices.
* **Entrega:** el público puede listar/buscar.

### PR6 — Frontend público (listado + detalle) SSR/ISR + SEO base (4–6 días)

* Páginas: home, listado, detalle.
* Estados loading/error, meta tags, sitemap básico.
* **Entrega:** MVP navegable.

### PR7 — Leads (POST) + anti-bot + rate-limit + panel backoffice (2–3 días)

* Turnstile + rate limit.
* Backoffice: ver leads por listing.
* **Entrega:** flujo comercial completo.

### PR8 — Media upload (signed URLs) + galería + “derivados mínimos” (4–6 días)

* Subida directa a bucket.
* Worker para thumbnail/webp (si es mucho, partir sin derivados y agregarlos en PR9).
* **Entrega:** fotos funcionando sin reventar performance.

### PR9 — UF job + conversión UF→CLP cacheada (2–3 días)

* Integración CMF (apikey) + fallback mindicador.
* Recalcular `price_clp` en cambios UF o en lectura.
* **Entrega:** UF/CLP siempre consistente. ([api.cmfchile.cl][1])

### PR10 — Hardening + Observabilidad + runbook deploy (2–3 días)

* Logs estructurados + request-id + métricas mínimas.
* Backups, rotación de secrets, checklist de deploy.
* **Entrega:** “listo para público” con deuda controlada.

**Total estimado MVP público:** **25–37 días-persona** (200–296 horas) con apoyo de IA.
Con una sola persona dedicando ~20h/semana: **10–15 semanas**. Con dedicación full-time: **5–8 semanas**.

**Cómo convertir a $ (sin inventar sueldos):**

* `Costo dev = horas_totales * tarifa_hora`
* Ejemplo: 240h * (tu tarifa) = costo construcción.

---

## 6) Tests

Mínimo por hito:

* **Unit**: validaciones, conversión UF/CLP, parseo filtros.
* **Integration**: queries de listados con paginación cursor + índices.
* **E2E smoke** (Playwright/Cypress): listar → abrir detalle → enviar lead → verlo en backoffice.
* **Casos borde**:

  * UF faltante para el día (usar último valor válido).
  * Listing sin media.
  * Rate-limit/Turnstile fallando (degradar con error claro).

---

## 7) Riesgos y mitigaciones

* **R1: Subestimar el “backoffice”** (suele comer más tiempo que el público).

  * Mitigación: backoffice minimalista (sin roles complejos; 1 admin por org al inicio).
* **R2: Costos inesperados por mapas**.

  * Mitigación: empezar con link a Google Maps + lat/lng visibles; activar MapTiler cuando haya tracción. (Evitar tiles públicos OSM en producción). ([wiki.openstreetmap.org][6])
* **R3: UF fuente externa falla**.

  * Mitigación: CMF como primario (apikey) + fallback mindicador + “last known good” en DB. ([api.cmfchile.cl][1])
* **R4: Infra “1 VPS” se cae / operación pesa**.

  * Mitigación: backups automáticos + monitoreo básico; o migrar a PaaS (Escenario B) cuando haya señales de tracción.

---

## 8) Implementación (si se solicitó código)

No solicitado en este mensaje.

---

### Referencias internas usadas (normativa del proyecto)



[1]: https://api.cmfchile.cl/documentacion/UF.html "https://api.cmfchile.cl/documentacion/UF.html"
[2]: https://mindicador.cl/ "https://mindicador.cl/"
[3]: https://resend.com/pricing "https://resend.com/pricing"
[4]: https://www.cloudflare.com/application-services/products/turnstile/ "https://www.cloudflare.com/application-services/products/turnstile/"
[5]: https://developers.cloudflare.com/r2/pricing/ "https://developers.cloudflare.com/r2/pricing/"
[6]: https://wiki.openstreetmap.org/wiki/Pt%3ATile_usage_policy "https://wiki.openstreetmap.org/wiki/Pt%3ATile_usage_policy"
[7]: https://www.maptiler.com/cloud/pricing/ "https://www.maptiler.com/cloud/pricing/"
[8]: https://mapsplatform.google.com/pricing/ "https://mapsplatform.google.com/pricing/"
[9]: https://www.digitalocean.com/pricing/droplets "https://www.digitalocean.com/pricing/droplets"
[10]: https://vercel.com/pricing "https://vercel.com/pricing"
[11]: https://render.com/pricing "https://render.com/pricing"
[12]: https://developers.cloudflare.com/workers/platform/pricing/ "https://developers.cloudflare.com/workers/platform/pricing/"
