# =====================================================================================
# LAB — BÚSQUEDAS “DE VERDAD” (OpenSearch Dev Tools Console)
# =====================================================================================
# REGLAS DEL LAB:
# - NO hay comentarios dentro de JSON (porque JSON no soporta comentarios).
# - Las explicaciones van SIEMPRE antes de cada request.
# - Ejecuta en orden.
# - Los DELETE pueden dar 404 si no existe: es normal.
#
# QUÉ VAS A TOCAR AQUÍ:
# 1) Template (settings + analysis + mapping pro)
# 2) Índice v1 + alias read/write (contrato)
# 3) Bulk realista (recetas con pasos, herramientas, ingredientes; object y nested)
# 4) Búsquedas serias:
#    - bool filter vs must/should (scoring)
#    - boosting, function_score, recency
#    - highlight + fragmentos
#    - aggregations (terms/range) + post_filter
#    - collapse (dedupe) + inner_hits
#    - object vs nested: falso positivo vs query correcta
#    - debugging: validate, explain, profile
# 5) Paginación seria: PIT + search_after
# 6) Runtime fields: enriquecer búsquedas sin reindexar
# =====================================================================================



# -------------------------------------------------------------------------------------
# 0) SALUD Y VISTAS RÁPIDAS
# - _cluster/health: estado general (green/yellow/red)
# - _cat/nodes: roles/heap/cpu (vista “humana”)
# - _cat/indices: tamaño/docs/health (vista “humana”)
# -------------------------------------------------------------------------------------

GET _cluster/health?pretty
GET _cat/nodes?v
GET _cat/indices?v



# -------------------------------------------------------------------------------------
# 0.1) LIMPIEZA IDEMPOTENTE
# - Borramos índices y plantilla si existían.
# - Si un DELETE da 404: ok (entorno limpio).
# -------------------------------------------------------------------------------------

DELETE recetas-search-v1
DELETE recetas-search-v2
DELETE _index_template/recetas_search_template



# -------------------------------------------------------------------------------------
# 0.2) LIMPIEZA DE ALIAS
# - Si existían alias colgando de índices antiguos, los quitamos.
# - Ojo: remove exige index+alias; si ese par no existe, da error.
#   En un lab “de consola” lo asumimos como parte del juego.
# -------------------------------------------------------------------------------------

POST _aliases
{
  "actions": [
    { "remove": { "index": "recetas_base_v3", "alias": "recetas_write" } },
    { "remove": { "index": "recetas_base_v3", "alias": "recetas_write" } }
  ]
}

POST _aliases
{
  "actions": [
    { "remove": { "index": "recetas_base_v3", "alias": "recetas_read" } },
    { "remove": { "index": "recetas_base_v3", "alias": "recetas_read" } }
  ]
}

GET _cat/aliases


# -------------------------------------------------------------------------------------
# 1) TEMPLATE: SETTINGS + ANALYSIS + MAPPING “PRO”
# IDEA:
# - Trabajar con índices versionados (recetas-search-v1, v2, …)
# - Crear una PLANTILLA aplicable por patrón (recetas-search-*)
#
# SETTINGS:
# - shards/replicas/refresh_interval
# - analysis: normalizer (keyword), analyzer (text), edge_ngram (autocomplete)
#
# MAPPING:
# - tenant_id/doc_type/id: keyword normalizado (case/acento-insensitive)
# - nombre/descripcion:
#     - text para full-text
#     - subcampo .raw keyword para aggs/sort
#     - subcampo .ac con edge_ngram para autocomplete
# - copy_to -> fulltext: buscador global (sin duplicar en _source)
# - herramientas: keyword multi-valor
# - pasos: NESTED (mantiene coherencia en queries por paso)
# - ingredientes_obj: OBJECT (aplana -> permite falsos positivos)
# - ingredientes_nst: NESTED (coherencia real)
# - tipos extra: geo_point e ip (para mostrar variedad real)
# -------------------------------------------------------------------------------------

