Stack recomendada (2026): **Next.js 16** para frontend SSR/ISR y **React 19.2** para UI (según documentación oficial). ([Next.js][1])
Patrones UX a imitar: “busca por ubicación + tipo de propiedad” (como Aitue) y cards con **precio, dormitorios, baños, m²** y CTA tipo “solicitud online de visita” (como Portalinmobiliario). ([Inmobiliaria Aitue][2])
Referencia de estándares de código (para el repo/IA): -estate/00-contexto-y-requisitos.md

# Plataforma Profesional para Corredora de Propiedades

## Alcance, Requisitos y Criterios de Escalabilidad

## 1) Objetivo del producto

Construir una plataforma web para **publicar y administrar propiedades** (arriendo inicialmente, venta opcional), con:

* **Portal público**: búsqueda, filtros, detalle de propiedad, galería de fotos, planos (PDF/imagen), dimensiones y atributos, contacto/lead.
* **Portal privado (corredora / equipo)**: login, gestión de anuncios (draft/publish), carga de multimedia, administración de leads, métricas y reporting.

**Condición clave**: el sistema debe ser **read-heavy** (miles de visitantes) y **write-light** (pocas usuarias/os administradores).

---

## 2) Actores / Roles

### Público (sin login)

* Visitante (browsing)
* Interesado (envía lead, solicita visita)

### Privado (con login)

* `ADMIN`: gestiona organización, usuarios, configuración, catálogos (amenities), moderación.
* `AGENT`: crea/edita anuncios, sube imágenes, gestiona leads.
* `ANALYST` (opcional): acceso a métricas/exports sin permisos de edición.

---

## 3) Flujos principales

### 3.1 Portal público

1. Buscar propiedades (ubicación + tipo + filtros).
2. Ver resultados (cards).
3. Abrir detalle de anuncio:

   * Galería (fotos)
   * Planos (PDF/imagen)
   * Atributos: m² totales, m² construidos, dormitorios, baños, pisos, estacionamientos, bodega, orientación, antigüedad, etc.
   * Mapa (ubicación aproximada configurable)
   * CTA: contactar / solicitar visita
4. Enviar lead:

   * nombre, email, teléfono, mensaje, preferencia horario (opcional)
5. Confirmación + track de conversión.

### 3.2 Portal privado (corredora)

1. Login
2. Dashboard:

   * anuncios publicados vs borradores
   * leads recientes (últimas 24/48h)
   * rendimiento (views/leads/conversión por anuncio)
3. Crear/editar anuncio:

   * datos principales, precio, atributos, ubicación, amenities
   * subir fotos/planos
   * previsualización
4. Publicar / Despublicar / Archivar
5. Gestión de leads:

   * pipeline (NEW → CONTACTED → QUALIFIED → CLOSED / SPAM)
   * notas internas y asignación

---

## 4) Requisitos no funcionales (NFR)

### 4.1 Rendimiento / escala

* **TTFB bajo** en páginas públicas (SSR/ISR + CDN).
* **Imágenes optimizadas**: variantes (thumb/medium/large) + CDN.
* **Cacheabilidad**:

  * detalle anuncio cacheable con invalidación al publicar/editar.
  * búsqueda cacheable por 30–120s con SWR.
* API preparada para picos de tráfico (rate-limit + autoscaling).
* DB optimizada para filtros frecuentes + geosearch.

### 4.2 Disponibilidad

* Target: 99.9% (dependiendo del presupuesto).
* Deploys sin downtime (rolling).
* Backups automáticos DB + retención.

### 4.3 Seguridad

* Auth fuerte para admin (OIDC/OAuth2.1 o password+2FA).
* RBAC por organización.
* Protección anti-spam (rate limit + captcha opcional) en leads.
* Firmas (signed URLs) para uploads; control de acceso a drafts.

### 4.4 SEO

* URLs estables (slug), sitemap, meta tags.
* Structured data (schema.org) para propiedades (opcional).
* SSR/ISR para indexación.

---

## 5) Decisiones abiertas (para cerrar diseño fino)

* ¿Solo una corredora (single-tenant) o varias corredoras en el futuro (multi-tenant)?
* ¿Arriendo únicamente o arriendo + venta?
* ¿Moneda: CLP/UF? ¿Ambas? ¿Conversión diaria requerida?
* ¿Mostrar ubicación exacta o aproximada (privacidad)?
* ¿Integración de WhatsApp/CRM (HubSpot, etc.) para leads?
* ¿Exigir login a visitantes para “favoritos” o “guardar búsqueda”?

