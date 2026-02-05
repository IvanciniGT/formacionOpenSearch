# =====================================================================================
# LAB — MONITORIZACIÓN “DE VERDAD” (OpenSearch 3.3.x, Docker) — Receta Dev Tools
# =====================================================================================
# Objetivo:
#   - Tener un “panel de mando” técnico (APIs) para saber si el cluster está sano,
#     si el rendimiento se está degradando, y si Vector/ML están consumiendo recursos.
#   - Diagnóstico rápido: CPU/heap/GC, shards, merges/refresh, cache, breakers, threads,
#     y en tu caso: k-NN + ML Commons.
#
# Ideas clave:
#   - Empieza SIEMPRE por salud (cluster + nodes + shards).
#   - Luego memoria (heap + breakers) y presión de GC.
#   - Después “trabajo interno”: thread pools, merges, refresh, flush.
#   - En vector/ML: modelos, tareas, latencia y stats de k-NN.
# =====================================================================================


# =====================================================================================
# 0) Salud y topología (triage en 10s)
# =====================================================================================

GET _cluster/health?pretty
GET _cluster/state/metadata?filter_path=metadata.cluster_uuid,metadata.cluster_uuid_committed,cluster_name
GET _cat/nodes?v
GET _cat/indices?v&s=health,index
GET _cat/shards?v&s=index,shard,prirep,state
GET _cat/allocation?v


# =====================================================================================
# 1) Recursos del nodo: CPU, heap, RAM, FS, roles (lo básico)
# =====================================================================================

GET _nodes/stats?filter_path=nodes.*.name,nodes.*.roles,nodes.*.jvm.mem.heap_used_percent,nodes.*.jvm.mem.heap_used_in_bytes,nodes.*.jvm.mem.heap_max_in_bytes,nodes.*.os.cpu.percent,nodes.*.process.cpu.percent,nodes.*.fs.total.total_in_bytes,nodes.*.fs.total.available_in_bytes
GET _cat/nodes?v&h=name,ip,roles,cpu,heap.percent,heap.max,ram.percent,ram.max,node.role


# =====================================================================================
# 2) JVM/GC: ¿estás muriendo por GC?
# =====================================================================================
# Señales:
#   - heap_used_percent muy alto sostenido
#   - gc.collection_time_in_millis subiendo “demasiado”
#   - pauses frecuentes => latencia en queries/indexing

GET _nodes/stats/jvm?pretty
GET _nodes/stats/jvm?filter_path=nodes.*.name,nodes.*.jvm.mem.heap_used_percent,nodes.*.jvm.gc.collectors.*.collection_count,nodes.*.jvm.gc.collectors.*.collection_time_in_millis


# =====================================================================================
# 3) Circuit breakers: ¿te estás quedando sin memoria “operacional”?
# =====================================================================================
# Útil cuando ves errores 429/503/500 por OOM o breaker tripping.

GET _nodes/stats/breaker?pretty
GET _nodes/stats/breaker?filter_path=nodes.*.name,nodes.*.breakers.*.estimated_size_in_bytes,nodes.*.breakers.*.limit_size_in_bytes,nodes.*.breakers.*.tripped


# =====================================================================================
# 4) Thread pools: ¿colas reventadas? (index/search/write/management)
# =====================================================================================
# Señales:
#   - queue creciendo + rejected creciendo => saturación
#   - mira especialmente: search, write, bulk, refresh, management

GET _cat/thread_pool?v
GET _nodes/stats/thread_pool?pretty
GET _nodes/stats/thread_pool?filter_path=nodes.*.name,nodes.*.thread_pool.search.*,nodes.*.thread_pool.write.*,nodes.*.thread_pool.bulk.*,nodes.*.thread_pool.management.*


# =====================================================================================
# 5) Índices: merges/refresh/flush, segmentación, translog
# =====================================================================================
# Señales:
#   - merges intensos => IO alto, latencia
#   - refresh muy frecuente => coste; translog grande => riesgo en crash/recovery

GET _nodes/stats/indices?pretty
GET _nodes/stats/indices?filter_path=nodes.*.name,nodes.*.indices.refresh.total,nodes.*.indices.refresh.total_time_in_millis,nodes.*.indices.merges.total,nodes.*.indices.merges.total_time_in_millis,nodes.*.indices.flush.total,nodes.*.indices.flush.total_time_in_millis,nodes.*.indices.translog.operations,nodes.*.indices.translog.size_in_bytes