PUT _index_template/recetas_search_template
{
  "index_patterns": ["recetas-search-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "refresh_interval": "1s",
      "index.mapping.total_fields.limit": 2000,
      "analysis": {
        "normalizer": {
          "kw_norm": {
            "type": "custom",
            "filter": ["lowercase", "asciifolding"]
          }
        },
        "filter": {
          "edge_ngram_filter": {
            "type": "edge_ngram",
            "min_gram": 2,
            "max_gram": 15
          }
        },
        "analyzer": {
          "spanish_text": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["lowercase", "asciifolding", "stop", "snowball"]
          },
          "autocomplete_edge": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["lowercase", "asciifolding", "edge_ngram_filter"]
          }
        }
      }
    },
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "tenant_id": { "type": "keyword", "normalizer": "kw_norm" },
        "doc_type": { "type": "keyword", "normalizer": "kw_norm" },
        "id": { "type": "keyword", "normalizer": "kw_norm" },

        "nombre": {
          "type": "text",
          "analyzer": "spanish_text",
          "fields": {
            "raw": { "type": "keyword", "normalizer": "kw_norm", "ignore_above": 256 },
            "ac":  { "type": "text", "analyzer": "autocomplete_edge", "search_analyzer": "spanish_text" }
          },
          "copy_to": ["fulltext"]
        },
        "descripcion": {
          "type": "text",
          "analyzer": "spanish_text",
          "fields": {
            "raw": { "type": "keyword", "normalizer": "kw_norm", "ignore_above": 256 }
          },
          "copy_to": ["fulltext"]
        },

        "fulltext": {
          "type": "text",
          "analyzer": "spanish_text",
          "norms": false,
          "index_options": "positions"
        },

        "fecha_creacion": { "type": "date" },
        "tiempo_total_min": { "type": "integer" },
        "calorias": { "type": "integer" },
        "precio": { "type": "scaled_float", "scaling_factor": 100 },
        "dificultad": { "type": "keyword", "normalizer": "kw_norm" },
        "apto_veggie": { "type": "boolean" },

        "tags": { "type": "keyword", "normalizer": "kw_norm", "ignore_above": 128 },
        "cocina": { "type": "keyword", "normalizer": "kw_norm" },
        "herramientas": { "type": "keyword", "normalizer": "kw_norm" },

        "pasos": {
          "type": "nested",
          "properties": {
            "n": { "type": "integer" },
            "accion": { "type": "keyword", "normalizer": "kw_norm" },
            "texto": { "type": "text", "analyzer": "spanish_text", "copy_to": ["fulltext"] },
            "tiempo_min": { "type": "integer" },
            "temp_c": { "type": "integer" },
            "herramienta": { "type": "keyword", "normalizer": "kw_norm" }
          }
        },

        "ingredientes_obj": {
          "type": "object",
          "properties": {
            "nombre": { "type": "keyword", "normalizer": "kw_norm" },
            "cantidad": { "type": "integer" },
            "unidad": { "type": "keyword", "normalizer": "kw_norm" }
          }
        },

        "ingredientes_nst": {
          "type": "nested",
          "properties": {
            "nombre": { "type": "keyword", "normalizer": "kw_norm" },
            "cantidad": { "type": "integer" },
            "unidad": { "type": "keyword", "normalizer": "kw_norm" }
          }
        },

        "origen_geo": { "type": "geo_point" },
        "ip_origen":  { "type": "ip" },

        "familia": { "type": "keyword", "normalizer": "kw_norm" }
      }
    }
  }
}



# -------------------------------------------------------------------------------------
# 2) CREAR ÍNDICE v1 Y ALIAS READ/WRITE
# - El índice recetas-search-v1 hereda settings+mappings de la plantilla por patrón.
# - Se crean alias:
#     recetas_write => escritura (solo 1 write index activo)
#     recetas_read  => lectura (puede apuntar a 1 o varios índices)
# -------------------------------------------------------------------------------------

