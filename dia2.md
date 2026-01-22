
# Indices

Los índices es donde se cargan los documentos. Guardaremos el documento (una copia de él en el índice? depende de la configuración).

## Documentos

ES/OS lo que trabajan es con documentos JSON.
Aquí no meto un pdf. -> convertir en texto -> JSON -> OS

Una gracia que tiene JSON y muchos otros formatos es que los documentos tienen estructura, que además no necesariemente es linea, ni fija... Puede ser variables y arboladas.

```json
    {
        "nombre": "Tortilla de patatas",
        "ingredientes": [
            {
                "ingrediente": "huevos",
                "cantidad": "4",
                "unidad": "unidades"
            },
            {
                "ingrediente": "patatas",
                "cantidad": "3",
                "unidad": "unidades"
            },
            {
                "ingrediente": "aceite de oliva",
                "cantidad": "100",
                "unidad": "ml"
            }
        ],
        "instrucciones": "Pelar y cortar las patatas. Freírlas en aceite caliente. Batir los huevos y mezclar con las patatas. Cocinar la mezcla en una sartén hasta que esté cuajada.",
        "tiempo_preparacion_minutos": 30,
        "dificultad": "media",
        "requiere_horno": false
    }
```

A la estructura que tiene un documento en JSON se le denomina ESQUEMA. De hecho existe un lenguaje para definir esquemas JSON (JSON Schema).

Cuando creamos un índice en ES/OS, una de las cosas que VAMOS a configurar (Ya que ES/OS tienen la MALDITA opción de no requerir que lo haga) son los campos que vamos guardar indexados de nuestros documento... así como el tratamiento que se le va a dar a cada campo (analizadores, tokenizadores, filtros, etc). Esto se conoce por el nombre de MAPPING (mapeo).

Todo índice tiene un mapping asociado.

### Mapping

Lo que definimos es los campos que vamos a guardar indexados de nuestros documentos, así como el tratamiento que se le va a dar a cada campo (analizadores, tokenizadores, filtros, etc).

Realmente, en un índice, definimos analizadores, tokenizadores, filtros, etc (todo eso se conoce como ANALYSIS), y luego definimos el mapping de los documentos, que usa esos analizadores, tokenizadores, filtros, etc.

```http
PUT http://miServidor:9200/recetas_italianas
{
  "settings": {
    "analysis": {...},
    otros settings
  },
    "mappings": {   
        ...
    },
    "aliases": {
        ...
    }
}
```

En realidad... esto tampoco es lo que hacemos.
Al final, vamos a acabar seguramente con montón de índices, para guardar cosas muy similares o iguales:
- recetas_españolas
- recetas_italianas
- recetas_mexicanas

- log_app1_enero2024
- log_app1_febrero2024
- log_app2_enero2024
- log_app2_febrero2024

En la práctica, lo que hacemos es definir un TEMPLATE de índice, que define settings, mappings, y luego creamos índices que usan ese template.

Además eso es muy cómodo.. trabajamos con patrones de nombres de índices.

Plantilla que aplique a todos los índices que se creen que empiecen por "recetas_"

### Tipos de datos que definimos en el mapping

- "text"         Son los campos con textos sobre los que vamos a hacer búsquedas full-text.
- "keyword"      Son los campos con textos sobre los que vamos no vamos a hacer búsquedas full-text, sino:
                -  búsquedas exactas
                -  agregaciones
                -  ordenaciones
   SOLO CON ESOS 2 ya tenemos el lío co los mappings automáticos que hace ES/OS.
   Además... puede ser que incluso me interese indexar un campo como "text" y "keyword" a la vez.
   Por ejemplo... tengo el campo "nombre de la receta": quiero busquedas a texto completo? SI -> TEXT
                            Quiero poder sacar un cuadro de mando agreagando por nombre de receta? SI -> KEYWORD

ES/OS no guardan documentos... NI CAMPOS... generan índices! Lo que no esté en un índice no puedo usarlo para nada.
    {
        "id": 123,
        "nombre": "Tortilla de patatas"
        ...
    }
Una copia de ese documento, quizás la guardo.. y esa copia se usa para: en las búsquedas, mostrar contexto del resultado, etc. PARA NADA MAS!
Y ahora puedo coger el campo nombre y guardarlo en un índice inverso:
    totilla -> 123(1)
    patatas -> 123(3)

    Y eso me sirve para hacer búsqueda fulltext? SI!
    Eso me sirve para mostrar en una tabla el nombre de la receta? NO!
    Si quiero hacer esto, tengo que guardar el campo nombre en otro índice KEYWORD: (nombre.keyword | nombre.raw)
    "Tortilla de patatas" -> 123

Un documento... 1Kb... puede dar lugar en ES a 12 Kbs de datos.

