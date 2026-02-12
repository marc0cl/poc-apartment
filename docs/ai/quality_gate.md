# Quality Gate — Entregas de IA

Checklist obligatorio antes de dar una entrega por “terminada”.

## 1) Veracidad
- [ ] No inventé endpoints/modelos/stack/comandos/resultados.
- [ ] Declaré Supuestos (S1, S2, ...) cuando faltó info.
- [ ] Incluí “Fuentes consultadas” con rutas reales del repo.

## 2) Diseño / Arquitectura (si aplica)
- [ ] El camino crítico (read-heavy vs write-heavy) está identificado.
- [ ] Hay contratos claros (DTOs/OpenAPI si aplica).
- [ ] Decisiones grandes → ADR (o lo propuse explícitamente).

## 3) Seguridad (si aplica)
- [ ] AuthN/AuthZ por tenant cuando corresponda.
- [ ] Rate limit / anti-spam en endpoints públicos sensibles.
- [ ] No loggeo secretos ni PII sensible.

## 4) Datos (si aplica)
- [ ] Migraciones forward-only.
- [ ] Índices y paginación para listados.
- [ ] Evité N+1 / I/O dentro de loops.

## 5) Observabilidad (si aplica)
- [ ] request-id y logs estructurados mínimos.
- [ ] Errores consistentes (Problem Details si aplica).

## 6) Entrega
- [ ] Plan incremental por PRs (PR1, PR2, PR3).
- [ ] Riesgos y mitigaciones.
