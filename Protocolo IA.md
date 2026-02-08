# Protocolo del Agente de IA (GPT) para Desarrollo de Software
> Este documento define **cómo debe razonar y responder** el agente de IA al diseñar, generar, refactorizar o revisar código en este repositorio.
> Objetivo: **precisión, cero alucinaciones, planificación sólida, escalabilidad, mantenibilidad y performance con criterio**.

---

## 0) Mandamientos (NO negociables)

### 0.1 Veracidad y anti-alucinación
- **Prohibido inventar**:
  - APIs, endpoints, flags, librerías, versiones, funciones, comportamientos, resultados de benchmarks o “best practices” sin comprobar.
  - estructura del repo si no está visible.
- Si falta información, el agente debe:
  - **declarar explícitamente lo desconocido**.
  - **proponer opciones** y explicar trade-offs.
  - **hacer supuestos mínimos**, enumerados y razonables, y continuar.
- **Nunca afirmar** que se corrieron tests, builds, migraciones, linters o benchmarks si no se ejecutaron.
- Diferenciar siempre:
  - **Hechos** (observados en el repo o provistos por el usuario)
  - **Supuestos**
  - **Recomendaciones**

### 0.2 Respeta el contexto del repo
- Antes de proponer cambios grandes (framework, estructura, patrón), **leer señales del repo**:
  - dependencias (package.json / go.mod / build.gradle / pom.xml)
  - convenciones de carpetas
  - estilo (lint/formatter)
  - patrones existentes (arquitectura, errores, DTOs, etc.)
- **No introducir** una tecnología “porque es moderna” si:
  - el repo ya resolvió ese problema con un enfoque válido (ej: CSS vars vs Sass),
  - o el costo de migración supera el beneficio.

### 0.3 Calidad > cantidad
- Prioridad: **Correctness > Seguridad > Simplicidad > Observabilidad > Performance > “elegancia”**
- No sobre-ingenierizar por reflejo. Pero:
  - si el problema es claramente **read-heavy** / **concurrencia alta** / **I/O intensivo**, entonces sí diseñar para eso desde el inicio.

---

## 1) Flujo de trabajo obligatorio (por cada tarea)

### 1.1 Entender el objetivo
El agente debe identificar y escribir:
- Qué se quiere construir/cambiar.
- Restricciones: stack, versiones, compatibilidad, tiempos, equipo, SLA.
- Si es backend: carga esperada, picos, latencia, consistencia, privacidad.
- Si es frontend: SEO, accesibilidad, performance, estado, caché.

### 1.2 Proponer un plan antes de codear
- Presentar un **plan breve en pasos**, con:
  - arquitectura/capas,
  - modelos de datos,
  - endpoints/contratos,
  - migraciones,
  - strategy de caché/concurrencia,
  - estrategia de tests.
- El plan debe ser **incremental** (PRs pequeños y verificables), no un “big bang”.

### 1.3 Validar el plan (calidad y coherencia)
Antes de escribir código, revisar:
- ¿Respeta convenciones y patrones del repo?
- ¿Minimiza acoplamiento y repeticiones?
- ¿Escala el camino crítico de lectura?
- ¿Cumple seguridad base (auth, rate-limit, input validation, logs)?
- ¿Tiene una historia clara de migración?

### 1.4 Implementación incremental
- Preferir cambios pequeños por commit/PR.
- Mantener cambios localizados (evitar refactors masivos sin necesidad).
- Reusar utilidades existentes si son correctas.
- Si se agrega dependencia:
  - justificar,
  - evaluar mantenimiento y superficie de ataque,
  - pin/versionado razonable.

### 1.5 Quality Gate antes de dar por terminado
El agente debe “pasar” mentalmente esta lista:
- ¿Compila/build?
- ¿Tests unitarios clave listos?
- ¿Integración clave cubierta?
- ¿Errores y edge-cases cubiertos?
- ¿Observabilidad mínima (logs, métricas)?
- ¿Documentación / OpenAPI / migraciones actualizadas?

---

## 2) Política de decisión: preguntar vs asumir

### Preguntar (mínimo) si afecta:
- Seguridad (auth, permisos, exposición de datos).
- Integridad de datos (migraciones destructivas).
- Reglas de negocio ambiguas.
- Performance/escala (si el diseño cambia mucho según carga).

### Asumir y avanzar si:
- Es un detalle de UX/label menor.
- La opción por defecto es estándar y reversible.

En ambos casos: documentar supuestos.

---

## 3) Escalabilidad y performance (heurísticas obligatorias)

### 3.1 Clasificar el tipo de carga
Antes de proponer optimizaciones, clasificar:
- **CPU-bound**: parsers, imágenes, cálculos, agregaciones pesadas.
- **I/O-bound**: DB, red, storage, APIs externas.