PUT recetas-search-v1
{
  "aliases": {
    "recetas_read": {},
    "recetas_write": { "is_write_index": true }
  }
}

GET _cat/aliases/recetas_*?v
GET recetas-search-v1/_mapping?pretty



# -------------------------------------------------------------------------------------
# 3) INGESTA: BULK (NDJSON)
# - Dataset pequeño pero con variedad:
#   - 2 tenants: acme / globex
#   - familia: “croquetas” con variantes => útil para collapse/dedupe
#   - pasos nested con herramienta y temperaturas
#   - ingredientes duplicados en object y nested (para comparar queries)
# - Importante: NDJSON => líneas alternas (acción/doc).
# -------------------------------------------------------------------------------------

POST recetas_write/_bulk
{"index":{"_id":"r1"}}
{"tenant_id":"acme","doc_type":"receta","id":"r1","familia":"croquetas_jamon","nombre":"Croquetas de jamón","descripcion":"Croquetas cremosas de jamón ibérico y bechamel. Rebozado crujiente.","cocina":"es","tags":["fritura","tapa","clasico"],"dificultad":"media","apto_veggie":false,"tiempo_total_min":35,"calorias":420,"precio":3.50,"fecha_creacion":"2026-01-22T10:00:00Z","herramientas":["sarten","cazo","termometro"],"origen_geo":{"lat":40.4168,"lon":-3.7038},"ip_origen":"10.0.1.10","ingredientes_obj":[{"nombre":"jamon","cantidad":120,"unidad":"g"},{"nombre":"leche","cantidad":500,"unidad":"ml"},{"nombre":"harina","cantidad":60,"unidad":"g"},{"nombre":"mantequilla","cantidad":60,"unidad":"g"}],"ingredientes_nst":[{"nombre":"jamon","cantidad":120,"unidad":"g"},{"nombre":"leche","cantidad":500,"unidad":"ml"},{"nombre":"harina","cantidad":60,"unidad":"g"},{"nombre":"mantequilla","cantidad":60,"unidad":"g"}],"pasos":[{"n":1,"accion":"preparar","texto":"Hacer un roux con mantequilla y harina.","tiempo_min":5,"temp_c":120,"herramienta":"cazo"},{"n":2,"accion":"cocinar","texto":"Añadir leche y cocer la bechamel hasta espesar.","tiempo_min":12,"temp_c":95,"herramienta":"cazo"},{"n":3,"accion":"mezclar","texto":"Incorporar jamón picado y enfriar la masa.","tiempo_min":10,"temp_c":25,"herramienta":"bol"},{"n":4,"accion":"freir","texto":"Formar croquetas, rebozar y freír a 180C.","tiempo_min":8,"temp_c":180,"herramienta":"sarten"}]}
{"index":{"_id":"r2"}}
{"tenant_id":"acme","doc_type":"receta","id":"r2","familia":"croquetas_marisco","nombre":"Croquetas de marisco","descripcion":"Croquetas con gambas y caldo suave. Perfectas como entrante.","cocina":"es","tags":["fritura","marisco"],"dificultad":"media","apto_veggie":false,"tiempo_total_min":40,"calorias":460,"precio":4.20,"fecha_creacion":"2026-01-22T10:10:00Z","herramientas":["sarten","cazo"],"origen_geo":{"lat":36.7213,"lon":-4.4217},"ip_origen":"10.0.1.11","ingredientes_obj":[{"nombre":"gambas","cantidad":200,"unidad":"g"},{"nombre":"leche","cantidad":450,"unidad":"ml"},{"nombre":"harina","cantidad":70,"unidad":"g"}],"ingredientes_nst":[{"nombre":"gambas","cantidad":200,"unidad":"g"},{"nombre":"leche","cantidad":450,"unidad":"ml"},{"nombre":"harina","cantidad":70,"unidad":"g"}],"pasos":[{"n":1,"accion":"cocinar","texto":"Saltear gambas y reservar.","tiempo_min":4,"temp_c":160,"herramienta":"sarten"},{"n":2,"accion":"cocinar","texto":"Preparar bechamel y añadir marisco picado.","tiempo_min":15,"temp_c":95,"herramienta":"cazo"},{"n":3,"accion":"freir","texto":"Formar, rebozar y freír.","tiempo_min":8,"temp_c":180,"herramienta":"sarten"}]}
{"index":{"_id":"r3"}}
{"tenant_id":"globex","doc_type":"receta","id":"r3","familia":"papas_arrugadas","nombre":"Papas arrugadas con mojo","descripcion":"Papas pequeñas cocidas con sal. Mojo rojo picante.","cocina":"es","tags":["hervido","canarias","salsa"],"dificultad":"facil","apto_veggie":true,"tiempo_total_min":30,"calorias":280,"precio":2.10,"fecha_creacion":"2026-01-22T11:00:00Z","herramientas":["olla"],"origen_geo":{"lat":28.2916,"lon":-16.6291},"ip_origen":"10.0.2.10","ingredientes_obj":[{"nombre":"papas","cantidad":600,"unidad":"g"},{"nombre":"sal","cantidad":80,"unidad":"g"},{"nombre":"ajo","cantidad":3,"unidad":"ud"},{"nombre":"pimenton","cantidad":2,"unidad":"cda"}],"ingredientes_nst":[{"nombre":"papas","cantidad":600,"unidad":"g"},{"nombre":"sal","cantidad":80,"unidad":"g"},{"nombre":"ajo","cantidad":3,"unidad":"ud"},{"nombre":"pimenton","cantidad":2,"unidad":"cda"}],"pasos":[{"n":1,"accion":"cocer","texto":"Cocer papas con mucha sal hasta arrugar.","tiempo_min":22,"temp_c":100,"herramienta":"olla"},{"n":2,"accion":"triturar","texto":"Hacer mojo con ajo, pimentón y aceite.","tiempo_min":6,"temp_c":25,"herramienta":"mortero"}]}
{"index":{"_id":"r4"}}
{"tenant_id":"globex","doc_type":"receta","id":"r4","familia":"paella_marisco","nombre":"Paella de marisco","descripcion":"Arroz meloso con marisco y caldo. Sofrito tradicional.","cocina":"es","tags":["arroz","marisco"],"dificultad":"dificil","apto_veggie":false,"tiempo_total_min":70,"calorias":680,"precio":12.90,"fecha_creacion":"2026-01-22T11:30:00Z","herramientas":["paellera","cucharon"],"origen_geo":{"lat":39.4699,"lon":-0.3763},"ip_origen":"10.0.2.11","ingredientes_obj":[{"nombre":"arroz","cantidad":400,"unidad":"g"},{"nombre":"caldo","cantidad":1200,"unidad":"ml"},{"nombre":"mejillones","cantidad":300,"unidad":"g"},{"nombre":"gambas","cantidad":250,"unidad":"g"}],"ingredientes_nst":[{"nombre":"arroz","cantidad":400,"unidad":"g"},{"nombre":"caldo","cantidad":1200,"unidad":"ml"},{"nombre":"mejillones","cantidad":300,"unidad":"g"},{"nombre":"gambas","cantidad":250,"unidad":"g"}],"pasos":[{"n":1,"accion":"sofreir","texto":"Hacer sofrito y nacarar el arroz.","tiempo_min":10,"temp_c":140,"herramienta":"paellera"},{"n":2,"accion":"cocer","texto":"Añadir caldo y cocer sin remover.","tiempo_min":18,"temp_c":100,"herramienta":"paellera"},{"n":3,"accion":"reposar","texto":"Reposar 5 minutos antes de servir.","tiempo_min":5,"temp_c":25,"herramienta":"paellera"}]}
{"index":{"_id":"r5"}}
{"tenant_id":"acme","doc_type":"receta","id":"r5","familia":"croquetas_jamon","nombre":"Croquetas de jamón (versión ligera)","descripcion":"Menos grasa, horno en vez de fritura. Bechamel suave.","cocina":"es","tags":["tapa","horno"],"dificultad":"media","apto_veggie":false,"tiempo_total_min":45,"calorias":360,"precio":3.80,"fecha_creacion":"2026-01-23T09:00:00Z","herramientas":["horno","bandeja"],"origen_geo":{"lat":40.4168,"lon":-3.7038},"ip_origen":"10.0.1.12","ingredientes_obj":[{"nombre":"jamon","cantidad":100,"unidad":"g"},{"nombre":"leche","cantidad":500,"unidad":"ml"},{"nombre":"harina","cantidad":60,"unidad":"g"}],"ingredientes_nst":[{"nombre":"jamon","cantidad":100,"unidad":"g"},{"nombre":"leche","cantidad":500,"unidad":"ml"},{"nombre":"harina","cantidad":60,"unidad":"g"}],"pasos":[{"n":1,"accion":"preparar","texto":"Preparar masa y formar croquetas.","tiempo_min":20,"temp_c":25,"herramienta":"bol"},{"n":2,"accion":"hornear","texto":"Hornear a 200C hasta dorar.","tiempo_min":15,"temp_c":200,"herramienta":"horno"}]}