`````

````md
# docs/real-estate/01-arquitectura-tecnologica.md

# Arquitectura Tecnológica Propuesta
## Enfoque: escalable, cache-friendly, contractual (OpenAPI), y optimizado para lectura

## 1) Stack recomendado (opción A: rápida y performante)

### Frontend
- **Next.js (App Router)**: SSR/ISR + caché en edge/CDN.
- **React** + **TypeScript**.
- UI: 
  - Opción 1: Tailwind + design tokens con CSS custom properties.
  - Opción 2: CSS Modules + variables CSS.
- Cliente de API: generado desde OpenAPI (ej. orval/openapi-typescript) para tipado end-to-end.

### Backend API
- **Go** (API REST):
  - Router: `chi` o `gin` (ligero, alto rendimiento).
  - PostgreSQL driver: `pgx`.
  - Queries tipadas: `sqlc` (evita ORMs pesados).
  - Observabilidad: OpenTelemetry (traces/metrics), logs estructurados (zap).
- OpenAPI 3.1 como contrato de verdad (generar stubs, clientes, docs).

### Data / Infra
- **PostgreSQL 16+** + **PostGIS** (búsqueda geográfica).
- Redis (cache, rate-limit, locks simples).
- Object storage:
  - **GCS Bucket + Cloud CDN** o **S3 + CloudFront**.
  - Acceso por signed upload URLs (direct-to-storage).
- Cola / jobs:
  - GCP Pub/Sub o Cloud Tasks (o SQS si AWS).
- Email/transaccional:
  - SendGrid / Resend / SES (según cloud).

---

## 2) Componentes (alto nivel)

- `web`: Next.js (público + admin bajo /admin)
- `api`: servicio Go (REST)
- `worker`: procesador de imágenes / publicación / notificaciones
- `db`: Postgres + PostGIS
- `cache`: Redis
- `storage-private`: bucket privado (drafts)
- `storage-public`: bucket público (publicados) + CDN
- `analytics-ingest`: endpoint de eventos → cola → agregación (daily stats)

---

## 3) Principios arquitectónicos clave

### 3.1 Separar lectura pública de escritura privada
- Lectura pública: cacheable, escalable, barata.
- Escritura privada: validaciones fuertes, auditoría y RBAC.

### 3.2 Multimedia fuera de la API
- La API no debe “recibir” megabytes.
- Flujo recomendado:
  1) API genera `upload_url` (signed) + `object_key`
  2) Frontend sube directo al bucket
  3) API confirma y encola procesamiento (thumbnails, metadata)

### 3.3 Observabilidad desde el día 1
- Request ID en todo request.
- Traces API + worker.
- Métricas: latencia endpoints, errores, colas, tiempos de procesamiento de imágenes.

---

## 4) Estrategia de caching

### 4.1 Páginas públicas (Next.js)
- Home y landing: ISR (revalidate 5–15 min).
- Detalle anuncio: ISR (revalidate 5–10 min) + invalidación por webhook interno al publicar/editar.
- Listado/búsqueda:
  - SSR con caching en edge (60–120s).
  - O fetch server-side con `stale-while-revalidate`.

### 4.2 API pública
- GET /listings y GET /listings/{id} cacheables.
- ETag + If-None-Match (ahorra payload).
- CDN puede cachear JSON si controlas headers.

---

## 5) Escalabilidad de base de datos

- Índices para filtros frecuentes (precio, dormitorios, baños, comuna, tipo, status).
- PostGIS para geofiltro (radio/box) con índice GiST.
- Full-text search simple en Postgres (tsvector) o OpenSearch si crece mucho.
- Read replicas si se dispara el tráfico.
- Conexiones: pooler (PgBouncer) si la plataforma crece.

---

## 6) Alternativa (opción B: enterprise JVM)
- API: Java 21 + Spring Boot 3
- Migraciones: Flyway
- Similar enfoque para storage, cache y jobs
`````

````md
# docs/real-estate/02-api-rest.md

# API REST v1 - Especificación práctica (contract-first)

## 0) Convenciones generales

### Base URL
- `https://api.tu-dominio.com/v1`

### Content-Type
- Requests JSON: `Content-Type: application/json`
- Responses JSON: `Content-Type: application/json; charset=utf-8`
- Errores: `Content-Type: application/problem+json`

### Headers estándar
- `X-Request-ID: <uuid>` (opcional en request; obligatorio en response)
- `Accept-Language: es-CL` (opcional)
- `User-Agent: ...` (cliente)
- `Authorization: Bearer <access_token>` (solo admin)
- `Idempotency-Key: <uuid>` (para POST críticos, recomendado)
- `If-Match: "<etag>"` (para updates con control de concurrencia)
- `If-None-Match: "<etag>"` (para cache 304 en GET)

### Formato de error (Problem Details)
Ejemplo:
```json
{
  "type": "https://tu-dominio.com/problems/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "One or more fields are invalid.",
  "instance": "/v1/admin/listings",
  "errors": [
    { "field": "price.amount", "code": "min", "message": "must be >= 0" }
  ],
  "request_id": "b9f0cfe7-3d8e-4a74-9a6d-3c1aef2f4c4d"
}
```

### Paginación
- Preferir **cursor pagination** en listados:
  - `limit` (1..100)
  - `cursor` (string opaco)
- Respuesta:
```json
{
  "items": [],
  "page": { "next_cursor": "..." }
}
```

### Sorting
- `sort=published_at:desc`
- `sort=price.amount:asc`
- Si hay múltiples: `sort=published_at:desc,price.amount:asc`

### Filtros (principio)
- Solo filtros indexables en DB en v1.
- Evitar queries “infinitas”; si crece, pasar a Search service.

