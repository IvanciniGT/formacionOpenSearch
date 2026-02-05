# =====================================================================================
# LAB — SINÓNIMOS CON FICHERO (OpenSearch 3) — Recetas
# =====================================================================================
# Objetivo:
#   - Usar sinónimos en fichero (synonyms_path) para búsquedas reales.
#   - Aplicarlos SOLO en búsqueda (search-time), no en indexación.
#   - Recargar sinónimos “en caliente” sin reindexar (refresh_search_analyzers).
#
# Ideas clave:
#   - synonym_graph soporta multi-palabra y es lo recomendado para queries. (token filter)
#   - synonyms_path: ruta a fichero (absoluta o relativa a /usr/share/opensearch/config).
#   - updateable:true: permite recargar sinónimos en runtime (si no, no hay hot-reload).
#   - refresh_search_analyzers: aplica cambios del fichero a los analyzers de búsqueda.
#     (Si tu build no soporta esto, alternativa: close/open index o recreación).
# =====================================================================================


# --- 0) Comprobación rápida del cluster (opcional, pero útil) ---
GET _cluster/health?pretty
GET _cat/nodes?v
GET _cat/indices?v


# --- 1) Limpieza idempotente ---
# (Si no existe -> 404 -> ok)
DELETE recetas-syn-v1


# =====================================================================================
# 2) Crear índice “de búsqueda” (recetas-syn-v1)
# =====================================================================================
# Diseño:
#   - index-time analyzer: “es_index” (SIN sinónimos) => índice limpio y estable
#   - search-time analyzer: “es_search” (CON sinónimos desde fichero) => flexible
#
# Campos:
#   - nombre, descripcion: text con analyzer + search_analyzer
#   - fulltext: campo agregado con copy_to (para búsquedas tipo “caja única”)
#   - ingredientes_detalle: nested (para queries coherentes por ingrediente)
#   - pasos: object (a propósito; para enseñar que NO es nested)
#   - tags: keyword (aggs/sort/filters)
#   - herramientas: keyword multi-valor
#   - tiempo_total_min: integer, precio: scaled_float
#   - fecha_creacion: date
# =====================================================================================

PUT recetas-syn-v1
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 1,
    "refresh_interval": "1s",
    "analysis": {
      "filter": {
        "recetas_syns": {
          "type": "synonym_graph",
          "synonyms_path": "synonyms_recetas_es.txt",
          "updateable": true
        }
      },
      "analyzer": {
        "es_index": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding"]
        },
        "es_search": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "recetas_syns"]
        }
      }
    }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "tenant_id": { "type": "keyword" },
      "id": { "type": "keyword" },

      "nombre": {
        "type": "text",
        "analyzer": "es_index",
        "search_analyzer": "es_search",
        "copy_to": ["fulltext"],
        "fields": {
          "raw": { "type": "keyword", "ignore_above": 256 }
        }
      },

      "descripcion": {
        "type": "text",
        "analyzer": "es_index",
        "search_analyzer": "es_search",
        "copy_to": ["fulltext"]
      },

      "fulltext": {
        "type": "text",
        "analyzer": "es_index",
        "search_analyzer": "es_search",
        "norms": false,
        "index_options": "positions"
      },

      "tags": { "type": "keyword" },
      "herramientas": { "type": "keyword" },

      "tiempo_total_min": { "type": "integer" },
      "precio": { "type": "scaled_float", "scaling_factor": 100 },
      "fecha_creacion": { "type": "date" },

      "ingredientes_detalle": {
        "type": "nested",
        "properties": {
          "nombre": { "type": "keyword" },
          "cantidad": { "type": "scaled_float", "scaling_factor": 100 },
          "unidad": { "type": "keyword" },
          "marca":  { "type": "keyword" }
        }
      },

      "pasos": {
        "type": "object",
        "properties": {
          "n": { "type": "integer" },
          "texto": { "type": "text", "analyzer": "es_index", "search_analyzer": "es_search" },
          "tiempo_min": { "type": "integer" }
        }
      }
    }
  }
}


# =====================================================================================
# 3) Ingest de datos (bulk NDJSON)
# =====================================================================================
# Notas:
#  - “bacon” NO aparece en documentos (solo “jamón”), para demostrar sinónimos.
#  - “papa(s)” aparece en unos docs y “patata(s)” en otros.
#  - ingredientes_detalle es nested: podremos pedir “marca=X y nombre=jamon” coherente.
# =====================================================================================

