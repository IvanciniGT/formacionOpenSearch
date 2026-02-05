
PUT /indice-vectorial
{
  "settings": {
    "index": {
      "knn": true
    }
  },
  "mappings": {
    "properties": {
      "nombre_producto": {
        "type": "text"
      },
      "caracteristicas_vector": {
        "type": "knn_vector",
        "dimension": 3,
        "method": {
          "name": "hnsw",
          "engine": "lucene",
          "space_type": "l2"
        }
      }
    }
  }
}

POST /indice-vectorial/_bulk
{ "index": { "_id": "1" } }
{ "nombre_producto": "Laptop Gamer Alienware", "caracteristicas_vector": [0.9, 0.9, 0.9] }
{ "index": { "_id": "2" } }
{ "nombre_producto": "Ratón USB Básico", "caracteristicas_vector": [0.1, 0.2, 0.1] }
{ "index": { "_id": "3" } }
{ "nombre_producto": "Laptop Oficina Dell", "caracteristicas_vector": [0.5, 0.5, 0.5] }

GET /indice-vectorial/_search
{
  "size": 2,
  "query": {
    "knn": {
      "caracteristicas_vector": {
        "vector": [0.8, 0.9, 0.9],
        "k": 2
      }
    }
  }
}





