```md
# docs/real-estate/00-decisiones-cerradas.md

# Decisiones cerradas (v1)

## 1) Multi-tenant + multi-usuario (corredores)
- El sistema soporta **múltiples organizaciones** (`organizations`) y dentro de cada una múltiples usuarios (jefa + corredores).
- Autorización en admin por **RBAC** (roles por organización).

## 2) Tipos de operación
- Se soporta **Arriendo** y **Venta** (sin pasarela de pagos).

## 3) Monedas y UF (CLP + UF)
- Se muestran **siempre ambos valores** en cada anuncio:
  - **UF** (principal de referencia).
  - **CLP** (valor calculado según UF vigente).
- La UF debe actualizarse automáticamente (diaria) y recalcular precios.

## 4) Ubicación
- La ubicación se muestra **exacta** (dirección + lat/lng).

## 5) Leads / comunicación
- V1: formulario de lead + tracking de conversiones.
- Futuro: **WhatsApp** como canal de comunicación principal (link + tracking desde v1).

## 6) Favoritos y búsquedas guardadas (requieren login)
- Visitantes pueden navegar sin login.
- Para **favoritos** y **guardar búsqueda**: requiere autenticación de visitante (cuentas públicas).
```

```md
# docs/real-estate/01-arquitectura-tecnologica.md

# Arquitectura Tecnológica (v1) - Ajustada a decisiones cerradas

## 1) Frontend (público + admin)
- Framework React con SSR/ISR para SEO y performance.
- Admin en ruta `/admin` (misma app), protegido por auth.
- Público:
  - listado/búsqueda
  - detalle de anuncio
  - favoritos / búsquedas guardadas (solo logueado)

## 2) Backend (API + workers)
Servicios recomendados (separación lógica aunque partan en monorepo):
- `api`: REST (OpenAPI 3.1 contract-first)
- `worker`: jobs (UF, thumbnails, notificaciones, stats)
- `webhook/internal`: endpoints internos protegidos (purga cache, reindex, etc.)

## 3) Almacenamiento de multimedia (fotos + planos)
- Bucket privado para drafts + Signed URLs para upload directo.
- Bucket público para publicados + CDN (cache agresiva).
- Derivados:
  - fotos: webp 400/800/1200/1600 + cover
  - planos: PDF + preview PNG (opcional)

## 4) Multi-tenant (organizaciones) + dominios
- `organizations` como tenant.
- Opcional white-label:
  - `orgslug.tu-dominio.com` o dominio custom.
  - Resolución por `Host` → lookup en `organization_domains`.
- Marketplace (opcional):
  - `tu-dominio.com` puede listar anuncios de múltiples organizaciones.

## 5) UF y precios (CLP ↔ UF)
### 5.1 Fuente UF (recomendada)
- Job diario que consume UF desde una fuente primaria (requiere credenciales/keys).
- Guardar UF en tabla `fx_rates`.
- Recalcular `listings.price_uf_amount` y `listings.price_clp_amount` para que el API devuelva ambos valores ya listos (fast reads).

### 5.2 Caching y coherencia
- El job de UF incrementa una versión `system_settings.uf_version` o actualiza `fx_rates.current`.
- El API incluye `uf_version` en ETag para que los caches se revaliden cuando cambia la UF.

## 6) Autenticación
- Admin: sesión con refresh token (cookie HttpOnly) + access token corto.
- Público (visitantes): auth simple (email OTP/magic link recomendado) para habilitar:
  - favoritos
  - búsquedas guardadas

## 7) Observabilidad (desde el día 1)
- Logs estructurados + request id.
- Métricas (latencia, errores, throughput).
- Tracing distribuido (API + worker).

## 8) Benchmarks UX que conviene replicar (sin copiar UI)
- Cards con precio en UF, dormitorios, baños, m² y CTA visible.
- CTA estilo “solicitud online”/“agendar visita” como acción de alto valor.
- Disclaimer legal: imágenes referenciales y precios sujetos a disponibilidad (práctica común en inmobiliarias).
```

````md
# docs/real-estate/02-api-rest.md

# API REST v1 (actualizada)
Base URL: `https://api.tu-dominio.com/v1`