POST recetas_write/_refresh



# -------------------------------------------------------------------------------------
# 4) AUTOCOMPLETE “DE VERDAD”
# - Se consulta el subcampo nombre.ac (edge_ngram) para prefijos.
# - Esto es lo típico de un buscador estilo Netflix / ecommerce.
# - Ojo: aquí usamos match sobre nombre.ac (no fulltext).
# -------------------------------------------------------------------------------------

GET recetas_read/_search
{
  "size": 10,
  "_source": ["id","nombre","familia","tenant_id"],
  "query": {
    "match": {
      "nombre.ac": "croq"
    }
  }
}



# -------------------------------------------------------------------------------------
# 5) BÚSQUEDA “BUSCADOR GLOBAL” CON FILTROS SERIOS
# IDEA:
# - Filter: no puntúa, es cacheable y barato (tenant, tags, rangos).
# - Must: full-text con scoring (lo que define relevancia).
# - Should: boosting “si aparece X, sube”.
# - Además metemos:
#   - highlight para UX
#   - sort secundario por fecha (recency) si empatan
# -------------------------------------------------------------------------------------

GET recetas_read/_search
{
  "size": 10,
  "track_total_hits": true,
  "_source": ["id","nombre","descripcion","tags","tiempo_total_min","precio","fecha_creacion","tenant_id"],
  "query": {
    "bool": {
      "filter": [
        { "term": { "tenant_id": "acme" } },
        { "range": { "precio": { "lte": 4.00 } } }
      ],
      "must": [
        { "match": { "fulltext": "bechamel jamon" } }
      ],
      "should": [
        { "term": { "tags": "tapa" } }
      ],
      "minimum_should_match": 0
    }
  },
"highlight": {
  "pre_tags": ["<mark>"],
  "post_tags": ["</mark>"],
  "require_field_match": false,
  "fields": {
    "nombre": {
      "number_of_fragments": 0
    },
    "descripcion": {
      "fragment_size": 140,
      "number_of_fragments": 2
    }
  }
},
  "sort": [
    { "_score": "desc" },
    { "fecha_creacion": "desc" }
  ]
}