Otros tipos de datos:
- interger
- float
- double
- date
- boolean
- arrays
- object | nested 
   Eso en nuestro ejemplo de las recetas, el campo ingredientes.
   Si los guardo todos juntos: object se aplana la información.
   Si busco: Dame una receta con más de cantidad > 100  nombre_ingrediente= huevos... me puede sacar una receta que tenga 3 huevos y 200 patatas.
   Si necesito que se conserve la relación entre los campos dentro del array, uso nested.
- geo_point / geo_shape
   Para datos geoespaciales.
- ip
   Para direcciones IP.
   Luego podemos hacer búsquedas por tramos: 192.168.0.0/16

Lo habitual es comenzar con un índice con mapping dinámico (ES crea el mapping según los documentos que voy metiendo), y luego ir refinándolo y cerrarlo.
A la hora de definir el índice podemos elegir si queremos:
```
dynamic: true | false | strict
```

true: ES/OS me va a crear el mapping según los documentos que voy metiendo.
false: Ignore campos nuevos que vayan llegando.
strict: Si llega un campo nuevo que no está en el mapping, ERROR. <--- Ideal para producción (pero... depende de la situación).


## Analysis

Además de los mappings, en los índices definimos el procedimiento de análisis de los textos.

Analyzer = char_filter + tokenizer + token_filter(s)

> "Croqueta de <b>jamón</b>"
> char_filter -> "Croqueta de jamón"        QUITAR ETIQUETAS HTML, guiones, etc
> tokenizer   -> ["Croqueta", "de", "jamón"]    whitespace, ngram, etc
> token_filter -> ["croqueta", "jamon"]     lowercase, stopwords, stemming

### Tokenizadores:
- standard (palabras separadas por espacios y puntuación)
- whitespace (palabras separadas por espacios)
- keyword (todo el texto como un solo token)
- edge_ngram (subcadenas desde el inicio)       <--- Autocompletar (empieza por)
- ngram (subcadenas en todo el texto)            <--- contiene  (CUIDAO AQUI.. que el tamaño del índice se dispara)

    Croqueta de jamón -> ngram (3,5)
    "Cro", "roqu", "oque", "quet", "eta", "ta ", "a d", " de ", "de j", "e ja", " jam", "jamó", "amón"
                      -> edge_ngram (3,5)
                        "Cro", "Croq", "Croqu"


        Nombre de la receta: Tortilla de patatas          <- standard
        
        Como funcionará entonces el buscador?
            App: Pantalla buscar receta: [                        |BUSCAR]
                                           patatas         Funciona? SI
                                           patata          Funciona? DEPENDE del token_filter
                                           pat             Funciona? NO
                                           ^^^ Si quiero esos: EDGE_NGRAM
        Entro en netflix.. y busco pelicula: "sta" -> Me salen resultados ... que están usando? EDGE_NGRAM

# token_filter:

- lowercase
- ascii_folding (quita tildes, ñ -> n, ç -> c, etc)
- stop (quita palabras vacías: el, la, de, y, etc) preposiciones, artículos, conjunciones, etc
- snowball | stemmer (raíz de la palabra) correr, corriendo, corrí -> corr  (plurales, tiempos verbales, género, etc)
- synonym (sinónimos) cine, película | synonym_graph

Esos de ahí son analyzers... Luego están los normalizers (como analyzers pero sin tokenizers, para campos keyword).

Solo llevan filtros

# Adicionalmente... Settings

Los settings son configuraciones adicionales del índice.
Y OJO! Hay settings que no se pueden cambiar una vez creado el índice.

- Número de shards y réplicas:
    - number_of_shards: 3         <- Este NO puedo cambiarlo luego. (Shrinking y splitting)
    - number_of_replicas: 1       <- Este si puedo cambiarlo luego.
- refresh_interval: 1s   <- Cada cuanto tiempo se actualiza el índice para que las búsquedas vean los nuevos documentos.
   Si es más o menos importante que los nuevos documentos estén disponibles rápido para búsquedas.
   Si no es tan importante lo puedo poner a 30s o más, y así mejoro el rendimiento de las escrituras.
- max_result_window: 10000   <- Por defecto, las búsquedas solo pueden devolver 10.000 resultados.
- Tiempo que debe transcurrir antes de que ES/OS reasigne los shards que estaban en un nodo que ha caído.
  - index.unassigned.node_left.delayed_timeout: 36000m
    OJO! Hay una forma global de desactivar la reasignación automática de shards (No la queremos):
        Existen distintos tipos de nodos data:
        - data_hot
        - data_warm
        - data_cold
        Los nodos podrán tener acceso a sistemas de almacenamiento más rápidos o más lentos.
        Índices que se estén escribiendo y son muy activos -> data_hot   NVME
        Índices que ya no se escriben pero se consultan -> data_warm     SSD
        Índices que ya no se escriben ni se consultan -> data_cold       HDD
