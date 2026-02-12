# poc-apartment

Plataforma (PoC) para corredoras de propiedades: **portal público** (búsqueda + detalle + lead) + **portal admin** (publicación, media, leads, métricas).

> Estado del repo: **docs-first**. La implementación de código (web/api/db/worker) se construye incrementalmente en iteraciones (ver “Documentación y decisiones”).

---

## Objetivo

1) Fijar claramente **requisitos e invariantes** del producto.  
2) Reducir “alucinaciones” y retrabajo usando:
   - decisiones explícitas (ADRs),
   - contrato-first (OpenAPI/DTOs cuando corresponda),
   - migraciones forward-only,
   - quality gates (checklists + CI cuando exista).

---

## Invariantes del producto (v1)

- Multi-tenant: una **organización** (corredora) con múltiples usuarios (jefa + corredores) y RBAC por organización.
- Operaciones: **arriendo y venta** (sin pasarela de pagos).
- Monedas: **UF + CLP siempre visibles**; UF se actualiza automáticamente y recalcula precios.
- Ubicación: **exacta** (dirección + lat/lng visibles).
- Leads: formulario + tracking; WhatsApp/CRM como extensión futura.
- Público navega sin login; favoritos y búsquedas guardadas requieren login de visitante.

> Si un cambio toca estos puntos, debe ir con ADR.

---

## Documentación y decisiones (source of truth)

### Normativa del repo
- `Protocolo IA.md` → reglas anti-alucinación + flujo de trabajo del agente.
- `Response Template.md` → estructura obligatoria para respuestas/diseños.
- `ADR Template.md` → plantilla para decisiones importantes.

### Iteraciones
- `Primera iteracion.md`
- `Segunda Iteracion.md`
- `Tercera iteracion.md`

### Otros
- `Primera estimacion de costos.md`

> Nota: a medida que crezca el repo, esta documentación puede moverse a `docs/` manteniendo este README como índice.

---

## Cómo trabajar (contribuciones)

### Regla #1 — No inventar
- No se inventan endpoints, modelos ni comportamientos.
- Si falta información: declarar lo desconocido, listar supuestos mínimos y proponer alternativas con trade-offs.

### Regla #2 — Cambios grandes requieren ADR
Ejemplos: stack, auth, modelo de precios UF/CLP, estrategia de búsqueda, multi-tenant, storage/derivados de imágenes, etc.

### Regla #3 — Incremental (PRs pequeños)
- PRs verificables y reversibles.
- Migraciones **forward-only**.
- Cambios destructivos en 2 fases.

---

## Estructura objetivo del repo (cuando se implemente código)

> Esta sección es intencionalmente “objetivo”, no estado actual.

- `docs/`
  - `docs/ai/` (memoria, guías, quality gate)
  - `docs/adr/` (ADRs)
  - `docs/product/` (alcance, iteraciones)
  - `docs/real-estate/` (dominio: listings, leads, UF, media, etc.)
- `apps/`
  - `apps/web/` (portal público + admin)
  - `apps/api/` (REST /v1)
  - `apps/worker/` (jobs: UF, media, notificaciones)
- `db/`
  - `db/migrations/` (SQL forward-only)

---

## Desarrollo local

⚠️ Aún no hay instrucciones de ejecución porque la implementación no está fijada en el repo.

Cuando se agregue código, este README deberá incluir:
- prerequisitos (runtime, DB, etc.),
- `.env.example`,
- comandos estándar (dev/test/lint/migrate),
- guía “primer boot” (seed mínimo).

---

## Seguridad (mínimos desde el día 1)

- No commitear secretos (usar `.env.example` + secrets en CI).
- Rate-limit en endpoints públicos sensibles (leads/auth/events).
- No loggear tokens/PII sensible.
- Errores sanitizados hacia cliente.

---

## Licencia

TBD.
