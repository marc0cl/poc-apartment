# MEMORIA DEL PROYECTO — Apartamentos (v1)
Última actualización: 2026-02-12
Dueño del doc: <nombre/rol>

## 0) One-liner
Plataforma multi-tenant para corredoras de propiedades: portal público (búsqueda/detalle/lead) + portal admin (publicación/medios/leads/métricas).

## 1) Invariantes (NO negociables en v1)
- Multi-tenant por organización + multi-usuario con RBAC.
- Operaciones: arriendo y venta (sin pasarela de pagos).
- Monedas: UF + CLP SIEMPRE visibles; UF se actualiza automáticamente y se refleja en anuncios.
- Ubicación: exacta (dirección + lat/lng visibles) [si esto cambia, requiere ADR].
- Leads: formulario + tracking; WhatsApp/CRM como extensión futura.
- Público navega sin login; favoritos y búsquedas guardadas requieren login de visitante.

Fuente: docs/real-estate/00-decisiones-cerradas.md

## 2) Principios de diseño
- Read-heavy: optimizar lectura (cache/CDN/ETag/índices/paginación por cursor).
- Contract-first: OpenAPI es fuente de verdad; errores Problem Details; versionado /v1.
- Migraciones DB: forward-only; cambios destructivos en 2 fases.
- Seguridad: RBAC por tenant; rate-limit fuerte en endpoints públicos sensibles (leads/auth/events).
- Observabilidad mínima desde día 1: request-id + logs estructurados.

Fuentes:
- Protocolo IA: docs/Protocolo IA.md
- Plantilla de respuesta: docs/Response Template.md

## 3) Glosario (dominio)
- Organization (tenant): corredora/equipo.
- Agent: corredor/a.
- Listing: anuncio publicable (draft/published/archived).
- Property: entidad física (dirección/geo/atributos).
- Lead: contacto entrante por anuncio.
- Asset: foto/plano/documento; derivados (thumbnails/webp).
- UF: indicador diario UF→CLP usado para display y filtros.

## 4) Mapa de documentación (source of truth)
- Decisiones cerradas: docs/real-estate/00-decisiones-cerradas.md
- Arquitectura v1: docs/real-estate/01-arquitectura-tecnologica.md
- API v1: docs/real-estate/02-api-rest.md
- Modelo de datos/migraciones: docs/real-estate/03-modelo-de-datos-y-migraciones.md
- UF y precios: docs/real-estate/06-uf-y-precios.md
- Favoritos/saved searches: docs/real-estate/07-favoritos-y-busquedas-guardadas.md
- Plan local-first/premium-ready: docs/Tercera iteracion.md
- ADRs: docs/ADR Template.md

## 5) Contratos clave (resumen)
- Listados/detalle DEVUELVEN precio UF+CLP + fx(date, version).
- Paginación por cursor (limit + cursor; next_cursor).
- Errores: application/problem+json (Problem Details).
- ETag incorpora versión de UF (uf_version) para invalidación.

## 6) Decisiones abiertas (priorizadas)
1) Stack final (web/backend/infra) para v1.
2) Fuente UF primaria y fallback (y manejo de fallas).
3) Estrategia de auth admin (OIDC vs password+2FA) y auth visitantes (OTP/magic link vs password).
4) Estrategia de storage (localfs vs GCS/S3) y derivados mínimos en v1.
5) Nivel de analítica (events) en v1: mínimo viable vs completo.
6) Búsqueda: Postgres simple vs motor dedicado (Meilisearch/OpenSearch) y cuándo migrar.

## 7) Cómo trabajar (reglas para PRs)
- Antes de codear: plan en el formato Response Template.
- Si la decisión afecta invariantes/stack/seguridad/datos: crear ADR.
- No inventar endpoints/modelos: OpenAPI + migraciones son verdad.
- PRs pequeños, verificables; incluir tests mínimos del path crítico.

## 8) Checklist rápida (para el agente IA)
- ¿Estoy inventando algo no respaldado por docs/contratos?
- ¿Cambiaría una invariante? → ADR.
- ¿Hay validaciones + auth + rate-limit en lo público?
- ¿Incluye paginación/índices/caché en camino read-heavy?
- ¿Logs con request-id y errores Problem Details?