# -------------------------------------------------------------------------------------
# 6) BOOSTING MÁS AGRESIVO: FUNCTION_SCORE
# IDEA:
# - Mantienes el match fulltext, pero “premias”:
#   - recetas recientes (gauss decay sobre fecha)
#   - recetas más baratas (field_value_factor inverso)
# - Esto se usa muchísimo en ranking real (search relevance).
# -------------------------------------------------------------------------------------

GET recetas_read/_search
{
  "size": 10,
  "_source": ["id","nombre","precio","fecha_creacion","calorias","tags"],
  "query": {
    "function_score": {
      "query": {
        "match": { "fulltext": "croquetas jamon" }
      },
      "functions": [
        {
          "gauss": {
            "fecha_creacion": {
              "origin": "2026-01-24T00:00:00Z",
              "scale": "7d",
              "decay": 0.5
            }
          }
        },
        {
          "field_value_factor": {
            "field": "precio",
            "factor": 1.0,
            "modifier": "reciprocal",
            "missing": 1.0
          }
        }
      ],
      "score_mode": "multiply",
      "boost_mode": "multiply"
    }
  }
}



# -------------------------------------------------------------------------------------
# 7) AGGREGATIONS REALES + POST_FILTER
# IDEA:
# - Caso típico: UI tipo ecommerce.
# - Quieres:
#   - resultados filtrados por texto
#   - FACETAS (aggs) para mostrar counts por tag/dificultad
# - PROBLEMA:
#   - Si aplicas el filtro de la UI dentro de query/filter,
#     las aggs se ven “ya filtradas” (y te quedan facet counts pobres).
# SOLUCIÓN:
# - query: define el “universo” del buscador (texto + tenant, etc)
# - aggs: se calculan sobre ese universo
# - post_filter: aplica el filtro final SOLO a hits (no a aggs)
# -------------------------------------------------------------------------------------

