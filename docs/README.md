# Documentación — Índice

## A) IA / Normativa (ley del repo)
- Protocolo (cómo opera la IA): `ai/protocolo.md`
- Plantilla de respuesta (estructura obligatoria): `ai/response_template.md`
- Memoria del proyecto (snapshot + invariantes): `ai/memoria_proyecto.md`
- Quality Gate (checklist de salida): `ai/quality_gate.md`
- PR Spec Template (cómo pedir/entregar PRs): `ai/pr_spec_template.md`

## B) ADRs (decisiones importantes)
- Plantilla ADR: `adr/template_adr.md`
- ADRs (cuando existan): `adr/NNN_<slug>.md`

Convención recomendada:
- `NNN_<slug>.md` (ej: `001_auth_publico_magic_link.md`)
- En cada ADR: contexto, decisión, alternativas, consecuencias, notas.

## C) Iteraciones (historia y aprendizaje)
Carpeta: `iterations/`
- `001_...`
- `002_primera_iteracion.md`
- `003_segunda_iteracion.md`
- `004_tercera_iteracion.md`

## D) Source of Truth (recomendado como próximo PR)
Cuando el contenido se estabilice, mover/extraer desde iteraciones a una carpeta dedicada:
- `product/` o `real_estate/` (a definir)
para: decisiones cerradas, API, datos, UX/perf.
