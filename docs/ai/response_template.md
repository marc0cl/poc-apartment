# Plantilla de Respuesta del Agente de IA (GPT)
> Usar esta estructura para responder a requerimientos de arquitectura y coding.

---

## 1) Supuestos (si faltan datos)
- S1: ...
- S2: ...
- S3: ...

## 2) Objetivo y alcance
- Objetivo:
- Fuera de alcance (por ahora):
- Criterios de éxito:

## 3) Contexto técnico observado (si aplica)
- Stack actual:
- Convenciones del repo:
- Restricciones:
- Fuentes consultadas (paths):
  - ...

## 4) Diseño propuesto
### 4.1 Arquitectura
- Capas / módulos:
- Dependencias:
- Estrategia de multi-tenant (si aplica):

### 4.2 Modelo de datos
- Entidades:
- Relaciones:
- Índices clave:
- Consistencia / transacciones:

### 4.3 API / Contratos
- Endpoints:
- Request/Response:
- Headers:
- Errores:
- Paginación/sorting:
- Validaciones:

### 4.4 Concurrencia / Paralelismo / Jobs
- ¿CPU o I/O bound?
- ¿Qué se paraleliza y por qué?
- Límite de concurrencia:
- Timeouts + cancelación:
- Retries:

### 4.5 Caché
- Qué se cachea:
- TTL:
- Invalidación:
- ETag/versioning:

### 4.6 Observabilidad
- Logs:
- Métricas:
- Traces:

## 5) Plan incremental (PRs)
- PR1:
- PR2:
- PR3:

## 6) Tests
- Unit:
- Integration:
- E2E:
- Casos borde:

## 7) Riesgos y mitigaciones
- Riesgo:
- Mitigación:

## 8) Implementación (si se solicitó código)
- Archivos a crear/modificar:
- Snippets o diffs:
- Comandos (lint/test/migrate):
EOF

cat <<'EOF' > docs/ai/AI_QUALITY_GATE.md
###
# Quality Gate del Agente de IA (GPT)
> Checklist para finalizar una entrega sin deuda oculta.

---

## 1) Correctness
- [ ] Tipos/contratos coherentes
- [ ] Edge cases principales cubiertos
- [ ] Manejo de errores consistente (sin silencios)
- [ ] Idempotencia en endpoints críticos (cuando aplica)

## 2) Seguridad
- [ ] Validación de inputs (tipo/rango/formato)
- [ ] Autenticación y autorización correctas
- [ ] Rate limit en endpoints públicos sensibles
- [ ] No se loggean secretos

## 3) Datos y migraciones
- [ ] Migraciones forward-only
- [ ] Cambios destructivos en dos fases
- [ ] Índices razonables para queries de listado
- [ ] Evitar N+1 / batch cuando aplica

## 4) Performance / Escala
- [ ] Identificación de CPU vs I/O bound
- [ ] Concurrencia limitada (no spawn infinito)
- [ ] Timeouts/cancelación en I/O externo
- [ ] Caché definida e invalidación clara

## 5) Observabilidad
- [ ] request-id/correlation id
- [ ] logs estructurados con contexto
- [ ] métricas mínimas (latencia, errores, colas)

## 6) Mantenibilidad
- [ ] SRP/Separation of concerns
- [ ] No duplicación obvia (DRY con criterio)
- [ ] Nombres claros
- [ ] Tests mínimos en el path crítico

## 7) UX/UI (si aplica)
- [ ] Accesibilidad básica (focus, labels)
- [ ] Estados de carga/error visibles
- [ ] No bloquear hilo principal con cálculos pesados