## 0) Convenciones
- JSON: `Content-Type: application/json`
- Errores: `application/problem+json`
- Auth admin: `Authorization: Bearer <token>` + refresh token en cookie
- Auth público: `Authorization: Bearer <token>` (o cookie) según implementación
- Multi-tenant público (si aplica white-label):
  - Header: `X-Tenant: <org_slug>` (opcional)
  - Alternativa: resolver por `Host` en gateway y setear `X-Tenant`.

---

## 1) Representación de precio (SIEMPRE UF + CLP)

En responses de listings (list y detail), el precio se entrega así:

```json
{
  "price": {
    "operation": "rent",
    "period": "month",
    "base": { "currency": "UF", "amount": 16.5 },
    "display": {
      "uf": { "amount": 16.5 },
      "clp": { "amount": 654000 }
    },
    "fx": {
      "uf_clp": { "value": 39698.37, "date": "2026-02-03" },
      "version": 123
    }
  }
}
```

Reglas:
- `base.currency` ∈ {`UF`,`CLP`}
- `display` siempre incluye ambos.
- `fx.date` debe ser el día de UF usada.
- `fx.version` sirve para invalidación de caches.

---

## 2) Público - Listings

### 2.1 GET /listings
Query params:
- `operation` = `rent|sale`
- `property_type` = `house|apartment|land|office|parking|warehouse`
- `q`
- `comuna_id` (o `comuna_slug`)
- `bedrooms_min`, `bathrooms_min`, `parking_min`
- `min_area_total_m2`, `min_area_built_m2`
- `min_price_uf`, `max_price_uf` (filtro principal recomendado)
- `min_price_clp`, `max_price_clp` (opcional; internamente normalizar)
- `sort=published_at:desc|price_uf:asc|price_clp:asc`
- `limit` (1..50), `cursor`

Response 200 (ejemplo item):
```json
{
  "items": [
    {
      "id": "uuid",
      "slug": "depto-2d1b-las-condes-metro-manquehue-1234",
      "title": "Departamento 2D1B a pasos del metro",
      "operation": "rent",
      "status": "published",
      "price": {
        "operation": "rent",
        "period": "month",
        "base": { "currency": "UF", "amount": 16.5 },
        "display": { "uf": { "amount": 16.5 }, "clp": { "amount": 654000 } },
        "fx": { "uf_clp": { "value": 39698.37, "date": "2026-02-03" }, "version": 123 }
      },
      "attributes": { "bedrooms": 2, "bathrooms": 1, "total_area_m2": 52.0, "built_area_m2": 48.0 },
      "address": { "street": "Av. Ejemplo 123", "comuna": "Las Condes", "city": "Santiago", "region": "RM" },
      "location": { "lat": -33.42, "lng": -70.61, "precision": "exact" },
      "media": {
        "cover": {
          "asset_id": "uuid",
          "url": "https://cdn.tu-dominio.com/listings/<id>/cover_1200.webp"
        }
      },
      "published_at": "2026-02-07T12:00:00Z"
    }
  ],
  "page": { "next_cursor": "..." }
}
```

---

### 2.2 GET /listings/{id_or_slug}
Response 200 incluye:
- descripción
- amenities
- fotos + planos
- ubicación exacta (lat/lng + dirección)

---

## 3) Público - Leads (contacto)

### 3.1 POST /leads
Request:
```json
{
  "listing_id": "uuid",
  "name": "Juan Pérez",
  "email": "[email protected]",
  "phone": "+56911111111",
  "message": "Hola, me interesa visitar.",
  "preferred_contact": "whatsapp",
  "consent": { "privacy_policy": true },
  "utm": { "source": "google", "medium": "cpc", "campaign": "arriendos_rm" }
}
```

Validaciones:
- `listing_id` debe estar publicado
- requerir al menos `email` o `phone`
- rate limit por IP + listing

Response 201:
```json
{ "id": "uuid", "status": "new", "created_at": "..." }
```

---

## 4) Público - Auth visitante (para favoritos y búsquedas guardadas)

> Recomendación v1: Email OTP / Magic link (reduce fricción y evita gestión de passwords).
> Si prefieres password, agrega `/public/auth/register` + reset password.

### 4.1 POST /public/auth/start
Request:
```json
{ "email": "[email protected]" }
```

Comportamiento:
- genera OTP o token de magic link (guardado hasheado + expira)
- envía email

Response 202:
```json
{ "sent": true }
```

### 4.2 POST /public/auth/verify
Request:
```json
{ "email": "[email protected]", "code": "123456" }
```