---

## 1) Autenticación (Admin)

### 1.1 POST /auth/login
**Uso**: login email+password (si NO usas OIDC externo).  
**Auth**: none  
**Idempotency-Key**: opcional

Request:
```json
{
  "email": "admin@tu-dominio.com",
  "password": "********"
}
```

Validaciones:
- email válido
- password min 10 chars

Response 200:
```json
{
  "access_token": "<jwt>",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "<opaque>",
  "user": {
    "id": "uuid",
    "role": "ADMIN",
    "organization_id": "uuid",
    "email": "admin@tu-dominio.com",
    "name": "Nombre"
  }
}
```

Errores:
- 401 credenciales inválidas
- 423 usuario bloqueado (opcional)

> Recomendación profesional: usar refresh_token en cookie HttpOnly + SameSite=Lax (si front y api comparten dominio).

---

### 1.2 POST /auth/refresh
Request:
```json
{ "refresh_token": "<opaque>" }
```

Response 200:
```json
{
  "access_token": "<jwt>",
  "expires_in": 900
}
```

Errores:
- 401 refresh inválido/expirado

---

### 1.3 POST /auth/logout
Auth: Bearer
Request:
```json
{ "refresh_token": "<opaque>" }
```
Response 204

---

### 1.4 GET /me
Auth: Bearer  
Response 200:
```json
{
  "id": "uuid",
  "organization_id": "uuid",
  "role": "AGENT",
  "email": "agent@tu-dominio.com",
  "name": "Agente"
}
```

---

## 2) Público - Listados y detalle de anuncios

### 2.1 GET /listings
**Auth**: none  
**Cache**: `public, max-age=60, stale-while-revalidate=600`

Query params (principales):
- `transaction_type` = `rent|sale` (default: rent)
- `property_type` = `house|apartment|land|office|parking|warehouse`
- `region_id`, `city_id`, `comuna_id` (ids internos)
- `q` (texto: título/descripción/dirección)
- `min_price`, `max_price`
- `currency` (CLP|UF|USD)
- `bedrooms_min`, `bathrooms_min`, `floors_min`
- `min_area_total_m2`, `min_area_built_m2`
- `parking_min`
- `amenities[]=pool&amenities[]=terrace`
- `pet_friendly=true|false`
- `furnished=true|false`
- `geo_lat`, `geo_lng`, `geo_radius_m` (si habilitas búsqueda radial)
- `limit` (1..50)
- `cursor` (opaco)
- `sort` (ej: `published_at:desc`)

Response 200:
```json
{
  "items": [
    {
      "id": "uuid",
      "slug": "depto-2d1b-las-condes-metro-manquehue-1234",
      "title": "Departamento 2D1B a pasos del metro",
      "transaction_type": "rent",
      "status": "published",
      "price": { "amount": 650000, "currency": "CLP", "period": "month" },
      "address": {
        "comuna": "Las Condes",
        "city": "Santiago",
        "region": "Región Metropolitana"
      },
      "attributes": {
        "bedrooms": 2,
        "bathrooms": 1,
        "floors": 1,
        "total_area_m2": 52.0,
        "built_area_m2": 48.0,
        "parking": 1
      },
      "media": {
        "cover": {
          "asset_id": "uuid",
          "url": "https://cdn.tu-dominio.com/listings/<listingId>/cover_1200.webp",
          "width": 1200,
          "height": 800
        }
      },
      "published_at": "2026-02-07T12:00:00Z"
    }
  ],
  "page": { "next_cursor": "eyJ..." }
}
```

Validaciones de query:
- `limit` rango
- precios >= 0
- geo params: lat [-90..90], lng [-180..180], radius <= 20000 (ej.)

Errores:
- 400 parámetros inválidos
- 429 rate limit

---

### 2.2 GET /listings/{listing_id_or_slug}
**Auth**: none  
**Cache**: `public, max-age=60, stale-while-revalidate=600`

Path param:
- `{listing_id_or_slug}`: UUID o slug.

