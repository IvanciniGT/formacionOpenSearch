
PUT /_cluster/settings
{
  "persistent": {
    "plugins.ml_commons.only_run_on_ml_node": false
  }
}

POST /_plugins/_ml/models/_register?deploy=true
{
  "name": "huggingface/sentence-transformers/msmarco-distilbert-base-tas-b",
  "version": "1.0.3",
  "model_format": "TORCH_SCRIPT"
}
GET /_plugins/_ml/models/_search
{
  "query": {
    "match_all": {}
  },
  "size": 50
}
GET /_plugins/_ml/models/OwrILpwB7nQkEVYFlFqW
PUT /_ingest/pipeline/nlp-pipeline
{
  "description": "text -> embedding (text_vector)",
  "processors": [
    {
      "text_embedding": {
        "model_id": "OwrILpwB7nQkEVYFlFqW",
        "field_map": {
          "text": "text_vector"
        }
      }
    }
  ]
}
PUT /nlp-demo
{
  "settings": {
    "index.knn": true,
    "default_pipeline": "nlp-pipeline"
  },
  "mappings": {
    "properties": {
      "text": { "type": "text" },
      "text_vector": {
        "type": "knn_vector",
        "dimension": 768,
        "method": {
          "name": "hnsw",
          "engine": "lucene",
          "space_type": "innerproduct"
        }
      }
    }
  }
}
POST /nlp-demo/_doc/1
{ "text": "OpenSearch permite búsquedas semánticas con embeddings y k-NN." }

POST /nlp-demo/_doc/2
{ "text": "Kafka es una plataforma de streaming distribuido para eventos." }

POST /nlp-demo/_doc/3
{ "text": "BM25 es un ranking clásico para búsqueda lexical basada en términos." }

POST /nlp-demo/_refresh
GET /nlp-demo/_search
{
  "size": 5,
  "query": {
    "neural": {
      "text_vector": {
        "query_text": "búsqueda mediante significados en OpenSearch",
        "model_id": "OwrILpwB7nQkEVYFlFqW",
        "k": 5
      }
    }
  }
}
