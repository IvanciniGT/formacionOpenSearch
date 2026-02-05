
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





sin()
log()


66.4M de parametros y cada parametro es un double que son 8 bytes => 66.4M * 8 bytes = 531.2MB

Un modelo con 1 Billon de parametros => 1B * 8 bytes = 8GB
Pero los hay con cientos de billones de parametros => 100B * 8 bytes = 800GB


11.76B -> 11.76B * 8 bytes = 94.08GB
Metelo en RAM.
Y ahora operar... al meno cada vez que lo ejecutes se harán 11.76B operaciones... y cada operación es una multiplicación de un double por otro double => 11.76B * 8 bytes = 94.08GB de operaciones... y eso sin contar las operaciones que se hacen con los datos de entrada... etc.
Eso pesa.


   Modelo 1    Modelo 2
Video ---> TEXT ---> VECTOR (embedding) --->
           TEXT ---> VECTOR (embedding) --->