Response 200:
```json
{
  "id": "uuid",
  "slug": "casa-4d3b-vina-del-mar-jardin-xyz",
  "title": "Casa 4D3B con patio grande",
  "transaction_type": "rent",
  "status": "published",
  "description": {
    "format": "markdown",
    "value": "Descripción larga..."
  },
  "price": { "amount": 1200000, "currency": "CLP", "period": "month" },
  "fees": { "common_expenses": { "amount": 80000, "currency": "CLP", "period": "month" } },
  "property": {
    "property_type": "house",
    "year_built": 2015,
    "address": {
      "street": "Av. Ejemplo 123",
      "comuna": "Viña del Mar",
      "city": "Viña del Mar",
      "region": "Valparaíso",
      "postal_code": "0000000",
      "display_mode": "approximate"
    },
    "location": {
      "lat": -33.024,
      "lng": -71.551,
      "precision": "approximate"
    },
    "attributes": {
      "bedrooms": 4,
      "bathrooms": 3,
      "floors": 2,
      "total_area_m2": 220.0,
      "built_area_m2": 160.0,
      "lot_area_m2": 220.0,
      "parking": 2,
      "storage": 1,
      "pet_friendly": true,
      "furnished": false
    },
    "amenities": [
      { "code": "terrace", "label": "Terraza" },
      { "code": "bbq", "label": "Quincho" }
    ]
  },
  "media": {
    "photos": [
      {
        "asset_id": "uuid",
        "url": "https://cdn.tu-dominio.com/listings/<listingId>/photo_1_1600.webp",
        "thumb_url": "https://cdn.tu-dominio.com/listings/<listingId>/photo_1_400.webp",
        "width": 1600,
        "height": 1067,
        "alt": "Fachada principal"
      }
    ],
    "floorplans": [
      {
        "asset_id": "uuid",
        "type": "pdf",
        "url": "https://cdn.tu-dominio.com/listings/<listingId>/plan_1.pdf",
        "title": "Plano primer piso"
      }
    ]
  },
  "contact": {
    "preferred_channels": ["form", "whatsapp"],
    "whatsapp_number": "+56912345678"
  },
  "published_at": "2026-02-07T12:00:00Z",
  "updated_at": "2026-02-07T12:30:00Z"
}
```

Errores:
- 404 no existe / no publicado
- 410 si está archivado y deseas ocultarlo públicamente

---

### 2.3 GET /amenities
Catálogo público para filtros.

Response 200:
```json
{
  "items": [
    { "code": "pool", "label": "Piscina", "category": "recreation" },
    { "code": "elevator", "label": "Ascensor", "category": "building" }
  ]
}
```

---

## 3) Público - Leads (contacto / solicitud)

### 3.1 POST /leads
**Auth**: none  
**Rate limit**: estricto (por IP y por listing)  
**Spam control**: captcha opcional + honeypot

Headers:
- `Idempotency-Key` recomendado
- `X-Forwarded-For` (lo inyecta el LB)

Request:
```json
{
  "listing_id": "uuid",
  "name": "Juan Pérez",
  "email": "[email protected]",
  "phone": "+56911111111",
  "message": "Hola, me interesa visitar el fin de semana.",
  "preferred_contact": "whatsapp",
  "utm": {
    "source": "google",
    "medium": "cpc",
    "campaign": "arriendos_rm"
  },
  "consent": {
    "privacy_policy": true
  }
}
```

Validaciones:
- listing_id obligatorio y existente (status=published)
- name 2..120
- email válido (opcional si phone viene, pero exigir al menos uno)
- phone en formato E.164 si se entrega
- message 0..2000
- consent.privacy_policy = true

Response 201:
```json
{
  "id": "uuid",
  "status": "new",
  "created_at": "2026-02-07T12:40:00Z"
}
```

Errores:
- 422 validación
- 429 rate limit
- 409 idempotency key replay (si decides bloquear)

---

## 4) Público - Eventos (analítica)

### 4.1 POST /events
**Auth**: none  
**Uso**: ingestion ligera hacia cola (NO DB directo por evento).

Request:
```json
{
  "event_type": "listing_view",
  "listing_id": "uuid",
  "anon_id": "a_anon_cookie_id",
  "session_id": "s_session_cookie_id",
  "referrer": "https://google.com",
  "page": "/arriendo/depto-2d1b-las-condes-...",
  "ts": "2026-02-07T12:00:00Z"
}
```

Validaciones:
- event_type enum: listing_view, lead_submit, phone_click, whatsapp_click
- listing_id requerido para eventos del listing
- anon_id/session_id max length 128

Response 202 (accepted):
```json
{ "accepted": true }
```

---

## 5) Admin - CRUD de anuncios

> Todos los endpoints /admin requieren `Authorization: Bearer ...` y RBAC por organización.

### 5.1 POST /admin/listings
Crea un anuncio en estado `draft`.

Request:
```json
{
  "transaction_type": "rent",
  "title": "Departamento nuevo 2D1B",
  "property_type": "apartment",
  "price": { "amount": 650000, "currency": "CLP", "period": "month" },
  "address": {
    "street": "Av. Ejemplo 123",
    "comuna_id": "uuid",
    "display_mode": "approximate"
  },
  "location": {
    "lat": -33.42,
    "lng": -70.61
  }
}
```

Validaciones:
- title 10..120
- price.amount >= 0
- lat/lng válidos
- comuna_id existente
- display_mode enum: exact|approximate|hidden

Response 201:
```json
{
  "id": "uuid",
  "status": "draft",
  "slug": "departamento-nuevo-2d1b-av-ejemplo-123",
  "created_at": "2026-02-07T12:45:00Z"
}
```

---

### 5.2 GET /admin/listings
Query:
- `status=draft|published|archived`
- `q=...`
- `limit`, `cursor`, `sort`

Response 200:
```json
{
  "items": [
    { "id": "uuid", "title": "....", "status": "draft", "updated_at": "..." }
  ],
  "page": { "next_cursor": "..." }
}
```

---

### 5.3 GET /admin/listings/{id}
Response 200 (incluye campos de edición y media con estado).

---