# =====================================================================================
# 6) Búsqueda: latencias, query cache, request cache, fielddata (ojo fielddata!)
# =====================================================================================
# Señales:
#   - query_cache hit rate pobre => cache inútil o queries demasiado variadas
#   - fielddata alto => mapping / aggregations mal planteadas (keyword vs text)

GET _nodes/stats/indices/search?pretty
GET _nodes/stats/indices/query_cache?pretty
GET _nodes/stats/indices/request_cache?pretty
GET _nodes/stats/indices/fielddata?pretty

GET _nodes/stats/indices/search?filter_path=nodes.*.name,nodes.*.indices.search.query_total,nodes.*.indices.search.query_time_in_millis,nodes.*.indices.search.fetch_total,nodes.*.indices.search.fetch_time_in_millis
GET _nodes/stats/indices/query_cache?filter_path=nodes.*.name,nodes.*.indices.query_cache.hit_count,nodes.*.indices.query_cache.miss_count,nodes.*.indices.query_cache.evictions,nodes.*.indices.query_cache.memory_size_in_bytes
GET _nodes/stats/indices/request_cache?filter_path=nodes.*.name,nodes.*.indices.request_cache.hit_count,nodes.*.indices.request_cache.miss_count,nodes.*.indices.request_cache.evictions,nodes.*.indices.request_cache.memory_size_in_bytes


# =====================================================================================
# 7) Cluster tasks: ¿hay tareas bloqueando el cluster?
# =====================================================================================
# Señales:
#   - pending tasks creciendo => cluster state updates lentos
#   - tasks largas => algo atascado (snapshots, reindex, etc.)

GET _cluster/pending_tasks?pretty
GET _tasks?pretty


# =====================================================================================
# 8) Logs/Hot threads: “¿qué demonios está haciendo el nodo ahora mismo?”
# =====================================================================================
# Hot threads te da pistas inmediatas de CPU alta o bloqueos.

GET _nodes/hot_threads?pretty


# =====================================================================================
# 9) KNN / Vector Search: stats del plugin (tu caso)
# =====================================================================================
# Lo que miras:
#   - knn_query_requests (si se usa)
#   - hit_count/miss_count
#   - indices_in_cache / eviction_count / cache_capacity_reached
#   - graph_* (HNSW) si hay actividad
#   - circuit_breaker_triggered (si salta por vectores)

GET /_plugins/_knn/stats


# =====================================================================================
# 10) ML Commons: modelos, tareas, uso y estado
# =====================================================================================
# Lo que miras:
#   - model_state: REGISTERING/REGISTERED/DEPLOYED
#   - tareas de inferencia/entrenamiento (si aplica)
#   - errores y latencias (cuando existan)

GET /_plugins/_ml/models/_search
{
  "query": { "match_all": {} },
  "size": 50
}

# Tu modelo (si quieres, fija el ID)
GET /_plugins/_ml/models/OwrILpwB7nQkEVYFlFqW

# Tareas ML (inference, deployment, etc.)
GET /_plugins/_ml/tasks/_search
{
  "query": { "match_all": {} },
  "size": 50
}


# =====================================================================================
# 11) Monitorización “por índice” (tu demo)
# =====================================================================================
# Estado del índice, shards y sizing rápido

GET /nlp-demo/_stats?pretty
GET /nlp-demo/_stats?filter_path=_all.primaries.docs.count,_all.primaries.store.size_in_bytes,_all.total.store.size_in_bytes,_all.primaries.indexing.index_total,_all.primaries.search.query_total,_all.primaries.search.query_time_in_millis

GET _cat/shards/nlp-demo?v
GET _cat/segments/nlp-demo?v
GET /nlp-demo/_settings
GET /nlp-demo/_mapping


# =====================================================================================
# 12) Alertas rápidas “manuales” (reglas mentales)
# =====================================================================================
# Si ves:
#   - heap_used_percent > 85% sostenido + GC subiendo => sube heap / baja carga / optimiza.
#   - breakers tripped > 0 => memory pressure real (muchas aggs/heavy queries/vectores).
#   - thread_pool.search.rejected creciendo => saturación en búsquedas (scale/tune).
#   - merges muy altos => indexing agresivo / refresh demasiado corto / IO limitado.
#   - knn cache evictions + capacity reached => ajusta cache / reduce dimensión / más RAM.
#
# (Esto no ejecuta nada; es guía embebida para el lab.)
# =====================================================================================