### 3.2 Regla de oro de paralelismo/concurrencia
- Paralelizar sólo si:
  - las tareas son **independientes**,
  - hay ganancia clara (latencia o throughput),
  - existe control de recursos (pool, backpressure, timeouts).
- Si hay N llamadas externas independientes:
  - proponer **concurrencia con límite** (pool/semaphore), no “spawn infinito”.
- Siempre agregar:
  - **timeouts**
  - **cancellation** (context)
  - **reintentos con backoff** sólo en errores transitorios
  - **idempotencia** cuando aplica

### 3.3 Caching con criterio
- Cachear lecturas masivas (público) con invalidación clara.
- Evitar cachés “mágicos” sin estrategia:
  - TTL + ETag + revalidación,
  - “purge/invalidate” en publish/update.

### 3.4 Batching
- Preferir batch de queries o llamadas a servicios en vez de N+1.
- Evitar loops que disparan I/O por elemento.

### 3.5 Índices y queries
- Si hay filtros frecuentes: diseñar índices.
- Para listados: usar paginación consistente (cursor recomendado).
- Medir: `EXPLAIN ANALYZE` (Postgres) antes de asumir.

---

## 4) Reglas de seguridad base (SIEMPRE)

- Input validation:
  - tipos, rangos, enums, formatos (email, phone, uuid).
- AuthN/AuthZ:
  - RBAC por tenant/organización.
  - principle of least privilege.
- Rate limiting:
  - fuerte en endpoints públicos (leads, login, eventos).
- Anti-spam:
  - honeypot/captcha opcional + heurísticas + “cooldown”.
- Logs:
  - nunca loggear secretos (tokens, passwords).
  - usar request-id y correlación.
- Errores:
  - devolver errores sanitizados al cliente.
  - logs con contexto para diagnosticar.

---

## 5) Contratos (API, DTOs, tipos) como fuente de verdad

- Preferir **contract-first**:
  - OpenAPI (REST), o esquema JSON, o tipos TS compartidos.
- DTOs separados del modelo de dominio (evitar acoplar DB ↔ API).
- Versionado:
  - `/v1/...` y cambios breaking → `/v2`.
- Siempre especificar:
  - headers requeridos
  - query params
  - cuerpos de request/response
  - errores (Problem Details)
  - paginación
  - sorting/filtering soportado

---

## 6) Migraciones de BD (reglas de oro)

- Migraciones **forward-only** (evitar downgrades como “plan A”).
- Cambios destructivos en dos fases:
  1) agregar columna/tabla nueva, escribir dual
  2) backfill
  3) cambiar lecturas
  4) eliminar lo viejo
- Nunca bloquear tablas grandes sin plan:
  - evitar `ALTER TABLE ...` costosos en horas pico
  - preferir técnicas seguras (según DB)
- Mantener “source-of-truth”:
  - si se usa Postgres: SQL explícito o tool generadora pero controlada.

---

## 7) Estándares de código del repo (normativos)

El agente debe seguir la guía de mejores prácticas del repo:
- Java: tipado fuerte, excepciones con criterio, concurrencia con `java.util.concurrent`, evitar presión de GC.
- TypeScript: strict mode, evitar `any`, preferir `unknown`, tipos discriminados.
- CSS/SCSS: consistencia, especificidad baja, variables CSS, modularidad.
- Go: estilo idiomático, gofmt, interfaces pequeñas, context, canales/locks con criterio.
- Rust: ownership, `Result/Option`, evitar `unwrap` en producción, async vs threads con criterio.
- React: pure components, hooks rules, estado mínimo, effects como escape hatch.

---

## 8) Formato de salida del agente (para respuestas y PRs)

### 8.1 Cuando el usuario pide diseño
Responder con secciones:
- **Supuestos**
- **Arquitectura** (diagrama textual si aplica)
- **Modelos de datos**
- **Endpoints**
- **Migraciones**
- **Estrategia de caché / concurrencia**
- **Estrategia de pruebas**
- **Riesgos y mitigaciones**
- **Plan incremental (PR1, PR2, PR3)**

### 8.2 Cuando el usuario pide implementación
- Indicar rutas de archivos y cambios concretos.
- No escribir “comentarios” en el código que repitan el prompt del usuario.
- Preferir código directo, claro y testeable.
- Añadir tests mínimos que validen el comportamiento solicitado.

---

## 9) Autochecklist antes de responder (rápido)

- [ ] ¿Estoy inventando algo?
- [ ] ¿Lo que propongo encaja con el repo y su stack?
- [ ] ¿Mencioné supuestos explícitos?
- [ ] ¿Hay riesgos de seguridad / datos / performance no tratados?
- [ ] ¿Propuse concurrencia/caché solo si aplica?
- [ ] ¿Incluí plan incremental y validaciones?