### 5.4 PATCH /admin/listings/{id}
Actualiza campos del draft/published con control de concurrencia.

Headers:
- `If-Match: "<etag>"` requerido (evita overwrites)

Request (ejemplo):
```json
{
  "title": "Departamento nuevo 2D1B (actualizado)",
  "description": { "format": "markdown", "value": "..." },
  "attributes": {
    "bedrooms": 2,
    "bathrooms": 1,
    "floors": 1,
    "total_area_m2": 52.0,
    "built_area_m2": 48.0,
    "parking": 1,
    "pet_friendly": false,
    "furnished": false
  },
  "amenities": ["elevator", "pool"]
}
```

Validaciones:
- atributos numéricos >= 0
- total_area_m2 >= built_area_m2 (si ambos presentes)
- amenities codes válidos
- description length <= 20000

Response 200:
```json
{
  "id": "uuid",
  "status": "draft",
  "etag": "\"d1f7a...\"",
  "updated_at": "2026-02-07T13:00:00Z"
}
```

Errores:
- 412 precondition failed (etag no coincide)
- 422 validación

---

### 5.5 POST /admin/listings/{id}/publish
Publica el anuncio.

Reglas de publicación (hard checks):
- title + description presentes
- price válido
- address/comuna presentes
- al menos 3 fotos y 1 cover
- lat/lng presentes (si display_mode != hidden)

Response 200:
```json
{
  "id": "uuid",
  "status": "published",
  "published_at": "2026-02-07T13:05:00Z"
}
```

---

### 5.6 POST /admin/listings/{id}/unpublish
Vuelve a draft (opcional) o “private”.

Response 200:
```json
{ "id": "uuid", "status": "draft" }
```

---

### 5.7 POST /admin/listings/{id}/archive
Archiva (no visible público).

Response 200:
```json
{ "id": "uuid", "status": "archived", "archived_at": "..." }
```

---

## 6) Admin - Multimedia (fotos/planos)

### 6.1 POST /admin/listings/{id}/media/upload-init
Request:
```json
{
  "kind": "photo",
  "filename": "livingroom.jpg",
  "content_type": "image/jpeg",
  "byte_size": 3456789,
  "checksum_sha256": "base64..."
}
```