GET recetas_read/_search
{
  "size": 10,
  "_source": ["id","nombre","tags","dificultad","precio"],
  "query": {
    "bool": {
      "filter": [
        { "term": { "tenant_id": "globex" } }
      ],
      "must": [
        { "match": { "fulltext": "marisco" } }
      ]
    }
  },
  "aggs": {
    "por_tag": {
      "terms": { "field": "tags", "size": 10 }
    },
    "por_dificultad": {
      "terms": { "field": "dificultad", "size": 10 }
    },
    "precio_ranges": {
      "range": {
        "field": "precio",
        "ranges": [
          { "to": 3.00 },
          { "from": 3.00, "to": 8.00 },
          { "from": 8.00 }
        ]
      }
    }
  },
  "post_filter": {
    "term": { "dificultad": "dificil" }
  }
}



# -------------------------------------------------------------------------------------
# 8) OBJECT vs NESTED — DEMOSTRACIÓN “BUG CONCEPTUAL”
# IDEA:
# - ingredientes_obj es object: se aplana (puede cruzar campos entre items)
# - Query: "nombre=jamon" y "cantidad=500" (500 es la leche en r1)
# - Con object puede dar match aunque no exista jamón con 500.
# -------------------------------------------------------------------------------------

GET recetas_read/_search
{
  "size": 10,
  "_source": ["id","nombre","ingredientes_obj"],
  "query": {
    "bool": {
      "must": [
        { "term": { "ingredientes_obj.nombre": "jamon" } },
        { "term": { "ingredientes_obj.cantidad": 500 } }
      ]
    }
  }
}



# -------------------------------------------------------------------------------------
# 9) NESTED — QUERY CORRECTA (coherente)
# IDEA:
# - Misma intención que arriba, pero garantizando que nombre y cantidad
#   pertenecen al MISMO elemento del array (nested).
# - Aquí NO debería devolver nada.
# -------------------------------------------------------------------------------------

GET recetas_read/_search
{
  "size": 10,
  "_source": ["id","nombre","ingredientes_nst"],
  "query": {
    "nested": {
      "path": "ingredientes_nst",
      "query": {
        "bool": {
          "must": [
            { "term": { "ingredientes_nst.nombre": "jamon" } },
            { "term": { "ingredientes_nst.cantidad": 500 } }
          ]
        }
      }
    }
  }
}



# -------------------------------------------------------------------------------------
# 10) NESTED EN “PASOS” — BUSCAR POR HERRAMIENTA Y TEMPERATURA EN EL MISMO PASO
# IDEA:
# - Quiero recetas donde exista un paso que:
#   - herramienta = sarten
#   - temp_c >= 175
# - Esto es un caso real de nested: condiciones coherentes dentro del paso.
# -------------------------------------------------------------------------------------