POST recetas-syn-v1/_bulk
{"index":{"_id":"r1"}}
{"tenant_id":"acme","id":"r1","nombre":"Croquetas de jamón","descripcion":"Croquetas cremosas de jamón ibérico y bechamel. Rebozado crujiente.","tags":["fritura","tapa","clasico"],"herramientas":["sarten","cazo"],"tiempo_total_min":35,"precio":3.50,"fecha_creacion":"2026-01-22T10:00:00Z","ingredientes_detalle":[{"nombre":"jamon","cantidad":1.50,"unidad":"kg","marca":"5J"},{"nombre":"leche","cantidad":1.00,"unidad":"l","marca":"Pascual"}],"pasos":[{"n":1,"texto":"Preparar bechamel.","tiempo_min":15},{"n":2,"texto":"Mezclar con jamón, enfriar y formar.","tiempo_min":10},{"n":3,"texto":"Freír hasta dorar.","tiempo_min":10}]}
{"index":{"_id":"r2"}}
{"tenant_id":"acme","id":"r2","nombre":"Papas arrugadas con mojo","descripcion":"Papas con sal y mojo picón. Sabor canario.","tags":["hervido","canario"],"herramientas":["olla"],"tiempo_total_min":30,"precio":2.10,"fecha_creacion":"2026-01-22T11:00:00Z","ingredientes_detalle":[{"nombre":"papa","cantidad":1.00,"unidad":"kg","marca":"local"},{"nombre":"sal","cantidad":0.10,"unidad":"kg","marca":"marina"}],"pasos":[{"n":1,"texto":"Cocer papas con sal.","tiempo_min":25},{"n":2,"texto":"Servir con mojo.","tiempo_min":5}]}
{"index":{"_id":"r3"}}
{"tenant_id":"globex","id":"r3","nombre":"Tortilla de patatas","descripcion":"Tortilla jugosa, patata pochada y huevo.","tags":["tortilla","clasico"],"herramientas":["sarten"],"tiempo_total_min":40,"precio":4.00,"fecha_creacion":"2026-01-22T12:00:00Z","ingredientes_detalle":[{"nombre":"patata","cantidad":0.80,"unidad":"kg","marca":"agria"},{"nombre":"huevo","cantidad":6.00,"unidad":"ud","marca":"campero"}],"pasos":[{"n":1,"texto":"Pochado de patata.","tiempo_min":20},{"n":2,"texto":"Cuajar con huevo.","tiempo_min":10},{"n":3,"texto":"Reposar y servir.","tiempo_min":10}]}
{"index":{"_id":"r4"}}
{"tenant_id":"globex","id":"r4","nombre":"Caldo de marisco (fumet)","descripcion":"Base para arroces: fumet concentrado de marisco.","tags":["caldo","marisco"],"herramientas":["olla"],"tiempo_total_min":60,"precio":6.50,"fecha_creacion":"2026-01-22T13:00:00Z","ingredientes_detalle":[{"nombre":"marisco","cantidad":1.20,"unidad":"kg","marca":"costa"},{"nombre":"agua","cantidad":2.00,"unidad":"l","marca":""}],"pasos":[{"n":1,"texto":"Tostar carcasas y aromáticos.","tiempo_min":10},{"n":2,"texto":"Cocer y espumar.","tiempo_min":40},{"n":3,"texto":"Colar y reducir.","tiempo_min":10}]}

POST recetas-syn-v1/_refresh


# =====================================================================================
# 4) “Microscopio”: ANALYZE (ver tokens)
# =====================================================================================
# Aquí compruebas que:
#  - es_index NO mete sinónimos
#  - es_search SÍ mete sinónimos (desde el fichero)
# =====================================================================================

POST recetas-syn-v1/_analyze
{
  "analyzer": "es_index",
  "text": "bacon patatas"
}

POST recetas-syn-v1/_analyze
{
  "analyzer": "es_search",
  "text": "bacon patatas"
}


# =====================================================================================
# 5) Búsquedas reales (ranking + filtros + highlight)
# =====================================================================================


# --- 5.1) Query “caja única”: busco bacon y debería devolver croquetas de jamón (sin que exista bacon en docs) ---
# Highlight:
#  - En OpenSearch el highlight marca el texto con tags (por defecto <em>...</em>)
#  - Tú quieres “marcas”: aquí fijamos explícitamente pre_tags/post_tags.
GET recetas-syn-v1/_search
{
  "size": 10,
  "_source": ["id","tenant_id","nombre","precio","tags","fecha_creacion"],
  "query": {
    "bool": {
      "must": [
        { "match": { "fulltext": { "query": "bacon", "operator": "and" } } }
      ]
    }
  },
  "highlight": {
    "pre_tags": ["<mark>"],
    "post_tags": ["</mark>"],
    "fields": {
      "nombre": {},
      "descripcion": {}
    }
  }
}


# --- 5.2) Query con filtros “de negocio”: tenant + precio máximo + tags ---
#   - must: relevance (match)
#   - filter: exacto, cacheable (term/range)
GET recetas-syn-v1/_search
{
  "size": 10,
  "_source": ["id","tenant_id","nombre","precio","tags"],
  "query": {
    "bool": {
      "must": [
        { "match": { "fulltext": { "query": "patatas", "operator": "and" } } }
      ],
      "filter": [
        { "term":  { "tenant_id": "globex" } },
        { "range": { "precio": { "lte": 5.00 } } }
      ]
    }
  },
  "sort": [
    { "_score": "desc" },
    { "precio": "asc" }
  ]
}


# --- 5.3) “Nested de verdad”: quiero recetas con ingrediente=jamon Y marca=5J (coherente) ---
# Esto solo es fiable si ingredientes_detalle es nested.
GET recetas-syn-v1/_search
{
  "_source": ["id","nombre","ingredientes_detalle"],
  "query": {
    "nested": {
      "path": "ingredientes_detalle",
      "query": {
        "bool": {
          "must": [
            { "term": { "ingredientes_detalle.nombre": "jamon" } },
            { "term": { "ingredientes_detalle.marca": "5J" } }
          ]
        }
      }
    }
  }
}


# --- 5.4) Agregación útil: top tags ---
GET recetas-syn-v1/_search
{
  "size": 0,
  "aggs": {
    "top_tags": { "terms": { "field": "tags", "size": 10 } }
  }
}

# Edita el fichero opensearch-config/synonyms_recetas_es.txt y añade por ejemplo:
# croqueta, croquetas, kroketa, kroketas

# Recarga analyzers de búsqueda para este índice (o alias/wildcard)
POST /_plugins/_refresh_search_analyzers/recetas-syn-v1