Response 200:
```json
{
  "access_token": "<jwt>",
  "expires_in": 900,
  "refresh_token": "<opaque>",
  "user": { "id": "uuid", "email": "[email protected]" }
}
```

### 4.3 GET /public/me
Auth: Bearer
Response 200:
```json
{ "id": "uuid", "email": "[email protected]", "created_at": "..." }
```

---

## 5) Público - Favoritos (requiere login)

### 5.1 POST /public/me/favorites
Request:
```json
{ "listing_id": "uuid" }
```
Response 201:
```json
{ "ok": true }
```

### 5.2 GET /public/me/favorites?limit=20&cursor=...
Response 200:
```json
{
  "items": [
    { "listing_id": "uuid", "created_at": "...", "listing": { "id": "uuid", "title": "...", "slug": "...", "price": { "...": "..." } } }
  ],
  "page": { "next_cursor": "..." }
}
```

### 5.3 DELETE /public/me/favorites/{listing_id}
Response 204

---

## 6) Público - Búsquedas guardadas (requiere login)

### 6.1 POST /public/me/saved-searches
Request:
```json
{
  "name": "Deptos 2D1B Las Condes",
  "filters": {
    "operation": "rent",
    "property_type": "apartment",
    "comuna_id": "uuid",
    "bedrooms_min": 2,
    "bathrooms_min": 1,
    "max_price_uf": 20
  },
  "notify": {
    "enabled": true,
    "frequency": "daily"
  }
}
```

Validaciones:
- `filters` debe ser un subconjunto soportado por GET /listings
- límites máximos de complejidad (evitar queries infinitas)

Response 201:
```json
{ "id": "uuid", "created_at": "..." }
```

### 6.2 GET /public/me/saved-searches
Response 200:
```json
{ "items": [{ "id": "uuid", "name": "...", "filters": { }, "notify": { "enabled": true, "frequency": "daily" } }] }
```

### 6.3 PATCH /public/me/saved-searches/{id}
### 6.4 DELETE /public/me/saved-searches/{id}

---

## 7) Público - Indicadores (UF)

### 7.1 GET /indicators/uf
Response 200:
```json
{
  "indicator": "UF",
  "value_clp": 39698.37,
  "date": "2026-02-03",
  "source": "cmf",
  "version": 123
}
```

---

## 8) Admin - Listings (cambios clave)
- Crear/editar listing permite `price.base` en UF o CLP.
- Publish requiere:
  - ubicación exacta (street + lat/lng)
  - mínimo fotos
  - descripción
- Admin endpoints se mantienen como antes: `/admin/listings`, `/admin/media`, `/admin/leads`, etc.
````

````md
# docs/real-estate/03-modelo-de-datos-y-migraciones.md

# Modelo de Datos (PostgreSQL + PostGIS) - Actualizado

## 1) Cambios clave respecto al esquema base
- Multi-tenant: `organization_id` en entidades admin.
- Precio: almacenar **base** + **derivados** (UF y CLP) para listados rápidos.
- UF: tabla `fx_rates` + job diario.
- Público: `public_users` + `favorites` + `saved_searches`.

---

## 2) Migración: fx_rates + precio dual

### 0003_fx_rates_and_dual_prices.sql
```sql
DO $$ BEGIN
  CREATE TYPE fx_indicator AS ENUM ('UF_CLP');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

CREATE TABLE IF NOT EXISTS fx_rates (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  indicator fx_indicator NOT NULL,
  rate_date date NOT NULL,
  value numeric(18,6) NOT NULL CHECK (value > 0),
  source text NOT NULL CHECK (source IN ('cmf','bcentral','mindicador')),
  fetched_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (indicator, rate_date)
);

-- system settings (simple KV)
CREATE TABLE IF NOT EXISTS system_settings (
  key text PRIMARY KEY,
  value text NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now()
);

INSERT INTO system_settings(key, value)
VALUES ('uf_version', '0')
ON CONFLICT (key) DO NOTHING;

-- Listings: base + derivados UF/CLP
ALTER TABLE listings
  ADD COLUMN IF NOT EXISTS price_base_currency text CHECK (price_base_currency IN ('UF','CLP')) DEFAULT 'UF',
  ADD COLUMN IF NOT EXISTS price_base_amount numeric(14,2) NOT NULL DEFAULT 0 CHECK (price_base_amount >= 0),
  ADD COLUMN IF NOT EXISTS price_uf_amount numeric(14,6) NOT NULL DEFAULT 0 CHECK (price_uf_amount >= 0),
  ADD COLUMN IF NOT EXISTS price_clp_amount numeric(14,2) NOT NULL DEFAULT 0 CHECK (price_clp_amount >= 0),
  ADD COLUMN IF NOT EXISTS price_fx_rate_date date,
  ADD COLUMN IF NOT EXISTS operation text CHECK (operation IN ('rent','sale')) DEFAULT 'rent';

-- (Opcional) si existían columnas antiguas, marcar como deprecated y migrar datos con un script
-- Recomendar: un script 1-off que setea base/derivados usando el fx_rate actual.

CREATE INDEX IF NOT EXISTS ix_listings_price_uf ON listings(price_uf_amount);
CREATE INDEX IF NOT EXISTS ix_listings_price_clp ON listings(price_clp_amount);
CREATE INDEX IF NOT EXISTS ix_listings_operation ON listings(operation);
```

