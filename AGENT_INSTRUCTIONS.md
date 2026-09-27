# 🤖 Atlas: Protocolo y Verificación de Enseñanzas para Agentes

> **Guía para Agentes de IA (Forge, Subagentes y Asistentes de Código) sobre cómo consultar y aplicar las enseñanzas y buenas prácticas de Atlas.**

---

## 🎯 Cuándo Consultar Atlas

Como agente autónomo, debes consultar las guías de Atlas siempre que diseñes, implementes o revises código en las siguientes áreas:

1. **Diseño de Sistemas y Arquitectura**:
   - Concurrencia, pools de trabajadores o ajuste de runtimes $\rightarrow$ Consultar [`01-concurrency-and-runtime-traps.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/01-concurrency-and-runtime-traps.md).
   - Eventos, colas de mensajes o deduplicación distribuida $\rightarrow$ Consultar [`02-event-driven-streaming-and-idempotency.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/02-event-driven-streaming-and-idempotency.md) y [`07-state-machines-and-streaming-pipelines.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/07-state-machines-and-streaming-pipelines.md).
   - Operaciones de bases de datos, esquemas o transacciones $\rightarrow$ Consultar [`06-database-transactions-and-queries.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/06-database-transactions-and-queries.md).
   - Capas de cache y proxies intermedios $\rightarrow$ Consultar [`05-caching-layers-and-edge-proxies.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/05-caching-layers-and-edge-proxies.md).

2. **Implementación de Código y Refactorización**:
   - Contratos de API, DTOs y validaciones de entrada $\rightarrow$ Consultar [`04-api-contracts-and-validation.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/04-api-contracts-and-validation.md).
   - Manejo de fechas, zonas horarias y ventanas de tiempo $\rightarrow$ Consultar [`03-temporal-data-and-time-windows.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/03-temporal-data-and-time-windows.md).
   - Concurrencia en Go, rutinas asíncronas o bucles de reintento $\rightarrow$ Consultar [`01-concurrency-and-runtime-traps.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/01-concurrency-and-runtime-traps.md).

3. **Diagnóstico y Análisis Técnico**:
   - Investigación de errores intermitentes (500, 502, 504) o problemas de rendimiento $\rightarrow$ Consultar [`01-concurrency-and-runtime-traps.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/01-concurrency-and-runtime-traps.md) y [`08-observability-rca-and-triaging.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/08-observability-rca-and-triaging.md).
   - Análisis de mensajes duplicados o acumulaciones en colas $\rightarrow$ Consultar [`02-event-driven-streaming-and-idempotency.md`](file:///c:/Users/juanc/Downloads/AISkills/atlas/guides/02-event-driven-streaming-and-idempotency.md).

---

## 📋 Lista de Verificación para Agentes

Antes de dar por finalizado un plan de implementación o aprobar código, verifica que se cumplan estas buenas prácticas:

### 1. Concurrencia y Runtimes
- [ ] ¿Las operaciones en vuelo deduplicadas (`singleflight`) desacoplan el contexto de cancelación del llamante con `context.WithoutCancel(ctx)`?
- [ ] ¿El servicio contenedorizado en Go sincroniza los hilos del runtime con la cuota de CPU de cgroups (ej. `_ "go.uber.org/automaxprocs"`)?
- [ ] ¿Las esperas y reintentos son sensibles a la cancelación del contexto (`select { case <-time.After(): case <-ctx.Done(): }`), evitando bloqueos rígidos?
- [ ] ¿Las rutinas que capturan pánicos asíncronos guardan el stack trace completo (`debug.Stack()`)?
- [ ] ¿Se evitan bloqueos globales compartidos cuando las operaciones pertenecen a claves o recursos independientes?

### 2. Bases de Datos y Persistencia
- [ ] ¿Se excluyen totalmente las llamadas de red externas (HTTP, RPC) dentro de los bloques de transacción de base de datos?
- [ ] ¿Las creaciones de registros emplean operaciones atómicas de upsert (`ReplaceOne` con `upsert: true`) en lugar de verificación previa seguida de inserción?
- [ ] ¿Las consultas sobre colecciones con borrado lógico filtran explícitamente los registros inactivos (`{ deleted: { $ne: true } }`)?
- [ ] ¿Los identificadores de consulta coinciden en tipo con el formato nativo del motor de base de datos?

### 3. Mensajería y Streams
- [ ] ¿Las claves de partición en streams incluyen el identificador de el recurso (`{tipoRecurso}_{recursoId}`) para asegurar orden FIFO por recurso?
- [ ] ¿Se manejan fallos parciales por elemento dentro del lote, evitando reintentar todo el lote completo?
- [ ] ¿La idempotencia del consumidor se valida contra una clave compuesta (`IdMensaje + IdRecurso`)?
- [ ] ¿Las transiciones de estado simultáneas en el mismo instante se unifican en el emisor para evitar desorden en la partición?

### 4. Datos Temporales y Ventanas
- [ ] ¿Las consultas y limpiezas de segmentos temporales continuos emplean intervalos semiabiertos ($[start, end)$ con desigualdad estricta `<` en el límite superior)?
- [ ] ¿Las ordenaciones de series temporales incluyen una clave secundaria única de desempate para garantizar resultados deterministas?
- [ ] ¿Las comprobaciones de fechas de expiración son inclusivas de la fecha límite final?
- [ ] ¿Las ventanas rodantes calculan sobre diferencias reales de tiempo en lugar de índices estáticos de array?

### 5. Contratos de API y Capas Intermedias
- [ ] ¿Los campos numéricos o booleanos que legítimamente pueden ser cero o falso usan tipos puntero (`*int`, `*bool`) en structs validados?
- [ ] ¿El contrato de datos distingue con claridad entre ausencia de información (`null`) y una medición de valor cero (`0`)?
- [ ] ¿Las cabeceras de cache (`Cache-Control`) están ausentes de todas las peticiones HTTP mutables (`POST`, `PUT`, `DELETE`)?
- [ ] ¿Las validaciones de entrada compartidas permiten todos los caminos legítimos de identificación sin descartar flujos alternativos?