GET recetas_read/_search
{
  "size": 10,
  "_source": ["id","nombre","pasos"],
  "query": {
    "nested": {
      "path": "pasos",
      "query": {
        "bool": {
          "filter": [
            { "term": { "pasos.herramienta": "sarten" } },
            { "range": { "pasos.temp_c": { "gte": 175 } } }
          ]
        }
      },
      "inner_hits": {
        "size": 3
      }
    }
  }
}



# -------------------------------------------------------------------------------------
# 11) COLLAPSE — DEDUPLICAR “FAMILIAS” (variantes) + INNER_HITS
# IDEA:
# - “croquetas_jamon” tiene varias variantes (r1, r5)
# - UI típica: mostrar 1 resultado por familia, pero con “variantes” desplegables.
# - collapse reduce duplicados y inner_hits devuelve ejemplos por grupo.
# -------------------------------------------------------------------------------------

GET recetas_read/_search
{
  "size": 10,
  "_source": ["id","nombre","familia","precio","fecha_creacion"],
  "query": {
    "match": { "fulltext": "croquetas" }
  },
  "collapse": {
    "field": "familia",
    "inner_hits": {
      "name": "variantes",
      "size": 5,
      "sort": [
        { "fecha_creacion": "desc" }
      ],
      "_source": ["id","nombre","precio","fecha_creacion"]
    }
  },
  "sort": [
    { "_score": "desc" }
  ]
}



# -------------------------------------------------------------------------------------
# 12) DEBUGGING: VALIDATE + EXPLAIN
# IDEA:
# - validate: te dice si la query es válida y cómo se reescribe.
# - explain: explica por qué un documento matchea y cómo puntúa (por doc).
# -------------------------------------------------------------------------------------

POST recetas_read/_validate/query?explain=true
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "tenant_id": "acme" } }
      ],
      "must": [
        { "match": { "fulltext": "bechamel jamon" } }
      ]
    }
  }
}

GET recetas_read/_explain/r1
{
  "query": {
    "match": { "fulltext": "bechamel jamon" }
  }
}



# -------------------------------------------------------------------------------------
# 13) DEBUGGING: PROFILE (costes)
# IDEA:
# - profile te muestra en qué se gasta el tiempo (query y collectors).
# - Útil para enseñar “por qué esta query es cara”.
# -------------------------------------------------------------------------------------

GET recetas_read/_search
{
  "profile": true,
  "size": 10,
  "query": {
    "bool": {
      "filter": [
        { "term": { "tenant_id": "acme" } }
      ],
      "must": [
        { "match": { "fulltext": "croquetas jamon" } }
      ]
    }
  }
}



# -------------------------------------------------------------------------------------
# 14) PAGINACIÓN SERIA: PIT + SEARCH_AFTER
# IDEA:
# - from/size profundo es caro y tiene límites.
# - PIT (Point-in-Time) congela una vista consistente del índice para paginar.
# - search_after requiere un sort estable (por ejemplo fecha + _id).
#
# PASOS:
# A) Abrir PIT
# B) Buscar primera página (sin search_after)
# C) Repetir con search_after usando el sort del último hit
# D) Cerrar PIT
# -------------------------------------------------------------------------------------

POST recetas_read/_search/point_in_time?keep_alive=2m

GET recetas_read/_search
{
  "size": 2,
  "pit": { "id": "PON_AQUI_EL_PIT_ID_DE_LA_RESPUESTA", "keep_alive": "2m" },
  "sort": [
    { "fecha_creacion": "desc" },
    { "_id": "asc" }
  ],
  "_source": ["id","nombre","fecha_creacion"],
  "query": {
    "match_all": {}
  }
}

