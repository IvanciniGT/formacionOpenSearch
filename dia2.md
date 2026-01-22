
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
# Otro tipo de cosas que podemos hacer:
## Generar índices con combinaciones de campos para optimizar consultas específicas

```json
{
    "nombre": "Iván",
    "apellido1": "Osuna",
    "apellido2": "Ayuste",
    "email": "ivan.osuna.ayuste@gmail.com",
    "dni": "12345678Z"
}
´´´json

App una pantalla para buscar personas:

    nombre:         [       ]
    apellido1:      [       ]
    apellido2:      [       ]
    email:          [       ]
    dni:            [       ]
                                BUSCAR


    Datos de la persona: [             | BUSCAR ]

        Que escriban ahí lo que sepan de la persona.

    Me puedo crear un índice juntando todos esos campos en uno solo:
    "mappings": {
        "properties": {
            "nombre": {
                "type": "keyword",
                "copy_to": "completo"
            },
            "apellido1": {
                "type": "keyword",
                "copy_to": "completo"
            },
            "apellido2": {
                "type": "keyword",
                "copy_to": "completo"
            },
            "email": {
                "type": "keyword",
                "copy_to": "completo"
            },
            "dni": {
                "type": "keyword",
                "copy_to": "completo"
            },
            "completo": {
                "type": "text",
                "analyzer": "spanish_text"
            }   
        }
    }

---

Iván               -> En el índice del campo nombre se va a guardar el token: ivan + Id del documento (posición)
Osuna              -> En el índice del campo apellido1 se va a guardar el token: osuna + Id del documento (posición)
Ayuste
ivan.osuna.ayuste@gmail.com
12345678Z
"Iván Osuna Ayuste ivan.osuna.ayuste@gmail.com 12345678Z"

Cúanto ocupa esto en el índice?
Me puede opcupar mucho o muy poco.
De qué depende el tamaño de un índice?
- Términos únicos que tengo en el índice.
- Número de documentos.

Iván            -> En el índice del campo nombre, se va a guardar el Id del documento (posición) (el token ya existe) *1
García
Page

*1 O no existe? SI... o NO
Es decir.. en el índice, en disco... los datos se guardan el ficheros de segmentos... Y ahí se va a guardar el token?
Depende de si en la escritura actual que se está haciendo en el segmento ese token está en otro documento o no!
En los ficheros de segmento se hace un append-only... 

Resumen... cuanto ocuparan los ficheros de segmento? En general mucho más que los datos consolidados en RAM.
Al consolidar en RAM, el token aparece 1 sola vez. Pero en disco puede estar 500!

NOTA: POR ESTO ES TAN IMPORTANTE TENER LA OPCION DE CONGELAR UN ÍNDICE y aprovechar en ese momento a consolidar los segmentos y eliminar duplicados. Puedo liberar una cantidad importante de espacio en disco.
EL MANTENIMIENTO DE ÍNDICES ES CRÍTICO EN ES/OS.

Esto puede ir a más!

Cuántos shards tengo?
- Tengo 1 shard... de 5Gbs
- Si ahora decido divirlo en 2 shards... cuánto ocupará cada shard? 2,5Gbs? NO... más!
  Por qué? Habrá un montón de tokens que estarán duplicados en ambos shards.

En RAM... los índices ocupan menos que en disco. Eso es verdad? En general SI... por la consolidación de términos..
Pero el efeecto quee contaba arriba, ocurre también en RAM:
    1 shard que en RAM ocupa 500Mbs y lo parto en 2 shards, cada shard quizás me ocupa 300-325 Mbs... así que al final en RAM tengo 600-650Mbs. Ya que hay términos que tengo que poner en RAM en ambos shards / servidores.

Esto es algo a monitorizar en los índices. El % del tamaño del índice que está marcado por los términos.
Y hay mirar bien...
Hay índices que se estabilizan con el tiempo... A partir de cierto momento solo añaden ubicaciones de documentos a términos ya existentes.
Hay otros índices que siguen creciendo en términos nuevos constantemente.

Eso marca la forma en la que me planteo particionar los índices.

---

Si no tengo un índice que soporte la funcionalidad que quiero ofrcer estoy JODIDO.
La forma en la que defino el índice está TOTALMENTE CONDICIONADA AL TIPO DE CONSULTAS QUE VOY A HACER.

El índice se crea para responder a ciertas búsquedas.
Y de hecho puede haber 500 campos en el documento que me cargan que no me interesan para nada.. y no los indexo (Que OS va a indexarlos por defecto... si no le digo nada dynamic: false). Y tiro espacio a HDD a borbotones.
Indexaré lo que me interesa y en la forma que me interesa.

Y ESTA TOTALMENTE LIGADO A LA FUNCIONALIDAD QUE VOY A OFRECER.

1º Defino lo que quiero ofrecer.
2º Defino las búsquedas que necesito.
3º Defino los índices que necesito para soportar esas búsquedas.

Y es un problema... ya que si el día de mañana quiero ofrecer otra funcionalidad, puede que los índices que tengo no la soporten. Y me toca desistir o reindexar.

No puedo tener un autocompletar si no he definido un índice con un analyzer que soporte autocompletar (edge_ngram).

Y de vez en cuando (muy de vez en cuando) me toca reindexar.
Y en estos casos, el tener la copia de los documentos originales es crítico (_source).
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


---

# Arquitectura de Índices

Esto es crítico!

Aquí juegan un papel funcamental los ALIAS de los índices.
Y por otro lado, el hacer un buen esquema(arquitectura) de los índices.


## Formas de crear índices:

### Crear índices por dominio funcional

    recetas_italianas_v1
    recetas_mexicanas_v1
    recetas_españolas_v1

    logs_bbdd_v1
    logs_web_servers_v1
    logs_app_servers_v1

#### Notas:

Esto tiene sentido cuando:
- Datos con un mapping estable y similares, los junto en un mismo índice.
- Consultas sobre un mismo dominio funcional.
- Retención de datos larga.

#### Ventajas: 

- Sencillo
- Mapping estricto, controlado y versionado.

#### Inconvenientes / Riesgos:

- Crece sin control el tamaño del índice (voy a acabar con shards muy grandes).
- Si necesito cambiar el mapping, me toca reindexar todo el índice si quiero soportar la funcionalidad nueva en los documentos antiguos.


#### COMENTARIO IMPORTANTE PARA ESTOS ÍNDICES
De este tipo de índices, siempre me conviene poner un _vX al final.
Con el tiempo las cosas cambian, y puede que necesite cambiar el mapping o settings de un índice.

Quiero tener libertad de poder el día de mañana crear una nueva versión del índice, con otro mapping o settings.

    recetas_italianas_v2

Y por supuesto no quiero tener que cambiar ni una coma de mi aplicación.
Para eso uso ALIAS.
Mi app trabajará contra el alias: 
    recetas_italianas o incluso recetas o recetas_italianas_ultima

    recetas_italianas -> recetas_italianas_v1
Cuando cree la nueva versión del índice, solo tengo que cambiar el alias:
    recetas_italianas -> recetas_italianas_v1 , recetas_italianas_v2

Quizás en el futuro hago migración de los datos del v1 al v2, y luego quito el v1 del alias.

### Índices por tiempo (series de tiempo)

    logs_app_server_2024_01
    logs_app_server_2024_02
    logs_app_server_2024_03

* Esto es compatible con los índices por dominio funcional.

La idea es, si voy a ir metiendo muchos datos, y quiero facilitar las operaciones de mantenimiento (shrink de índices en los que ya no voy a meter nada), esto es ideal.

#### Cuando me interesa:

- Ingesta continua y alta de datos.
- Datos inmutables (no se modifican una vez escritos).
- Aplicar politicas de mantenimiento (ILM) diferentes según la antigüedad de los datos.
- Queries que se centran en rangos temporales.

#### Ventajas:
- Facilita el mantenimiento de los índices.
- Puedo ir quitando datos de forma sencilla (borrando / cerrando índices antiguos).

#### Inconvenientes / Riesgos:
- Voy a tener mucho espacio en disco / RAM usado por los índices debido a la duplicación de términos entre índices.

Si estoy cargando videos,
En enero tendré un huevo de videos de gatitos...
Y en febrero otro huevo de videos de gatitos...
Y en marzo otro huevo de videos de gatitos...
Cada índice tendrá los términos "video", "gatitos", "divertido", "gracioso", etc.

Los términos pueden ocupar un 20-30% del tamaño del índice.

1 índice grande -> 300Gbs
3 índices mensuales -> 130Gbs (30% más) x 3 = 390Gbs


### Índices por tenant / cliente

    clienteA_recetas_v1
    clienteB_recetas_v1

#### Cuando me interesa:

- Tengo un volumen pequeño de clientes
- Separación clara de datos entre clientes

#### Inconvenientes / Riesgos:

- Si tengo un volumen muy grande de clientes, los términos van a estar muy duplicados entre índices... mucha más RAM y disco usado.

Una alternativa a esto es usar routing para separar los datos de los distintos clientes en shards diferentes dentro de un mismo índice. No significa que vaya a haber 1 shard por cliente... pero si que los datos de un cliente estén en un shard concreto.

Esto nos permite reducir la duplicación de términos entre índices. y me mantiene la velocidad de las consultas (ya que solo tengo que buscar en 1 shard).

---

# Operaciones de mantenimiento de índices

- Shrink: Reducir el número de shards de un índice (en este proceso además, se consolidan los segmentos y se eliminan términos duplicados).
- Split: Aumentar el número de shards de un índice (solo si veo que estoy teniendo problemas de rendimiento en las escrituras). -> En general lleva aparejado un aumento del cluster (+CPUs, +RAM, etc).
- Freeze: Congelar un índice. Notifico a OS que ese índice va a dejar de recibir datos nuevos.
  - Lo que pasa ees que en este momento es cuando me compensa hacer un shrink del índice, y consolidar los segmentos.
    Esa es una operación muy pesada desde el punto de vista computacional. 
    No la voy a estar haciendo si mañana llegan datos nuevos... y se me jode otra vez la estructura de segmentos.
  Estando así, el índice se puede seguir consultando.
- Unfreeze: Descongelar un índice congelado.
- Close: Cerrar un índice. No se puede consultar ni escribir.
  - Lo que consigo es liberar toda la RAM que estaba usando ese índice.
  - El índice sigue en disco, pero no se usa para nada.
- Open: Abrir un índice cerrado.
- Delete: Borrar un índice.

Estas operaciones luego las suelo automatizar con ILM (Index Lifecycle Management).


---

Split:

1 índice y 3 nodos y 3 shards primarias
Tiene sentido subir a 6 shards primarias? DEPENDE

En principio si hago eso es porque quiero aumentar la capacidad de escritura del índice. Poder estar tragando más datos por segundo.
A priori si puede mejorar.. siempre y cuando el cuello de botella no esté en HDD.

Si estoy teniendo problemas... muy probablemente el cuello de botella esté en HDD.
Si tarda mucho , es que no soy capaz de escribir los datos en disco lo suficientemente rápido.
El meter más presión a disco duplicando el número de shards va a empeorar la situación.

Si tengo ya una copia de un shard en un nodo (sea el primario o una réplica), no puedo poner otra copia de ese shard en el mismo nodo.

Pero un nodo puede tener 17 primarios de un mismo índice (si tengo 17 shards primarios).

---

Monitorización... la teneís out of the box?

ES/OS hacen un desmadre de narices en los índices con este tipo de datos...
Gneran datos para indexar a miles...

Le meto una linea de un log de a lo mejor 200 bytes... y ES/OS genera 5Kbs-10Kbs de datos en índices.
De los que no voy a usar ni la cuarta parte!

Estos datos hay que filtarlos MUCHISISISMO!
Y haciendo eso, libero una cantidad importante de espacio en disco y RAM.


---

INDICE: content
    doc_type: container | s_object | item_X

    A priori, ES/OS no sabe en que shards del índice están los documentos que busco.
    Y le toca levantar todos los shards del índice para buscar.
    Y buscar en todos... 
    Y quizás los datos que busco están en 1 solo shard.... pero busco en todos.
    Con el routing puedo indicarle a ES/OS en que shard(s) buscar.
     - Libera RAM.. y libera CPU de los data nodes... que pueden hacer otras cosas.

 
INDICE: colecciones
    doc_type: collection | collection_item


---

# Routing

Hot index / hot shards

- Necesito indentificar que índices son hot (los que están recibiendo datos nuevos constantemente y sobre los que se hacen búsquedas frecuentes).
- Dentro de esos índices, necesito identificar que shards son hot (los que están recibiendo datos nuevos constantemente y sobre los que se hacen búsquedas frecuentes).

Si tengo un buen reparto de shards (balanceo) esto no es tan crítico (En general con un routing básico, por defecto ES/OS ya hace un buen reparto de shards).


En la indexación, puedo indicar cómo quiero que se haga el routing de los documentos.

POST http://miServidor:9200/recetas_italianas/_doc/1?routing=clienteA
{
    documento
}

En las búsquedas luego hay que tenerlo en cuenta:
GET http://miServidor:9200/recetas_italianas/_search?routing=clienteA
{
    consulta
}


---

# Limitar el número de resultados devueltos por una consulta es fundamental... cuanto más pueda limitarlo mejor.

Aquí hay 2 cosas: 
- Tamaño de página <---- Nodos coordinadores consolidan los datos de multiples shards... y reordeeenan los resultados...            
                        Eso es así, pero no reordenan todos los resultados... solo los que me van a devolver en la página.
- Total de resultados   En cualquier caso, ES/OS tienen que calcular el total de resultados que hay para una consulta.
                        Si digo que la consulta no puede devolver más de 10.000 resultados, ES/OS dejará de buscar y ordenar una vez que haya encontrado 10.000 resultados.

    Si tengo que devolver un máximo de 200 resultado en la página:
    Lucene1 (SHARD1) -> 4000 resultados - ... solo mira los primeros 200     Al sacar datos de ellos, en cuanto llega 
    Lucene2 (SHARD2) -> 3000 resultados - ... solo mira los primeros 200     a 200 también para 
    Lucene3 (SHARD3) -> 5000 resultados - ... solo mira los primeros 200
