```http
# =====================================================================================
# LAB — TODO EN UNO (Dev Tools Console) — OpenSearch (Dashboards)
# =====================================================================================
# Idea general:
# - Aquí se toca TODO: settings/mappings/analysis + ingest + queries + alias + rollover + multi-tenant + routing.
# - Ejecuta en orden. Los DELETE/alias-remove pueden dar 404 si no existe: es normal.
#
# Nota “fork reality check”:
# - OpenSearch NO tiene el tipo "flattened" (de Elasticsearch). El equivalente práctico aquí es "flat_object".
#   Ojo con la diferencia: flat_object no se usa igual que flattened (ver comentarios en el bulk del doc 3).
# =====================================================================================


# --- 0.1 Salud / nodos / indices ---
# health:
#   green  = todo asignado (primarios + réplicas)
#   yellow = primarios ok, réplicas no asignadas (sin HA completa)
#   red    = falta algún primario (hay datos inaccesibles)
GET _cluster/health?pretty

# nodes: roles, heap, cpu, etc. (cat = “humano”)
GET _cat/nodes?v

# indices: tamaño, docs, health… (otra vista rápida)
GET _cat/indices?v


# --- 0.2 Limpieza (idempotente) ---
# Si no existe -> 404 -> ok.
DELETE recetas_base_v2
DELETE recetas_base_v3
DELETE recetas_object_v1
DELETE recetas_nested_v1
DELETE recetas_suggest_v1
DELETE recetas_shared_v1
DELETE acme-recetas_v1
DELETE globex-recetas_v1
DELETE logs-000001
DELETE logs-000002
DELETE _index_template/logs_template


# --- 0.3 Limpieza alias (solo si existen) ---
# Esto deja el entorno “sin contratos” para reconstruirlos luego.
# Nota: remove requiere especificar index + alias; si ese par no existe -> error -> se ignora.
POST _aliases
{
  "actions": [
    { "remove": { "index": "recetas_base_v2", "alias": "recetas_read" } },
    { "remove": { "index": "recetas_base_v2", "alias": "recetas_write" } },
    { "remove": { "index": "recetas_base_v3", "alias": "recetas_read" } },
    { "remove": { "index": "recetas_base_v3", "alias": "recetas_write" } },

    { "remove": { "index": "logs-000001", "alias": "logs_read" } },
    { "remove": { "index": "logs-000001", "alias": "logs_write" } },
    { "remove": { "index": "logs-000002", "alias": "logs_read" } },
    { "remove": { "index": "logs-000002", "alias": "logs_write" } }
  ]
}



# =====================================================================================
# 1) ÍNDICE “PRO” (recetas_base_v2)
# =====================================================================================
# Aquí está el “triángulo de siempre”:
#   - settings: cómo se comporta el índice (shards, replicas, refresh, analysis…)
#   - mappings: tipos de datos (text/keyword/int/date…), multifields, copy_to, dynamic templates…
#   - analysis (dentro de settings): tokenizers + filters + analyzers + normalizers
#
# Ideas clave:
#  - text: se analiza -> sirve para full-text (match, scoring BM25…)
#  - keyword: exacto -> sirve para term, aggs, sort (doc_values)
#  - normalizer (keyword): como analyzer pero para keyword (no tokeniza, solo normaliza)
#  - copy_to: “campo de búsqueda global” sin duplicar en _source
#  - dynamic_templates: reglas automáticas cuando entran campos nuevos
#  - payload_original enabled:false: guardo el JSON bruto pero NO creo mappings ni indexo nada (anti mapping-explosion)
# =====================================================================================

PUT recetas_base_v2
{
  "settings": {
    # Primarios: NO se cambian “a voluntad” (solo split/shrink/reindex).
    "number_of_shards": 3,

    # Réplicas: sí se cambian en caliente.
    "number_of_replicas": 1,

    # refresh_interval: cada cuánto “se ve” lo indexado en búsqueda.
    # (más alto => mejor ingest, peor near-real-time)
    "refresh_interval": "1s",

    # límite de campos de mapping: evita reventar por “mapping explosion”.
    "index.mapping.total_fields.limit": 2000,

    "analysis": {
      # ----------------------------------------
      # NORMALIZER (keyword)
      # ----------------------------------------
      # kw_norm:
      #  - lowercase: ACME == acme
      #  - asciifolding: "ACMÉ" => "acme" (esto es CLAVE para term queries en keyword)
      "normalizer": {
        "kw_norm": { "type": "custom", "filter": ["lowercase","asciifolding"] }
      },

      # ----------------------------------------
      # TOKEN FILTERS
      # ----------------------------------------
      "filter": {
        # Autocomplete: edge_ngram genera prefijos:
        #   croquetas -> cr, cro, croq, croqu, ...
        "edge_ngram_filter": { "type": "edge_ngram", "min_gram": 2, "max_gram": 15 },

        # Sinónimos (search-time): con synonym_graph para búsquedas (queries) compuestas.
        # Aquí se mete a propósito en el analyzer de BÚSQUEDA (search_with_synonyms),
        # para no “ensuciar” el índice si mañana cambias sinónimos.
        "es_synonyms": {
          "type": "synonym_graph",
          "synonyms": ["jamon, jamón, bacon","papas, patatas","marisco, mariscos"]
        }
      },

      # ----------------------------------------
      # ANALYZERS (text)
      # ----------------------------------------
      "analyzer": {
        # Analyzer “base” de indexación
        # - standard tokenizer: separa palabras (incluye números)
        # - lowercase + asciifolding: uniformidad (jamón == jamon)
        # - stop: stopwords (por defecto lista del idioma configurado por OS; aquí usamos stop genérico)
        # - snowball: stemming (reduce a raíz) -> útil para “estudiar/estudian/estudiante…”
        "spanish_text": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase","asciifolding","stop","snowball"]
        },

        # Autocomplete: edge_ngram SOLO para el campo .ac
        # Nota: esto sube el tamaño del índice (más tokens). Se compensa con UX.
        "autocomplete_edge": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase","asciifolding","edge_ngram_filter"]
        },

        # Analyzer de búsqueda (search_analyzer):
        # Igual que el base, pero con synonyms.
        # “Busco bacon” => también matchea jamón/jamon.
        "search_with_synonyms": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase","asciifolding","es_synonyms","stop","snowball"]
        }
      }
    }
  },

  "mappings": {
    # dynamic=true: permite campos nuevos (ojo en producción; aquí nos interesa para enseñar reglas)
    "dynamic": true,

    "dynamic_templates": [
      # Reglas automáticas:
      # - cualquier campo que acabe en *_id -> keyword con normalizer
      {
        "ids_as_keyword": {
          "match": "*_id",
          "mapping": { "type": "keyword", "normalizer": "kw_norm" }
        }
      },

      # meta como flat_object:
      # - “metadata variable” sin crear 1000 mappings distintos.
      #
      # PERO: flat_object NO es flattened.
      # - flattened (Elastic) permite meta.proveedor="canarias" como string directo.
      # - flat_object (OS) modela un “objeto plano” y su forma de indexación es distinta.
      #
      # Para evitar líos, en este lab dejamos meta/metadata definidos explícitamente y
      # en el bulk se usa el formato compatible (ver sección 3).
      {
        "meta_container_as_flat_object": {
          "path_match": "meta",
          "mapping": { "type": "flat_object" }
        }
      },

      # Regla por defecto para strings:
      # - text con analyzer + search_analyzer
      # - multifield raw (keyword) para aggs/sort/term exacto
      {
        "strings_default": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "text",
            "analyzer": "spanish_text",
            "search_analyzer": "search_with_synonyms",
            "fields": {
              "raw": { "type": "keyword", "ignore_above": 256, "normalizer": "kw_norm" }
            }
          }
        }
      }
    ],

    "properties": {
      # ----------------------------------------
      # Multi-tenant base (lógico):
      # tenant_id como keyword + normalizer (acento/case-insensitive)
      "tenant_id": { "type": "keyword", "normalizer": "kw_norm" },

      # id (del negocio) distinto del _id interno (puedes usar ambos)
      "id": { "type": "keyword" },

      # ----------------------------------------
      # Campos de texto “importantes” con multifields
      "nombre": {
        "type": "text",
        "analyzer": "spanish_text",
        "search_analyzer": "search_with_synonyms",

        # multifields:
        # - nombre.raw (keyword) => sort/aggs/term
        # - nombre.ac  (text edge_ngram) => autocomplete
        "fields": {
          "raw": { "type": "keyword", "ignore_above": 256, "normalizer": "kw_norm" },
          "ac":  { "type": "text", "analyzer": "autocomplete_edge", "search_analyzer": "spanish_text" }
        },

        # copy_to: campo “buscador global”
        "copy_to": ["fulltext"]
      },

      "ingredientes": {
        "type": "text",
        "analyzer": "spanish_text",
        "search_analyzer": "search_with_synonyms",
        "fields": {
          "raw": { "type": "keyword", "ignore_above": 256, "normalizer": "kw_norm" }
        },
        "copy_to": ["fulltext"]
      },

      # Campo “global” (copy_to)
      "fulltext": {
        "type": "text",
        "analyzer": "spanish_text",
        "search_analyzer": "search_with_synonyms",

        # norms=false: reduce algo de tamaño si no necesitas normalización por longitud (depende de ranking deseado)
        "norms": false,

        # positions: permite phrase queries / proximity (más coste, más capacidades)
        "index_options": "positions"
      },

      # ----------------------------------------
      # Tipos numéricos / fecha
      "tiempo_preparacion": { "type": "integer" },

      # scaled_float: dinero/precio con 2 decimales sin usar double (menos problemas de coma flotante)
      "precio": { "type": "scaled_float", "scaling_factor": 100 },

      "fecha_creacion": { "type": "date" },

      # ----------------------------------------
      # Keyword con null_value (cuando viene null, se indexa como "__NULL__")
      "tipo_comida": { "type": "keyword", "normalizer": "kw_norm", "null_value": "__NULL__" },

      # tags: keyword multi-valor
      "tags": { "type": "keyword", "normalizer": "kw_norm", "ignore_above": 128 },

      # ----------------------------------------
      # Meta flexible: flat_object
      "meta": { "type": "flat_object" },
      "metadata": { "type": "flat_object" },

      # ----------------------------------------
      # payload_original: guardo el JSON en _source para trazabilidad, pero NO indexo nada
      # (no se crean campos, no se puede buscar dentro, no consume mapping/ram)
      "payload_original": { "type": "object", "enabled": false }
    }
  }
}

# Inspección: ver settings/mapping (para “ver la criatura”)
GET recetas_base_v2/_settings?pretty
GET recetas_base_v2/_mapping?pretty



# =====================================================================================
# 2) ANALYZE — ver tokens
# =====================================================================================
# Esto es el “microscopio”:
# - Te dice exactamente qué tokens se generan.
# - Si cambias analyzer/filters, aquí ves el efecto.
# =====================================================================================

# Analyzer base (index-time)
POST recetas_base_v2/_analyze
{
  "analyzer": "spanish_text",
  "text": "Croquetas de marisco y jamón"
}

# Analyzer de búsqueda con sinónimos (search-time)
POST recetas_base_v2/_analyze
{
  "analyzer": "search_with_synonyms",
  "text": "patatas con bacon"
}

# Autocomplete edge-ngram: prefijos
POST recetas_base_v2/_analyze
{
  "analyzer": "autocomplete_edge",
  "text": "Croquetas"
}



# =====================================================================================
# 3) INGEST + QUERIES BÁSICAS
# =====================================================================================
# Bulk:
# - NDJSON: cada línea es un JSON
# - 1ª línea acción (index)
# - 2ª línea documento
#
# NOTA CRÍTICA (flat_object):
# - En OpenSearch, flat_object NO acepta el objeto arbitrario como “meta.proveedor":"canarias" igual que flattened.
# - Para que sea estable y no mezclar con el tema, aquí usamos metadata como objeto normal (metadata)
#   y meta (flat_object) lo alimentamos con un FORMATO plano compatible (ver doc 3).
#
# Si quieres “metadata libre” estilo flattened de Elastic, en OpenSearch lo habitual es:
# - (A) modelar explícito
# - (B) guardar en enabled:false (solo _source)
# - (C) usar un esquema controlado (o plugin) según caso
# =====================================================================================

POST recetas_base_v2/_bulk
{"index":{"_id":"1"}}
{"tenant_id":"ACMÉ","id":"1","nombre":"Croquetas de jamón","ingredientes":"jamón bechamel","tiempo_preparacion":25,"precio":3.50,"fecha_creacion":"2026-01-22T10:00:00Z","tipo_comida":"ES","tags":["FREIR","ES"],"metadata":{"origen":"abuelita"},"payload_original":{"raw":"JSON enorme que no quiero indexar"}}
{"index":{"_id":"2"}}
{"tenant_id":"acme","id":"2","nombre":"Croquetas de marisco","ingredientes":"mariscos bechamel","tiempo_preparacion":30,"precio":4.20,"fecha_creacion":"2026-01-22T10:05:00Z","tipo_comida":null,"tags":["FREIR","ES"],"metadata":{"origen":"costa"},"payload_original":{"raw":"otro JSON enorme"}}
{"index":{"_id":"3"}}
{"tenant_id":"Globex","id":"3","nombre":"Papas arrugadas","ingredientes":"papas sal mojo","tiempo_preparacion":30,"precio":2.10,"fecha_creacion":"2026-01-22T11:00:00Z","tipo_comida":"ES","tags":["HERVIR","ES"],
 "meta":{"proveedor":"canarias","lote":"X1"}}
{"index":{"_id":"4"}}
{"tenant_id":"Globex","id":"4","nombre":"Paella de marisco","ingredientes":"arroz marisco caldo","tiempo_preparacion":45,"precio":12.90,"fecha_creacion":"2026-01-22T11:30:00Z","tipo_comida":"ES","tags":["ARROZ","ES"]}

# (Opcional) refrescar para ver resultados inmediato sin esperar refresh_interval
POST recetas_base_v2/_refresh


# match sobre fulltext (copy_to)
# match:
# - analiza la query
# - calcula scoring
GET recetas_base_v2/_search
{
  "query": { "match": { "fulltext": "bechamel jamon" } }
}

# Sinónimos (bacon ≈ jamón)
GET recetas_base_v2/_search
{
  "query": { "match": { "fulltext": "bacon" } }
}

# Sinónimos (patatas ≈ papas)
GET recetas_base_v2/_search
{
  "query": { "match": { "fulltext": "patatas" } }
}

# term exacto sobre keyword normalizado
# term:
# - NO analiza
# - exige coincidencia exacta en keyword
# Aquí funciona aunque en el doc venga "ACMÉ" porque tenant_id tiene normalizer kw_norm
GET recetas_base_v2/_search
{
  "query": { "term": { "tenant_id": "acme" } }
}

# null_value: tipo_comida=null se indexa como "__NULL__"
GET recetas_base_v2/_search
{
  "query": { "term": { "tipo_comida": "__NULL__" } }
}

# Aggregation por tags (keyword)
# size:0 => no devuelvas hits, solo buckets
GET recetas_base_v2/_search
{
  "size": 0,
  "aggs": { "por_tag": { "terms": { "field": "tags" } } }
}

# Sort por nombre.raw (keyword)
# No se hace sort por text, se hace por keyword (doc_values)
GET recetas_base_v2/_search
{
  "query": { "match_all": {} },
  "sort": [{ "nombre.raw": "asc" }]
}

# Autocomplete por nombre.ac (edge_ngram)
GET recetas_base_v2/_search
{
  "query": { "match": { "nombre.ac": "croq" } }
}

# enabled:false: payload_original existe en _source pero no es searchable
GET recetas_base_v2/_doc/1?pretty



# =====================================================================================
# 4) OBJECT vs NESTED — “bug conceptual”
# =====================================================================================
# object:
# - arrays de objetos se “aplanan” y pueden cruzar atributos => falso positivo
# nested:
# - cada elemento del array se indexa como mini-doc oculto => coherencia, pero más coste
# =====================================================================================

PUT recetas_object_v1
{
  "mappings": {
    "properties": {
      "nombre": { "type": "text" },
      "ingredientes_detalle": {
        "type": "object",
        "properties": {
          "nombre": { "type": "keyword" },
          "cantidad": { "type": "integer" },
          "unidad": { "type": "keyword" }
        }
      }
    }
  }
}

POST recetas_object_v1/_doc/1
{
  "nombre":"Tortilla",
  "ingredientes_detalle":[
    {"nombre":"huevo","cantidad":1,"unidad":"ud"},
    {"nombre":"papa","cantidad":2,"unidad":"ud"}
  ]
}

# Query tramposa: huevo + cantidad=2
# Con object, puede devolver match aunque NO exista un ingrediente “huevo con cantidad 2”
GET recetas_object_v1/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "ingredientes_detalle.nombre": "huevo" } },
        { "term": { "ingredientes_detalle.cantidad": 2 } }
      ]
    }
  }
}

PUT recetas_nested_v1
{
  "mappings": {
    "properties": {
      "nombre": { "type": "text" },
      "ingredientes_detalle": {
        "type": "nested",
        "properties": {
          "nombre": { "type": "keyword" },
          "cantidad": { "type": "integer" },
          "unidad": { "type": "keyword" }
        }
      }
    }
  }
}

POST recetas_nested_v1/_doc/1
{
  "nombre":"Tortilla",
  "ingredientes_detalle":[
    {"nombre":"huevo","cantidad":1,"unidad":"ud"},
    {"nombre":"papa","cantidad":2,"unidad":"ud"}
  ]
}

# Misma intención, pero nested => ahora NO debería devolver nada
GET recetas_nested_v1/_search
{
  "query": {
    "nested": {
      "path": "ingredientes_detalle",
      "query": {
        "bool": {
          "must": [
            { "term": { "ingredientes_detalle.nombre": "huevo" } },
            { "term": { "ingredientes_detalle.cantidad": 2 } }
          ]
        }
      }
    }
  }
}



# =====================================================================================
# 5) ALIAS read/write + BLUE/GREEN (v2 -> v3)
# =====================================================================================
# Alias como contrato:
# - recetas_read  => lectura (puede apuntar a 1 o varios índices)
# - recetas_write => escritura (solo 1 write index activo)
#
# Blue/Green:
# - Creo v3
# - Reindex v2 -> v3
# - Swap alias (atómico) => la app “ni se entera”
# =====================================================================================

POST _aliases
{
  "actions": [
    { "add": { "index": "recetas_base_v2", "alias": "recetas_read" } },
    { "add": { "index": "recetas_base_v2", "alias": "recetas_write", "is_write_index": true } }
  ]
}

GET _cat/aliases/recetas_*?v

# Escritura por alias (contrato)
POST recetas_write/_doc
{
  "tenant_id":"acme",
  "id":"10",
  "nombre":"Gazpacho",
  "ingredientes":"tomate pepino",
  "tiempo_preparacion":20,
  "precio":2.50,
  "fecha_creacion":"2026-01-22T12:10:00Z",
  "tipo_comida":"ES",
  "tags":["FRIO","ES"]
}

# v3: aquí metemos nested “real” como parte del índice principal
PUT recetas_base_v3
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "1s",
    "analysis": {
      "normalizer": {
        "kw_norm": { "type": "custom", "filter": ["lowercase","asciifolding"] }
      },
      "filter": {
        "es_synonyms": {
          "type": "synonym_graph",
          "synonyms": ["jamon, jamón, bacon","papas, patatas","marisco, mariscos"]
        }
      },
      "analyzer": {
        "spanish_text": {
          "type":"custom","tokenizer":"standard",
          "filter":["lowercase","asciifolding","stop","snowball"]
        },
        "search_with_synonyms": {
          "type":"custom","tokenizer":"standard",
          "filter":["lowercase","asciifolding","es_synonyms","stop","snowball"]
        }
      }
    }
  },
  "mappings": {
    # dynamic strict: a partir de aquí solo lo que declaro
    # (producción: suele ser lo deseable)
    "dynamic": "strict",
    "properties": {
      "tenant_id": { "type":"keyword","normalizer":"kw_norm" },
      "id": { "type":"keyword" },

      "nombre": {
        "type":"text",
        "analyzer":"spanish_text",
        "search_analyzer":"search_with_synonyms",
        "fields": { "raw": { "type":"keyword","normalizer":"kw_norm","ignore_above":256 } },
        "copy_to":["fulltext"]
      },

      "ingredientes": {
        "type":"text",
        "analyzer":"spanish_text",
        "search_analyzer":"search_with_synonyms",
        "fields": { "raw": { "type":"keyword","normalizer":"kw_norm","ignore_above":256 } },
        "copy_to":["fulltext"]
      },

      "ingredientes_detalle": {
        "type":"nested",
        "properties": {
          "nombre": { "type":"keyword","normalizer":"kw_norm" },
          "cantidad": { "type":"integer" },
          "unidad": { "type":"keyword","normalizer":"kw_norm" }
        }
      },

      "fulltext": {
        "type":"text",
        "analyzer":"spanish_text",
        "search_analyzer":"search_with_synonyms",
        "norms": false,
        "index_options":"positions"
      },

      "tiempo_preparacion": { "type":"integer" },
      "precio": { "type":"scaled_float", "scaling_factor": 100 },
      "fecha_creacion": { "type":"date" },
      "tipo_comida": { "type":"keyword","normalizer":"kw_norm","null_value":"__NULL__" },
      "tags": { "type":"keyword","normalizer":"kw_norm","ignore_above": 128 }
    }
  }
}

# Reindex:
# - copia docs de v2 a v3
# - se usa para migraciones / cambios de mapping “no compatibles”
POST _reindex?wait_for_completion=true
{
  "source": { "index": "recetas_base_v2" },
  "dest":   { "index": "recetas_base_v3" }
}

# Swap alias (atómico)
POST _aliases
{
  "actions": [
    { "remove": { "index": "recetas_base_v2", "alias": "recetas_read" } },
    { "remove": { "index": "recetas_base_v2", "alias": "recetas_write" } },
    { "add":    { "index": "recetas_base_v3", "alias": "recetas_read" } },
    { "add":    { "index": "recetas_base_v3", "alias": "recetas_write", "is_write_index": true } }
  ]
}

GET _cat/aliases/recetas_*?v



# =====================================================================================
# 6) TIME-SERIES (logs): TEMPLATE + ROLLOVER + ALIAS
# =====================================================================================
# Patrón típico:
# - logs-000001, logs-000002, ...
# - logs_write apunta al índice “activo”
# - logs_read puede apuntar a todos (o a un subconjunto)
#
# Ventaja:
# - Retención = borrar índices enteros (barato)
# - Rollover = cortar por tamaño/docs/edad (control del tamaño de shard)
# =====================================================================================

PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "refresh_interval": "5s"
    },
    "mappings": {
      "dynamic": false,
      "properties": {
        "@timestamp": { "type": "date" },
        "service": { "type": "keyword" },
        "level": { "type": "keyword" },
        "host": { "type": "keyword" },
        "message": { "type": "text" }
      }
    }
  }
}

PUT logs-000001
{
  "aliases": {
    "logs_write": { "is_write_index": true },
    "logs_read": {}
  }
}

POST logs_write/_doc
{
  "@timestamp":"2026-01-22T08:00:00Z",
  "service":"api",
  "level":"INFO",
  "host":"node1",
  "message":"Boot"
}

# Rollover forzado (para demostrarlo sin esperar tamaño/edad):
# - max_docs=1 => en cuanto haya 1 doc, crea logs-000002 y mueve logs_write
POST logs_write/_rollover
{
  "conditions": { "max_docs": 1 }
}

GET _cat/indices/logs-*?v
GET _cat/aliases/logs_write?v



# =====================================================================================
# 7) MULTI-TENANT: FÍSICO vs LÓGICO
# =====================================================================================
# Físico (índice por tenant):
# - aislamiento simple
# - permisos por índice más directos
# - puede explotar número de índices si hay muchos tenants
#
# Lógico (índice compartido):
# - escala mejor con miles de tenants
# - exige disciplina: SIEMPRE filtrar por tenant_id en queries
# - control de “ballenas”: routing / índices dedicados / tiering
# =====================================================================================

PUT acme-recetas_v1
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "nombre": { "type": "text" },
      "tiempo_preparacion": { "type": "integer" }
    }
  }
}

PUT globex-recetas_v1
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "nombre": { "type": "text" },
      "tiempo_preparacion": { "type": "integer" }
    }
  }
}

POST acme-recetas_v1/_doc
{ "nombre":"Croquetas ACME","tiempo_preparacion":25 }

POST globex-recetas_v1/_doc
{ "nombre":"Paella Globex","tiempo_preparacion":45 }


PUT recetas_shared_v1
{
  "settings": { "number_of_shards": 2, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "tenant_id": { "type": "keyword" },
      "nombre": { "type": "text" },
      "tiempo_preparacion": { "type": "integer" }
    }
  }
}

POST recetas_shared_v1/_doc
{ "tenant_id":"acme","nombre":"Croquetas shared","tiempo_preparacion":25 }

POST recetas_shared_v1/_doc
{ "tenant_id":"globex","nombre":"Paella shared","tiempo_preparacion":45 }

# term exacto por tenant (keyword)
GET recetas_shared_v1/_search
{
  "query": { "term": { "tenant_id": "acme" } }
}



# =====================================================================================
# 8) ROUTING + HOT SHARD (mecanismo)
# =====================================================================================
# routing:
# - Por defecto, el routing usa _id (hash) => intenta repartir uniforme entre shards.
# - Si tú fuerzas routing=acme:
#     hash("acme") => SIEMPRE cae en el mismo shard (para ese índice)
#   Resultado: concentras (menos fan-out) PERO puedes crear hot shard si acme “es ballena”.
#
# En serio:
# - routing mejora latencia cuando SIEMPRE consultas por esa clave
# - routing puede destrozar balance si el reparto de tenants no es homogéneo
# =====================================================================================

POST recetas_write/_doc?routing=acme
{
  "tenant_id":"acme",
  "id":"9001",
  "nombre":"Receta ACME 1",
  "ingredientes":"bacon patatas marisco",
  "tiempo_preparacion":5,
  "precio":1.00,
  "fecha_creacion":"2026-01-22T12:20:00Z",
  "tipo_comida":"ES",
  "tags":["ACME"]
}

# search con routing:
# - “toca” menos shards (ideal si tu patrón de acceso es por tenant)
GET recetas_read/_search?routing=acme
{
  "query": {
    "bool": {
      "filter": [{ "term": { "tenant_id": "acme" } }],
      "must":   [{ "match": { "fulltext": "marisco" } }]
    }
  }
}

# Ver qué índice real hay detrás del alias recetas_read
GET _cat/aliases/recetas_read?v

# Ver shards del índice (distribución, primarios/réplicas, nodos)
# (Si el alias apunta a recetas_base_v3, esto te lo enseña directo)
GET _cat/shards/recetas_base_v3?v



# =====================================================================================
# 9) KNOBS OPERATIVOS: replicas + refresh_interval
# =====================================================================================
# number_of_replicas:
# - sube HA y throughput de lectura (más copias para servir búsquedas)
# - sube coste de almacenamiento e ingest (hay que replicar)
#
# refresh_interval:
# - sube ingest cuando lo subes (menos refreshes)
# - baja “near real time” (tarda más en verse en búsqueda)
# =====================================================================================

PUT recetas_base_v3/_settings
{
  "index": { "number_of_replicas": 2 }
}

PUT recetas_base_v3/_settings
{
  "index": { "refresh_interval": "10s" }
}
```