# Página 1 (usa PIT)
GET /_search
{
  "size": 2,
  "query": {
    "bool": {
      "filter": [
        { "term": { "tenant_id": "acme" } }
      ],
      "must": [
        { "match": { "fulltext": "croquetas jamon" } }
      ]
    }
  },
  "pit": {
    "id": "48a5QQMRcmVjZXRhcy1zZWFyY2gtdjEWTjI2WDNJNkRTOVdnbGJiUlZJcmdQZwIWTHE1bUVUM1JRNjZPSlEyR1BLRjhkdwAAAAAAAAAAMBZCU0VNYkRkcVRiLVJiSnh5aU5Sc093EXJlY2V0YXMtc2VhcmNoLXYxFk4yNlgzSTZEUzlXZ2xiYlJWSXJnUGcBFk4yNmZVVkp5VF95TXBFb3kzNWtXdlEAAAAAAAAAADQWMkhHY2ZfZ1ZSam0wUERrQlBCX2E5dxFyZWNldGFzLXNlYXJjaC12MRZOMjZYM0k2RFM5V2dsYmJSVklyZ1BnABZOMjZmVVZKeVRfeU1wRW95MzVrV3ZRAAAAAAAAAAA1FjJIR2NmX2dWUmptMFBEa0JQQl9hOXcBFk4yNlgzSTZEUzlXZ2xiYlJWSXJnUGcAAA==",
    "keep_alive": "2m"
  },
  "sort": [
    { "_score": "desc" },
    { "fecha_creacion": "desc" }
  ]
}

# Página 2 (search_after con el sort del último hit de la página anterior)
GET /_search
{
  "size": 2,
  "query": {
    "bool": {
      "filter": [
        { "term": { "tenant_id": "acme" } }
      ],
      "must": [
        { "match": { "fulltext": "croquetas jamon" } }
      ]
    }
  },
  "pit": {
    "id": "48a5QQMRcmVjZXRhcy1zZWFyY2gtdjEWTjI2WDNJNkRTOVdnbGJiUlZJcmdQZwIWTHE1bUVUM1JRNjZPSlEyR1BLRjhkdwAAAAAAAAAAMBZCU0VNYkRkcVRiLVJiSnh5aU5Sc093EXJlY2V0YXMtc2VhcmNoLXYxFk4yNlgzSTZEUzlXZ2xiYlJWSXJnUGcBFk4yNmZVVkp5VF95TXBFb3kzNWtXdlEAAAAAAAAAADQWMkhHY2ZfZ1ZSam0wUERrQlBCX2E5dxFyZWNldGFzLXNlYXJjaC12MRZOMjZYM0k2RFM5V2dsYmJSVklyZ1BnABZOMjZmVVZKeVRfeU1wRW95MzVrV3ZRAAAAAAAAAAA1FjJIR2NmX2dWUmptMFBEa0JQQl9hOXcBFk4yNlgzSTZEUzlXZ2xiYlJWSXJnUGcAAA==",
    "keep_alive": "2m"
  },
  "sort": [
    { "_score": "desc" },
    { "fecha_creacion": "desc" }
  ],
  "search_after": [0.46080106, 1769158800000]
}


DELETE _search/point_in_time
{
  "pit_id": "48a5QQMRcmVjZXRhcy1zZWFyY2gtdjEWTjI2WDNJNkRTOVdnbGJiUlZJcmdQZwIWTHE1bUVUM1JRNjZPSlEyR1BLRjhkdwAAAAAAAAAAMBZCU0VNYkRkcVRiLVJiSnh5aU5Sc093EXJlY2V0YXMtc2VhcmNoLXYxFk4yNlgzSTZEUzlXZ2xiYlJWSXJnUGcBFk4yNmZVVkp5VF95TXBFb3kzNWtXdlEAAAAAAAAAADQWMkhHY2ZfZ1ZSam0wUERrQlBCX2E5dxFyZWNldGFzLXNlYXJjaC12MRZOMjZYM0k2RFM5V2dsYmJSVklyZ1BnABZOMjZmVVZKeVRfeU1wRW95MzVrV3ZRAAAAAAAAAAA1FjJIR2NmX2dWUmptMFBEa0JQQl9hOXcBFk4yNlgzSTZEUzlXZ2xiYlJWSXJnUGcAAA=="
}