Validaciones:
- kind enum: photo|floorplan|document
- content_type whitelist:
  - photo: image/jpeg, image/png, image/webp
  - floorplan: application/pdf, image/*
- byte_size max (ej 20MB foto, 50MB pdf)

Response 201:
```json
{
  "asset_id": "uuid",
  "object_key": "org/<orgId>/listings/<listingId>/<assetId>/original.jpg",
  "upload": {
    "url": "https://storage-provider/signed-url",
    "method": "PUT",
    "headers": {
      "Content-Type": "image/jpeg"
    },
    "expires_at": "2026-02-07T13:10:00Z"
  }
}
```

---

### 6.2 POST /admin/media/{asset_id}/upload-complete
Request:
```json
{
  "listing_id": "uuid"
}
```

Response 202:
```json
{
  "asset_id": "uuid",
  "status": "processing"
}
```

Worker:
- genera variantes webp (400, 800, 1200, 1600)
- extrae metadata (width/height)
- si PDF: genera preview png (opcional)
- mueve/copia a bucket público si listing está publicado

---

### 6.3 PATCH /admin/listings/{id}/media/reorder
Request:
```json
{
  "items": [
    { "asset_id": "uuid1", "order": 1, "is_cover": true, "alt": "Fachada" },
    { "asset_id": "uuid2", "order": 2, "is_cover": false, "alt": "Living" }
  ]
}
```

Validaciones:
- orden 1..N sin duplicados
- exactamente 1 cover si hay media

Response 200:
```json
{ "ok": true }
```

---

### 6.4 DELETE /admin/media/{asset_id}
Elimina asociación y opcionalmente objeto en storage (si no está referenciado).

Response 204

---

## 7) Admin - Leads

### 7.1 GET /admin/leads
Query:
- `status=new|contacted|qualified|closed|spam`
- `listing_id=uuid` (opcional)
- `from`, `to` (ISO date)
- `limit`, `cursor`, `sort=created_at:desc`

Response:
```json
{
  "items": [
    {
      "id": "uuid",
      "listing_id": "uuid",
      "name": "Juan Pérez",
      "email": "[email protected]",
      "phone": "+56911111111",
      "message": "...",
      "status": "new",
      "created_at": "..."
    }
  ],
  "page": { "next_cursor": "..." }
}
```

---

### 7.2 PATCH /admin/leads/{id}
Request:
```json
{
  "status": "contacted",
  "assigned_to": "uuid",
  "note": "Llamado realizado, agendar visita sábado."
}
```

Validaciones:
- status enum
- assigned_to pertenece a la organización

Response 200:
```json
{ "id": "uuid", "status": "contacted" }
```

---

## 8) Admin - Métricas

### 8.1 GET /admin/metrics/overview?from=YYYY-MM-DD&to=YYYY-MM-DD
Response 200:
```json
{
  "range": { "from": "2026-02-01", "to": "2026-02-07" },
  "totals": {
    "listing_views": 12450,
    "unique_visitors": 8400,
    "leads": 132,
    "conversion_rate": 0.0106
  },
  "top_listings": [
    {
      "listing_id": "uuid",
      "title": "Depto 2D1B ...",
      "views": 2300,
      "leads": 22,
      "conversion_rate": 0.0096
    }
  ]
}
```

---

### 8.2 GET /admin/metrics/listings/{listing_id}?from=...&to=...
Response 200:
```json
{
  "listing_id": "uuid",
  "series": [
    { "day": "2026-02-01", "views": 120, "leads": 2 },
    { "day": "2026-02-02", "views": 180, "leads": 1 }
  ]
}
```
````

````md
# docs/real-estate/03-modelo-de-datos-y-migraciones.md

# Modelo de datos + Estrategia de migraciones (PostgreSQL)

## 1) Principios de modelado

- Primary keys: UUID (`gen_random_uuid()`).
- Multi-tenant ready: `organization_id` en entidades sensibles (listings, leads, media).
- Read-heavy: índices para filtros típicos.
- Geosearch: PostGIS `geography(Point, 4326)` + GiST.
- Campos “core” como columnas (para filtrar); atributos raros a `jsonb` (con moderación).

---

## 2) Entidades (resumen)

- `organizations`: corredora/equipo.
- `users`: agentes/admins.
- `properties`: entidad física (dirección, geo, atributos base).
- `listings`: anuncio publicable (precio, texto, status, publicación).
- `media_assets`: archivos en storage (foto/plano/pdf).
- `listing_media`: orden + cover + alt.
- `amenities` + `listing_amenities`: catálogo de características.
- `leads`: contactos entrantes por anuncio.
- `lead_events`: historial (cambio de estado, notas, asignación).
- `daily_listing_stats`: agregados diarios (views, leads).
- `audit_log`: auditoría de cambios.

---

## 3) Migraciones (SQL) - ejemplo

> Herramienta sugerida:
> - Go: `golang-migrate` (SQL forward-only)
> - Java: Flyway (SQL forward-only)

### 0001_init.sql
```sql
-- Extensions
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS citext;
CREATE EXTENSION IF NOT EXISTS postgis;

-- Enums
DO $$ BEGIN
  CREATE TYPE listing_status AS ENUM ('draft', 'published', 'archived');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE transaction_type AS ENUM ('rent', 'sale');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE property_type AS ENUM ('house', 'apartment', 'land', 'office', 'parking', 'warehouse');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE media_kind AS ENUM ('photo', 'floorplan', 'document');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE media_status AS ENUM ('pending', 'uploaded', 'processing', 'ready', 'failed');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE lead_status AS ENUM ('new', 'contacted', 'qualified', 'closed', 'spam');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE address_display_mode AS ENUM ('exact', 'approximate', 'hidden');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

-- Organizations
CREATE TABLE IF NOT EXISTS organizations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  slug text NOT NULL UNIQUE,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE IF NOT EXISTS users (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  email citext NOT NULL,
  name text NOT NULL,
  role text NOT NULL CHECK (role IN ('ADMIN','AGENT','ANALYST')),
  password_hash text, -- null si usas OIDC externo
  is_active boolean NOT NULL DEFAULT true,
  last_login_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (organization_id, email)
);

-- Properties (entidad física)
CREATE TABLE IF NOT EXISTS properties (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  property_type property_type NOT NULL,

  street text,
  street_number text,
  comuna_id uuid, -- catálogo interno (o tabla comunas)
  comuna_name text,
  city_name text,
  region_name text,
  postal_code text,

  address_display_mode address_display_mode NOT NULL DEFAULT 'approximate',

  geo geography(Point, 4326),
  
  bedrooms smallint CHECK (bedrooms >= 0),
  bathrooms smallint CHECK (bathrooms >= 0),
  floors smallint CHECK (floors >= 0),
  parking smallint CHECK (parking >= 0),
  storage smallint CHECK (storage >= 0),

  total_area_m2 numeric(10,2) CHECK (total_area_m2 >= 0),
  built_area_m2 numeric(10,2) CHECK (built_area_m2 >= 0),
  lot_area_m2 numeric(10,2) CHECK (lot_area_m2 >= 0),

  year_built smallint CHECK (year_built >= 1800 AND year_built <= 2100),

  pet_friendly boolean,
  furnished boolean,

  extra jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- Listings (anuncio publicable)
CREATE TABLE IF NOT EXISTS listings (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  property_id uuid NOT NULL REFERENCES properties(id) ON DELETE RESTRICT,

  transaction_type transaction_type NOT NULL DEFAULT 'rent',
  status listing_status NOT NULL DEFAULT 'draft',

  title text NOT NULL,
  slug text NOT NULL,
  description_markdown text,

  price_amount numeric(14,2) NOT NULL DEFAULT 0 CHECK (price_amount >= 0),
  price_currency text NOT NULL CHECK (price_currency IN ('CLP','UF','USD')),
  price_period text NOT NULL CHECK (price_period IN ('month','total')),

  common_expenses_amount numeric(14,2) CHECK (common_expenses_amount >= 0),
  common_expenses_currency text CHECK (common_expenses_currency IN ('CLP','UF','USD')),

  available_from date,

  contact_email citext,
  contact_phone text,
  whatsapp_phone text,

  published_at timestamptz,
  archived_at timestamptz,

  search_tsv tsvector,

  created_by uuid REFERENCES users(id),
  updated_by uuid REFERENCES users(id),

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  UNIQUE (organization_id, slug)
);

-- Media assets
CREATE TABLE IF NOT EXISTS media_assets (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  kind media_kind NOT NULL,
  status media_status NOT NULL DEFAULT 'pending',

  storage_provider text NOT NULL CHECK (storage_provider IN ('gcs','s3')),
  bucket text NOT NULL,
  object_key text NOT NULL,

  content_type text NOT NULL,
  byte_size integer NOT NULL CHECK (byte_size >= 0),
  width integer CHECK (width >= 0),
  height integer CHECK (height >= 0),

  checksum_sha256 text,

  created_by uuid REFERENCES users(id),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- Listing <-> media (orden y cover)
CREATE TABLE IF NOT EXISTS listing_media (
  listing_id uuid NOT NULL REFERENCES listings(id) ON DELETE CASCADE,
  asset_id uuid NOT NULL REFERENCES media_assets(id) ON DELETE RESTRICT,
  sort_order integer NOT NULL CHECK (sort_order >= 1),
  is_cover boolean NOT NULL DEFAULT false,
  alt_text text,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (listing_id, asset_id)
);

CREATE UNIQUE INDEX IF NOT EXISTS ux_listing_one_cover
ON listing_media(listing_id)
WHERE is_cover = true;

-- Amenities
CREATE TABLE IF NOT EXISTS amenities (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  code text NOT NULL UNIQUE,
  label text NOT NULL,
  category text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS listing_amenities (
  listing_id uuid NOT NULL REFERENCES listings(id) ON DELETE CASCADE,
  amenity_id uuid NOT NULL REFERENCES amenities(id) ON DELETE RESTRICT,
  PRIMARY KEY (listing_id, amenity_id)
);

-- Leads
CREATE TABLE IF NOT EXISTS leads (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  listing_id uuid NOT NULL REFERENCES listings(id) ON DELETE CASCADE,

  name text NOT NULL,
  email citext,
  phone text,
  message text,

  preferred_contact text CHECK (preferred_contact IN ('email','phone','whatsapp')),

  status lead_status NOT NULL DEFAULT 'new',
  assigned_to uuid REFERENCES users(id),

  utm jsonb NOT NULL DEFAULT '{}'::jsonb,
  meta jsonb NOT NULL DEFAULT '{}'::jsonb,

  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS lead_events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id uuid NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
  actor_user_id uuid REFERENCES users(id),
  event_type text NOT NULL CHECK (event_type IN ('status_change','note','assign')),
  payload jsonb NOT NULL DEFAULT '{}'::jsonb,
  created_at timestamptz NOT NULL DEFAULT now()
);

-- Daily stats (agregados)
CREATE TABLE IF NOT EXISTS daily_listing_stats (
  listing_id uuid NOT NULL REFERENCES listings(id) ON DELETE CASCADE,
  day date NOT NULL,
  views integer NOT NULL DEFAULT 0 CHECK (views >= 0),
  unique_visitors integer NOT NULL DEFAULT 0 CHECK (unique_visitors >= 0),
  leads integer NOT NULL DEFAULT 0 CHECK (leads >= 0),
  PRIMARY KEY (listing_id, day)
);

-- Audit log (cambios admin)
CREATE TABLE IF NOT EXISTS audit_log (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  actor_user_id uuid REFERENCES users(id),
  entity_type text NOT NULL,
  entity_id uuid NOT NULL,
  action text NOT NULL,
  before jsonb,
  after jsonb,
  created_at timestamptz NOT NULL DEFAULT now()
);

-- Indexes (búsqueda y filtros)
CREATE INDEX IF NOT EXISTS ix_listings_status_published_at ON listings(status, published_at DESC);
CREATE INDEX IF NOT EXISTS ix_listings_org_status ON listings(organization_id, status);
CREATE INDEX IF NOT EXISTS ix_listings_price ON listings(price_currency, price_amount);
CREATE INDEX IF NOT EXISTS ix_properties_type ON properties(property_type);
CREATE INDEX IF NOT EXISTS ix_properties_attrs ON properties(bedrooms, bathrooms, parking);
CREATE INDEX IF NOT EXISTS ix_properties_geo ON properties USING GIST (geo);

-- Full-text (simple)
CREATE INDEX IF NOT EXISTS ix_listings_search_tsv ON listings USING GIN (search_tsv);
```

### 0002_triggers.sql (updated_at + search_tsv)
```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DO $$ BEGIN
  CREATE TRIGGER trg_org_updated_at BEFORE UPDATE ON organizations
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TRIGGER trg_users_updated_at BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TRIGGER trg_properties_updated_at BEFORE UPDATE ON properties
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TRIGGER trg_listings_updated_at BEFORE UPDATE ON listings
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TRIGGER trg_media_assets_updated_at BEFORE UPDATE ON media_assets
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TRIGGER trg_leads_updated_at BEFORE UPDATE ON leads
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
EXCEPTION WHEN duplicate_object THEN NULL; END $$;


-- Search TSV update
CREATE OR REPLACE FUNCTION update_listing_search_tsv()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_tsv =
    setweight(to_tsvector('spanish', coalesce(NEW.title, '')), 'A') ||
    setweight(to_tsvector('spanish', coalesce(NEW.description_markdown, '')), 'B');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DO $$ BEGIN
  CREATE TRIGGER trg_listings_search_tsv
  BEFORE INSERT OR UPDATE OF title, description_markdown
  ON listings FOR EACH ROW
  EXECUTE FUNCTION update_listing_search_tsv();
EXCEPTION WHEN duplicate_object THEN NULL; END $$;
```

---

## 4) Notas de evolución (migraciones futuras)

- `0003_catalog_geo.sql`: tablas `regions/cities/comunas` normalizadas.
- `0004_listing_versions.sql`: versionado de anuncios (historial de edición).
- `0005_search_service.sql`: si migras a OpenSearch/Meilisearch, conservar Postgres como source-of-truth.
- `0006_media_derivatives.sql`: tabla para variantes generadas (webp sizes, pdf previews).
````

```md
# docs/real-estate/04-ux-ui-y-rendimiento.md

# UX/UI + Performance (orientado a inmobiliarias grandes)

## 1) Sitemap / Rutas

### Público
- `/` Home: buscador + destacados + confianza (sellos, contacto)
- `/arriendo` Listado/búsqueda (filtros + resultados)
- `/arriendo/[slug]` Detalle anuncio (galería + atributos + contacto)
- `/mapa` (opcional) vista mapa
- `/legal/privacidad` + `/legal/terminos`

### Admin
- `/admin/login`
- `/admin` Dashboard (métricas)
- `/admin/listings` Mis anuncios
- `/admin/listings/new`
- `/admin/listings/[id]/edit`
- `/admin/leads`
- `/admin/settings` (amenities, usuarios, marca)

---

## 2) Patrones UX clave (benchmarking)

### Búsqueda simple primero, filtros después
- Top bar:
  - Ubicación (comuna/ciudad) + tipo de propiedad (como patrón de buscador de inmobiliarias). 
- Resultados:
  - Cards con:
    - precio
    - dormitorios/baños
    - m² (útiles o totales)
    - CTA visible (contactar / solicitar visita)
  - Ordenamientos:
    - “Más recientes”
    - “Menor precio”
    - “Mayor precio”
    - “Más vistos” (opcional)

### Detalle de anuncio pensado para conversión
- Primer pantallazo:
  - galería (cover grande + thumbnails)
  - precio y badges (nuevo, rebajado, disponible desde)
  - atributos principales (chips: 2D, 1B, 52m², 1E)
  - CTA fijo (sticky): “Contactar”, “WhatsApp”, “Solicitar visita”
- Secciones:
  - descripción
  - características / amenities
  - plano (pdf/imagen) con preview
  - ubicación (mapa) con control de precisión (exact/approx/hidden)
  - “propiedades similares” (misma comuna + rango precio)

---

## 3) Performance web (lo que permite miles de visitantes)

### Front
- SSR/ISR para `/arriendo/[slug]` (indexable, rápido).
- Prefetch de rutas al hover (Next).
- Imágenes:
  - usar variantes y `srcset`
  - lazy-load bajo el fold
  - prioridad solo para cover

### CDN / Cache
- CDN delante del bucket público.
- JSON cacheable con `ETag`.
- Búsqueda cache 60–120s (SWR).

### Backend
- GET públicos sin joins pesados:
  - denormalizar datos necesarios en `listings` o usar views.
- rate-limit:
  - agresivo para POST /leads y /events
  - moderado para GET públicos
- jobs:
  - thumbnails/floorplan previews async (no bloquear publish)

---

## 4) Admin UX (productividad para corredora)

### Dashboard
- Cards:
  - Views 7d
  - Leads 7d
  - Conversión 7d
  - Top 5 anuncios
- Tabla leads recientes con quick actions (contacted/qualified)

### Editor de anuncio
- Wizard por pasos (reduce errores):
  1) Datos básicos
  2) Ubicación
  3) Atributos
  4) Multimedia
  5) Publicación (checklist)
- Checklist de publish con “errores accionables” (por campo, no genéricos).

---

## 5) Decisiones de UX para cerrar (impactan modelo/API)
- ¿Se muestra “m² útiles” y “m² construidos” como campos separados?
- ¿Se requiere “gasto común” siempre en arriendo?
- ¿Se soporta “solicitud online de visita” (agenda) como flujo?
- ¿Se implementan “favoritos” para público (requiere identidad o cookie)?
```

[1]: https://nextjs.org/blog/next-16 "https://nextjs.org/blog/next-16"
[2]: https://www.aitue.cl/ "https://www.aitue.cl/"
