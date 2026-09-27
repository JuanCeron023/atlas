# 🏛️ Atlas: Enseñanzas y Buenas Prácticas de Ingeniería de Software

> **Compendio de Principios Arquitectónicos, Patrones de Concurrencia y Lecciones de Diseño de Sistemas Distribuidos.**  
> Diseñado como una base de conocimiento de referencia técnica y buenas prácticas universales para agentes de IA y desarrolladores de software.

---

## 🎯 Propósito y Alcance

**Atlas** es un repositorio agnóstico y educativo de **enseñanzas fundamentales de ingeniería de software**. Reúne lecciones aprendidas sobre cómo diseñar, implementar y depurar arquitecturas backend robustas, concurrentes y escalables.

Cualquier agente autónomo (como agentes `forge`) o ingeniero de software que diseñe microservicios, canalizaciones de eventos o esquemas de bases de datos puede consultar Atlas como guía de buenas prácticas universales.

---

## 🧭 Matriz de Guías y Enseñanzas

| Área de Ingeniería | Problemas y Temas Tratados | Guía |
|---|---|---|
| **Concurrencia, Runtimes e Hilos** | Desacoplamiento de contextos en llamadas compartidas, cuotas CFS de CPU en contenedores, control de admisión (load shedding), contención de locks | [`01-concurrency-and-runtime-traps.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/01-concurrency-and-runtime-traps.md) |
| **Flujos de Eventos e Idempotencia** | Claves de partición compuestas, preservación de orden FIFO, reordenamiento en límites de batch, deduplicación e idempotencia | [`02-event-driven-streaming-and-idempotency.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/02-event-driven-streaming-and-idempotency.md) |
| **Manejo del Tiempo y Ventanas** | Turnos continuos extendidos, cálculo en zona horaria local vs UTC, matemáticas de intervalos semiabiertos $[start, end)$, ventanas rodantes | [`03-temporal-data-and-time-windows.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/03-temporal-data-and-time-windows.md) |
| **Contratos de API y Validación** | Tipos puntero para aceptar valores cero (`0`/`false`), semántica `null` vs `0`, borrado accidental en updates parciales, simetría en validadores | [`04-api-contracts-and-validation.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/04-api-contracts-and-validation.md) |
| **Estrategias de Cache y Proxies** | Semántica de métodos HTTP y headers de cache en POSTs, protección contra cache stampedes, invalidación suave (soft deprecation) | [`05-caching-layers-and-edge-proxies.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/05-caching-layers-and-edge-proxies.md) |
| **Transacciones y Persistencia** | Límites transaccionales y exclusión de I/O de red, upserts atómicos vs check-then-insert, tipado estricto en filtros, borrado lógico | [`06-database-transactions-and-queries.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/06-database-transactions-and-queries.md) |
| **Máquinas de Estado y Pipelines** | Jerarquía y precedencia de excepciones manuales sobre reglas automáticas, ciclos de vida Draft/Staged/Live, pipelines concurrentes | [`07-state-machines-and-streaming-pipelines.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/07-state-machines-and-streaming-pipelines.md) |
| **Observabilidad y Diagnóstico** | Propagación de identificadores de correlación, preservación de stack traces en recuperaciones asíncronas, integridad en el origen de errores | [`08-observability-rca-and-triaging.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/08-observability-rca-and-triaging.md) |

---

## ⚡ Los 10 Principios y Enseñanzas Fundamentales

1. **Desacoplar Contextos en Operaciones Compartidas**: Nunca ejecutar trabajo compartido deduplicado (como `singleflight`) bajo el contexto de cancelación de un solo llamante. Si el primero cancela, los demás no deben abortarse en cascada.
2. **Alinear el Runtime con los Límites de CPU del Contenedor**: Los runtimes que leen los núcleos físicos del host asignan hilos de más, provocando estrangulamiento de CPU por CFS. Es necesario sincronizar la concurrencia del runtime con los límites de cgroup.
3. **Excluir Todo I/O de Red de las Transacciones de Base de Datos**: Obtener los datos externos antes de abrir la transacción de base de datos. La transacción debe reservarse únicamente para escrituras atómicas locales rápidas.
4. **Preferir Upserts Atómicos sobre Verificaciones Previas (Check-Then-Insert)**: Buscar un registro y luego insertarlo es vulnerable a colisiones bajo concurrencia. Debe usarse siempre la operación atómica de upsert a nivel de motor de datos.
5. **Preservar el Orden con Claves de Partición Compuestas**: En streams de eventos, dirigir todas las transiciones de un recurso a la misma partición con `{tipoRecurso}_{recursoId}` garantiza entrega estrictamente ordenada (FIFO).
6. **Diferenciar la Ausencia de Datos del Cero Numérico**: En modelos y contratos, utilizar tipos puntero (`*int`, `*bool`). El valor `0` o `false` es una observación válida; `nil` o `null` representa que no hay datos.
7. **Usar Intervalos Semiabiertos $[start, end)$ en Líneas de Tiempo Continuas**: Para segmentación de tiempo continua, usar siempre desigualdad estricta (`<`) en el límite superior para evitar eliminar o contar dos veces el inicio del segmento siguiente.
8. **Mantener Simetría en Precondiciones de Entrada**: Un validador de entrada compartido nunca debe imponer precondiciones que solo aplican a un flujo particular, dejando sin atender flujos alternativos válidos.
9. **No Usar Cabeceras de Cache en Métodos HTTP Mutables**: Los proxies intermedios pueden descartar el cuerpo de peticiones `POST` o `PUT` si llevan cabeceras de cache. El cacheo debe reservarse para lecturas idempotentes.
10. **Preservar el Stack Trace en Manejadores de Pánicos Asíncronos**: Cualquier rutina de concurrencia que capture excepciones (`recover`) debe guardar y registrar el stack trace completo (`debug.Stack()`) para que el diagnóstico sea siempre posible.

---

## 🤖 Uso para Agentes Autónomos

Para conocer el protocolo y checklist de cómo los agentes deben consultar y aplicar estas enseñanzas, consulta [AGENT_INSTRUCTIONS.md](file:///c:/Users/juanc/Downloads/AISkills/atlas/AGENT_INSTRUCTIONS.md).