---

## 3) Migración: público (users + favorites + saved searches)

### 0004_public_users_favorites_saved_searches.sql
```sql
CREATE TABLE IF NOT EXISTS public_users (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email citext NOT NULL UNIQUE,
  is_active boolean NOT NULL DEFAULT true,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- OTP/magic link codes (hasheados)
CREATE TABLE IF NOT EXISTS public_auth_codes (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email citext NOT NULL,
  code_hash text NOT NULL,
  expires_at timestamptz NOT NULL,
  consumed_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now()
);

-- refresh sessions
CREATE TABLE IF NOT EXISTS public_sessions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  public_user_id uuid NOT NULL REFERENCES public_users(id) ON DELETE CASCADE,
  refresh_token_hash text NOT NULL,
  expires_at timestamptz NOT NULL,
  revoked_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (public_user_id, refresh_token_hash)
);

-- favorites (por tenant opcional)
CREATE TABLE IF NOT EXISTS listing_favorites (
  public_user_id uuid NOT NULL REFERENCES public_users(id) ON DELETE CASCADE,
  listing_id uuid NOT NULL REFERENCES listings(id) ON DELETE CASCADE,
  organization_id uuid REFERENCES organizations(id) ON DELETE CASCADE,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (public_user_id, listing_id)
);

CREATE INDEX IF NOT EXISTS ix_favorites_user_created ON listing_favorites(public_user_id, created_at DESC);

-- saved searches
CREATE TABLE IF NOT EXISTS saved_searches (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  public_user_id uuid NOT NULL REFERENCES public_users(id) ON DELETE CASCADE,
  organization_id uuid REFERENCES organizations(id) ON DELETE CASCADE,
  name text NOT NULL,
  filters jsonb NOT NULL DEFAULT '{}'::jsonb,
  notify_enabled boolean NOT NULL DEFAULT false,
  notify_frequency text CHECK (notify_frequency IN ('daily','weekly')) DEFAULT 'daily',
  last_notified_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS ix_saved_searches_user ON saved_searches(public_user_id);
```

---

## 4) Migración: multi-tenant domains (white-label)

### 0005_organization_domains.sql
```sql
CREATE TABLE IF NOT EXISTS organization_domains (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  domain text NOT NULL UNIQUE,
  is_primary boolean NOT NULL DEFAULT false,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX IF NOT EXISTS ux_org_primary_domain
ON organization_domains(organization_id)
WHERE is_primary = true;
```

---

## 5) Ubicación exacta
- Mantener `properties.geo` como `geography(Point, 4326)` con índice GiST.
- En v1 se valida que `location.precision = exact` y que existan `lat/lng`.

---

## 6) Jobs críticos (worker)
- `fx_update_uf_daily`: fetch UF → upsert `fx_rates` → increment `system_settings.uf_version`.
- `recompute_listing_prices`: actualizar `price_uf_amount` y `price_clp_amount` en listings.
- `media_process_asset`: thumbnails/derivados.
- `stats_aggregate_daily`: agrega views/leads → `daily_listing_stats`.
````