- max_terms_count: 65536   <- Número máximo de términos de una consulta.
- index.routing.allocation.require.tipo: rapidito
  <- Para definir que shards se asignan a qué tipo de nodos.
    A los nodos les puedo poner etiquetas: tipo: rapidito | lento


El número de shards es crítico. Claro.. es algo que no puedo cambiar luego.
La realidad es que me preocupa poco.

Si tengo un buen esquema de índices...
log_enero (2 shards.. y me habría venido genial tener 3 shards)...
Pues ya para el próximo mes hago log_febrero (3 shards).


```bash
curl -X PUT "http://localhost:9200/recetas_base_v1" -H 'Content-Type: application/json' -d '{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "1s",
    "analysis": {
      "normalizer": {
        "kw_norm": {
          "type": "custom",
          "filter": ["lowercase", "asciifolding"]
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
      },
      "filter": {
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 15
        }
      }
    }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "tenant_id": { "type": "keyword", "normalizer": "kw_norm" },
      "id": { "type": "keyword" },

      "nombre": {
        "type": "text",
        "analyzer": "spanish_text",
        "fields": {
          "raw": { "type": "keyword" },
          "ac":  { "type": "text", "analyzer": "autocomplete_edge", "search_analyzer": "spanish_text" }
        }
      },

      "ingredientes": {
        "type": "text",
        "analyzer": "spanish_text",
        "fields": {
          "raw": { "type": "keyword" }
        }
      },

      "tiempo_preparacion": { "type": "integer" },
      "precio": { "type": "scaled_float", "scaling_factor": 100 },

      "fecha_creacion": { "type": "date" },

      "tags": { "type": "keyword", "normalizer": "kw_norm" },

      "metadata": { "type": "flattened" }
    }
  }
}'
```


---

# Nota:

Toda comunicación con ES/OS se hace a través de su API RESTful. Lanzaremos peticiones HTTP (normalmente con JSON en el cuerpo) y recibiremos respuestas HTTP (normalmente con JSON en el cuerpo).
Cuando luego creamos apps, es frecuente usar librerías que nos ofrecen estas herramientas para distintas plataformas (Java, Python, JavaScript, etc), que hacen las llamadas HTTP por nosotors. Yo escribo JAVA, y esas librerías transforman el JAVA en llamadas HTTP a la API RESTful de ES/OS.

## Creación de un índice

```http
PUT http://miServidor:9200/recetas_italianas 
```

Eso es un desastre... o no!

---
## Desarrollo de software

### Gestión de un desarrollo de software

Antiguamente cuando arrancaba un proyecto de software, quería montar una app, seguíamos lo que se llamaba una metodología clásica (cascada, en V, etc). 

    REQUISITOS -> ANÁLISIS -> DISEÑO -> IMPLEMENTACIÓN -> PRUEBAS -> DESPLIEGUE -> MANTENIMIENTO

Hoy en día, hemos decidido mayoriatamente usar metodologías ágiles (Scrum, Kanban, etc).

La característica clave de las metodologías ágiles es ir entregando el producto de forma incremental, en pequeñas partes, y de forma iterativa (vamos mejorando esas partes en cada iteración).

Y entonces.. eso de hacer un diseño COMPLETO del sistema al principio del proyecto... pues como que no.

Esto tiene sus implicaciones.
Cuando trabajaba con el Oracle:
1º Montar el esquema de la base de datos completo!

La idea aquñi es diferente...
La idea es que un desarrollador pueda desde el minuto 0 empezar a cargar documentos en un índice, sin necesidad de tener que definir el mapping completo del índice.
Qué va a tener el documento? NPI.. yo que sé.. tengo 3 ejemplos... ya iré completando!

Dia 1 del desarrollo:

```http
PUT http://miServidor:9200/recetas_italianas
```

Y empieza a cargar datos... y funciono!

Según avanza mi desarrollo, las cosas van quedando más claras. Se van definiendo mejor los documentos que voy a usar.

ES, cuando creo un índice sin mapping, crea un mapping dinámico, que va adaptándose a los documentos que voy cargando.

Ese mapping tendrá PROBLEMAS FIJO ! SEGURO QUE LOS TIENE, NO ME VALEE!
Cuando tenga aquello más avanzado, me interesará exportar el mapping, revisarlo, corregirlo y volver a importarlo.

```http
GET http://miServidor:9200/recetas_italianas/_mapping
```
```http
PUT http://miServidor:9200/recetas_italianas
{
  "mappings": {
    ...
  }
}
```

A producción NO SUBE UN INDICE sin mapping!
En desarrollo es habitual que se use el mapping dinámico, pero en producción no.