```md
# docs/real-estate/06-uf-y-precios.md

# UF + Precios (UF/CLP) - Diseño para “siempre actualizado” y lectura rápida

## 1) Objetivo
- Un anuncio debe mostrar:
  - `UF` (monto)
  - `CLP` (monto calculado)
  - `fecha UF`
- Debe escalar a miles de lecturas sin recalcular en cada request de listado.

## 2) Estrategia
- `listings.price_base_currency` + `listings.price_base_amount` = valor que define el corredor.
- Job diario:
  1) Obtiene UF (UF→CLP)
  2) Guarda en `fx_rates` (indicator UF_CLP)
  3) Recalcula derivados en `listings`:
     - si base=UF: `price_uf_amount = base`, `price_clp_amount = base * uf_value`
     - si base=CLP: `price_clp_amount = base`, `price_uf_amount = base / uf_value`
  4) actualiza `listings.price_fx_rate_date = fx_rate.rate_date`
  5) incrementa `system_settings.uf_version`

## 3) Cache correctness
- El API:
  - devuelve `fx.version` en responses de listing.
  - usa ETag incorporando `uf_version` → cuando sube la UF, el ETag cambia.
- CDN:
  - puede cachear GET /listings por 60–120s (o más) sin riesgo de mostrar CLP “viejo” por días.
  - si quieres “casi exactitud instantánea”, baja TTL, pero normalmente 60–300s es buen balance.

## 4) Fallback de fuentes
- Primaria: proveedor oficial con API key.
- Secundaria: proveedor alternativo (si falla primaria) con alertas.
- Registrar `fx_rates.source` para auditoría.

## 5) Reglas UX (front)
- Mostrar:
  - `UF 16,5`
  - `CLP $654.000 (UF al 2026-02-03)`
- En listados, CLP en 2da línea como pediste.
```

```md
# docs/real-estate/07-favoritos-y-busquedas-guardadas.md

# Favoritos y Búsquedas Guardadas (requiere login visitante)

## 1) Favoritos (UX)
- En card de listado:
  - ícono corazón (toggle)
  - si no está logueado: modal “Inicia sesión para guardar favoritos”
- En header:
  - “Favoritos (N)”
- Página:
  - `/favoritos` → lista paginada, orden desc por fecha.

## 2) Saved Searches (UX)
- En página de resultados:
  - botón “Guardar búsqueda”
  - nombre editable (default: “Mi búsqueda”)
  - toggle “Avisarme por email” (daily/weekly)
- Página:
  - `/mis-busquedas` → CRUD + ejecutar búsqueda.

## 3) Modelo de filtros (backend)
- `saved_searches.filters` es JSON con subset de query params de `/listings`.
- Validar en backend:
  - solo keys soportadas
  - límites de rangos numéricos
  - `limit` y `cursor` NO se guardan dentro de filtros
- Para ejecutar una búsqueda guardada:
  - front toma `filters` y llama a GET `/listings` con esos params.

## 4) Notificaciones (opcional fase 2)
- Job diario:
  - para cada saved_search notify_enabled=true:
    - consulta anuncios publicados desde `last_notified_at`
    - si hay nuevos: email resumen
    - actualiza `last_notified_at`
- Evitar spam:
  - digest (no email por cada anuncio)
  - “mute” por usuario
```

Fuentes que soportan decisiones UX y UF:

* Cards inmobiliarias con **precio en UF**, dormitorios/baños/m² y CTA tipo “solicitud online de visita” (ejemplos en resultados). ([Portal Inmobiliario][1])
* Disclaimer típico en inmobiliarias (imágenes referenciales, precios sujetos a disponibilidad) y flujo de “agendar visita”. ([Inmobiliaria Aitue][2])
* UF vía API CMF (detalle de parámetros y endpoint UF del día; requiere API key). ([api.cmfchile.cl][3])
* Alternativa de respaldo (UF en mindicador). ([mindicador.cl][4])

Referencia de estándares de calidad para el repo/IA (guía que subiste):



[1]: https://www.portalinmobiliario.com/venta/departamento/3-dormitorios/santiago-metropolitana "https://www.portalinmobiliario.com/venta/departamento/3-dormitorios/santiago-metropolitana"
[2]: https://www.aitue.cl/ "https://www.aitue.cl/"
[3]: https://api.cmfchile.cl/documentacion/UF.html "https://api.cmfchile.cl/documentacion/UF.html"
[4]: https://mindicador.cl/api/uf "https://mindicador.cl/api/uf"
