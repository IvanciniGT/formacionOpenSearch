
# Qué es Opensearch?

Es un producto de software que proviene de otro producto llamado ElasticSearch. ElasticSearch lo fabrica una empresa llamada Elastic. En un momento dado cambiaron la licencia. Tenían una licencia OpenSource y ahora no... y hay gente que no le moló mucho ese cambio...
Y partiendo de la última versión OpenSource de ElasticSearch, Amazon creó Opensearch y lo liberó como OpenSource.

Realmente en este paquete tenemos diferetes productos.

ElasticSearc/Opensearch es un motor de indexación y búsqueda de documentos. No queda muy bien catalogarlo de BBDD No relacional, aunque la realidad es que en algunos (y solo en algunos) escenarios puede hacer las veces.

Está basado en Lucene, que es una librería OpenSource de la fundación Apache para indexación y búsqueda de documentos.

Realmente quién hace la indexación y búsqueda de documentos es Lucene, no Elasticsearch ni Opensearch. Qué pintan estos?
- Capa por encima de lucene para facilitar su uso
- Alta disponibilidad y escalabilidad (esto no existe en Lucene)

Una buena forma de definir Opensearch sería: Un orquestador de Lucene con alta disponibilidad y escalabilidad.

## Indexador de documentos?

Eso no lo hacen las BBDD? Si... pero con capacidades limitada.

### Índices

Tabla de datos: Recetas

    | ID | Nombre                   | Ingredientes          | Tiempo preparación |
    |----|--------------------------|-----------------------|--------------------|
    | 1  | Tortilla de patatas      | Huevos, patatas       | 15                 |
    | 2  | Ensalada de papas        | Lechuga, tomate       | 10                 |
    | 3  | Paella de marisco        | Arroz, mariscos       | 45                 |  
    | 4  | Gazpacho                 | Tomate, pepino        | 20                 |
    | 5  | Papas arrugadas          | Papas, sal marina     | 30                 |
    | 6  | Croquetas de jamón       | Jamón, bechamel       | 25                 |
    | 7  | Croquetas de marisco     | Mariscos, bechamel    | 30                 |


> SELECT * FROM RECETAS WHERE TIEMPO_PREPARACION < 20;

Cómo resuelve eso una BBDD a priori? Tiene que hacer un FULLSCAN de la tabla, leer todas las filas, verificando fila a fila si cumple la condición. Esto es lento. A pocos datos.. bueno... Con muchos datos.. innmanejable.
Un algoritmo FULLSCAN es un algotirmo de O(n)

Para evitar la degradación del sistema cuando empieza a crecer la tabla, las BBDD implementan índices.

### Índices en BBDD

Es lo mismo que un índice en un libro. 

Un índice es una copia ordenada de los datos junto con su ubicación.


    | ID | Nombre                   | Ingredientes          | Tiempo preparación |
    |----|--------------------------|-----------------------|--------------------|
    | 1  | Tortilla de patatas      | Huevos, patatas       | 15                 |
    | 2  | Ensalada de papas        | Lechuga, tomate       | 10                 |
    | 3  | Paella de marisco        | Arroz, mariscos       | 45                 |  
    | 4  | Gazpacho                 | Tomate, pepino        | 20                 |
    | 5  | Papas arrugadas          | Papas, sal marina     | 30                 |
    | 6  | Croquetas de jamón       | Jamón, bechamel       | 25                 |
    | 7  | Croquetas de marisco     | Mariscos, bechamel    | 30                 |

INDICE por Tiempo de preparación:

    10 -> Fila 2  

    15 -> Fila 1  

    20 -> Fila 4  

    25 -> Fila 6  
    27 -> Fila X TENIA EL HUECO RESERVADO (Habitualmente configuramos lo que se llama FILL FACTOR entorno al 50-75 que es el % de espacio reservado en cada página para futuras inserciones)
    30 -> Fila 5, Fila 7  



    45 -> Fila 3

El tener datos ordenados ofrece una ventaja gigante... ya que a partir de ese momento podemos usar otro tipo de algoritmos: BUSQUEDAS BINARIAS.

Una búsqueda binaria es la misma que hacemos en un DICCIONARIO!
10.000 palabras... Y corto la primera vez por 5.000... y me he pasado. Con esta única operación, descarto la mitad de las palabras = 5.000 operaciones.

10.000 -> 5000
5000 -> 2500
2500 -> 1250
1250 -> 625
625 -> 312
312 -> 156
156 -> 78
78 -> 39
39 -> 19
19 -> 9
9 -> 4
4 -> 2
2 -> 1
En 13 operaciones he encontrado la palabra.... Entre 10.000 palabras.

Una gracia de este tipo de algoritmos es que su orden de complejidad es O(log n).
Al crecer el tamaño de los datos, el número de operaciones crece muy lentamente.

Si tengo 1M de datos... serían 20 operaciones.
Si tengo 1B de datos... serían 30 operaciones. = log2(1.000.000.000.000) = 30

OJO: Los índices no son la panacea universal. Tienen sus contras:
- Ocupan espacio en disco = MUCHISIMO!
- Las actualizaciones son más lentas = Cada vez que hago un INSERT/UPDATE/DELETE tengo que actualizar el índice

Si vamos a hacer muchas más consultas que inserciones, nos compensan.

Eso que hemos descrito es lo que llamamos un índice directo. Un índice que apunta directamente a la fila de datos.

El problema es que ese tipo de índices está limitado a ciertos casos de uso.

INDICE Nombre
    Croquetas de jamón  -> Fila 6  
    Croquetas de marisco -> Fila 7  
    Ensalada de papas    -> Fila 2  
    Gazpacho            -> Fila 4  
    Papas arrugadas     -> Fila 5  
    Paella de marisco   -> Fila 3  
    Tortilla de patatas -> Fila 1

Si busco: Dame las recetas que se llaman: "Croquetas de jamón" el índice funciona cojonudo!
Si busco: Dame las recetas cuyo nombre comienza por Pa... el índice funciona cojonudo!
Si busco: Dame las recetas en cuyo nombre aparece la palabra marisco... el índice NO FUNCIONA!

PROBLEMÓN!... Para búsquedas DE TEXTO COMPLETO: FULLTEXT SEARCH estos índices no sirven.

Y entramos en el mundo de los índices invertidos, donde Lucene es el REY.
Algunas BBDD también permiten trabajar con índices invertidos, pero son pocas y no ofrecen mucha funcionalidad.
Por ejemplo, Oracle tiene un producto (incluido en la licencia de Oracle Database Standard Edition) llamado Oracle Text que permite hacer búsquedas de texto completo usando índices invertidos.

### Índices invertidos

Lo que hacemos es un procesamiento MUY AGRESIVO de los datos para crear un índice invertido, que nos permita hacer búsquedas de texto completo con un rendimiento aceptable.

Lo que se hace es un procesamiento en fases de los datos quee van llegando:
    Croquetas de jamón
    Croquetas de marisco
    Ensalada de papas
    Gazpacho         
    Papas arrugadas  
    Paella de marisco
    Tortilla de patatas

Un primer paso, sería tokenizar los datos. Separar las palabras (que puede ser algo ya de por si bastante complejo: Espacios, puntuación, paréntesis, etc)

    Croquetas-de-jamón
    Croquetas-de-marisco
    Ensalada-de-papas
    Gazpacho         
    Papas-arrugadas  
    Paella-de-marisco
    Tortilla-de-patatas

Un segundo paso podría ser la normalización de los datos (unificar mayúsculas/minúsculas, eliminar tildes, etc)

    croquetas-de-jamon
    croquetas-de-marisco
    ensalada-de-papas
    gazpacho         
    papas-arrugadas  
    paella-de-marisco
    tortilla-de-patatas

Además, podría hacer una eliminación de palabras que no aportan valor semántico (stop-words)

    croquetas-*-jamon
    croquetas-*-marisco
    ensalada-*-papas
    gazpacho         
    papas-*-arrugadas
    paella-*-marisco
    tortilla-*-patatas  

En algunos casos incluso podemos aplicar técnicas más avanzadas: stemming o lematización (reducir las palabras a su raíz semántica, género, número, etc)

    croquet-*-jamon
    croquet-*-marisc
    ensalad-*-pap
    gazpach         
    pap-*-arrugad
    paell-*-marisc
    tortill-*-patat

Y finalmente creamos el índice invertido:

    croquet -> Fila 1 (Palabra 1), Fila 2 (Palabra 1)
    jamon   -> Fila 1 (Palabra 3)
    marisc  -> Fila 2 (Palabra 3), Fila 3 (Palabra 2)
    ensalad -> Fila 3 (Palabra 1)
    pap     -> Fila 3 (Palabra 3), Fila 4 (Palabra 1)
    gazpach -> Fila 4 (Palabra 1)
    arrugad -> Fila 5 (Palabra 2)
    paell   -> Fila 2 (Palabra 1)
    tortill -> Fila 6 (Palabra 1)
    patat   -> Fila 6 (Palabra 3)

Esto es un índice invertido... y es el proceso que hace Lucene (y por tanto Opensearch/Elasticsearch) para permitir búsquedas de texto completo.

Es un proceso computacionalmente MUY CARO, muchísimo más que crear un índice directo en una BBDD tradicional.
Habitualmente, este tipo de indexaciones las hacemos de forma asíncrona.

Qué hacemos luego cuando se quiere hacer una búsqueda?

    Dame las recetas cuyo nombre contiene la palabra "Con Marisco"

    Aplicamos sobre el/los términos de búsqueda el mismo procesamiento (o parecido) que hicimos en la indexación:
    Paso1: Con-marisco
    Paso2: con-marisco
    Paso3: *-marisco  (con es una stop-word)
    Paso4: *-marisc
    "masic" -> Este token es el que buscamos en el índice invertido.

## Sobre Lucene

Lucene cuando le llegan datos para indexar, los procesa de esa forma que hemos hablado.
Los índices invertidos que crea los guarda en ficheros en disco.
A esos ficheros se les llama "segmentos".

Y OJO! No llevan el mismo tratamiento que en una BBDD tradicional.
En una BBDD tradicional los índices se guardan en un único fichero (o conjunto de ficheros) que se van actualizando conforme llegan nuevas operaciones DML (INSERT/UPDATE/DELETE).... Y se deja mucho hueco libre en las páginas para futuras inserciones (fill factor).

En el caso de Lucene, un índice se guarda en muchos ficheros (segmentos) que son INMUTABLES. 
Esos ficheros no se modifican... (acceso aleatorio en disco es caro)... Les vamos añadiendo cosas al final.

Cuando uno de esos ficheros se hace demasiado grande, Lucene comienza a crear nuevos ficheros (segmentos).

> Dicho esto. Qué debe hacer Lucene cuando se solicita una búsqueda?

Imaginad.. Me llegan hoy 10 documentos/datos para indexar con la palabra marisco.
Lucene crea un segmento (fichero) con esos 10 documentos.... y otros 800.000

Mañana llegan datos nuevos con la palabra marisco... Lucene crea un nuevo segmento (fichero) con esos nuevos datos... y otros 1.200.000

Esto implica que mientars que en una BBDD tradicional los ficheros contieneen los datos PREORDENADOS, en Lucene los datos en los ficheros (segmentos) están DESORDENADOS.
Por lo tanto, mientras que en una BBDD tradicional para buscar la palabra marisco podemos leer directamente del fichero del índice, en Lucene necesito coger TODOS LOS FICHEROS (segmentos) y unificarlos antes de hacer la búsqueda.

Lucene NO HACE BUSQUEDAS EN DISCO, una BBDD tradicional SI. Lucene usa los ficheros SOLO para persistir los datos.
Cada vez que quiero hacer una búsqueda, tengo que cargar TODOS LOS FICHEROS (segmentos) en memoria y unificarlos para luego hacer la búsqueda.

Esto tiene una implicación enorme... Más vale que ese índice YA ESTE EN MEMORIA cuando quiera hacer la búsqueda... porque si no, la latencia de la búsqueda se dispara. (HABLAREMOS DE ESTO... y tendrá su impacto en el diseño de INDICES y SHARDS en Opensearch/Elasticsearch)

UNA NOTA MÁS... Un índice en lucene NO ES LO MISMO que un índice en Opensearch/Elasticsearch.

---

# Los indexadores

Por defecto la misión de un indexador es generar un índice invertido para permitir búsquedas de texto completo.
Qué devuelve esa búsqueda? Una lista de documentos que cumplen la condición -> Los identificadores de los documentos.

Oye.. problema tuyo donde tiene guardados esos documentos.

Google es un indexador de documentos (webs)... Las páginas WEB se guardan en google o están guardadas en los servidores de cada web? En los servidores de cada web. Google NO GUARDA LAS WEBS (bueno... en realidad si... y Opensearch también... por defecto... aunque eso es configurable).

Los indexadores por defecto guardan una copia del documento completo en el índice... al menos tal y como estaba en el momento de la indexación.

Si tengo una WEB de un periódico con noticias (día 19-Enero) y google lo indexa... y genera índice... y guarda una copia de la web... y al día siguiente (20-Enero) hay otras noticias... Esas las tiene google? NO. Ni aparecen en las búsquedas.

Para qué guardan Google u Opensearch una copia del documento completo? Cuál es el objetivo real? FACETADO
En los resultados de las búsquedas, google u Opensearch pueden mostrarte "Facetas" un extracto literal del documento que ha sido indexado.... Cuidado... eso me garantiza que cuando acceda al documento en su ubicación REAL ese documento sigue existiendo? Y va a seguir conteniendo exactamente lo mismo que cuando fue indexado? NO.

El indexador guarda INDICES. Los documentos los guardaré en otro sitio.

Pero... y que pasaría si lo que indexo son DOCUMENTOS MUERTOS, es decir, documentos que sé que no van a cambiar nunca más? La copia que guarda el indexador sería suficiente.
Y en este caso podría usar el indexador como si fuera una BBDD No relacional.

Por cierto... que pasa cuando tengo un sistema de MONITORIZACION? Y leo logs... métricas... Esos datos están vivos? MUERTOS MUERTOS.... La medida de uso de CPU de hace 2 horas va a cambiar? NO.
Ahopra habrá una nueva medida de uso de CPU... pero la de hace 2 horas no va a cambiar.

Para este tipo de datos, usar un indexador como Opensearch/Elasticsearch como si fuera una BBDD No relacional es una opción MUY A TENER EN CUENTA. Además, como me permiten hacer búsquedas de texto completo, puedo hacer consultas muy complejas sobre esos datos, se convierten en herramientas muy potentes para monitorización de sistemas.

Y la realidad es que NO MUCHA GENTE usaba ElasticSearch como un motor de búsqueda de documentos... La mayoría lo usaba como una BBDD No relacional para monitorización de sistemas. En lo que llamamos el stack ELK (Elasticsearch, Logstash, Kibana).

En vuestro caso, utilizaís OpenSearch para las 2 cosas:
- Indexador de documentos (buscador de documentos)
- Monitorización de sistemas
---

Hemos dicho que Lucene ees quien se come todo el tinglao de la indexación y búsqueda de documentos.

# Que pintan entonces Opensearch/Elasticsearch?

## Entornos de producción

Los entornos de producción deben tener al menos 2 características que no comparten con el resto de entornos:
- Alta disponibilidad
  Tratar de garantizar que el servicio esté disponible una determinada cantidad de tiempo pactada previamente (SLA) 
  Tratar de garantizar la no pérdida de datos ante fallos hardware/software
  La tratamos de garantizar mediante redundancia (HW y SW)
- Escalabilidad
  Capacidad de ajustar la infraestructura a las necesidades del negocio en cada momento (crecimiento o decrecimiento)... en el caso de Indexadores: CRECIMIENTO 
    La puedo conseguir mediante Escalabilidad VERTICAL (más máquina) o HORIZONTAL (más máquinas)

Y la forma de conseguir esto es mediante la creación de CLÚSTERES (FISICO y LÓGICO)
                                                                     HW       SW

Pero... es más complejo. ES/OS son herramientas distribuidas. Qué significa eso?

No es solo que tenga un cluster de 10 nodos.. donde TODOS los nodos son IGUALES y hacen el mismo trabajo.
Significa que dentro de ese cluster, cada nodo puedee realizar un trabajo diferente al resto de nodos.
Y es el trabajo coordinado de todos los nodos lo que permite ofrecer la funcionalidad completa del sistema.

Un cluster != Distribuido

Puedo tener un cluster distribuido... o puedo tener un cluster NO distribuido.

    10 apaches/nginx no forman un cluster distribuido de servidores web... forman un cluster NO distribuido de servidores web... que por delante llevan un balanceador de carga.

Y asu vez.. este tema del cluster es lo que aportan (una de las cosas) ES/OS por encima de Lucene.
Lucene es un proceso Java que se ejecuta en un único servidor... Si ese servidor falla... adiós al servicio.
ES/OS orquesta múltiples nodos con Lucene para ofrecer alta disponibilidad y escalabilidad.

---

## Tipos de nodos en ES/OS

Un cluster de ES/OS puede tener muchos nodos... Y podemos (y queremos) que cada nodo tenga un rol diferente.

Un nodo puede adquirir uno o varios roles, de entre:
- Master (hoy en día ya no se llama master... se llama "nodo de coordinación del cluster")
- Data
- Ingest
- Coordinador de consultas (hoy en día ya no se llama client node)
- ... y otros.

Cuál es la misión de cada uno de esos roles.

### Nodos Master

Al menos se exige que existan 3 nodos con este rol en el cluster (puedo tener 1... para jugar).
Por qué pasamos de 1 a 3? Para evitar un problema llamado Brain Split (División cerebral).

    Nodo1 (maestro, data)   datoA. datoB
    Nodo2 (maestro, data)   datoA. datoC
    Nodo3 (maestro, data)   datoB. datoC

Llega el datoA... dónde lo guardo? En uno, en 2 o en los 3? 
 En los 3   I
 En 2       III

Pregunta... para que leches he montado un cluster con 3 nodos data? 
   HA? NO... si lo único que quiero es HA, con 2 nodos me vale.
   Escalabilidad? SI.. para poder acometer más operaciones de lectura y escritura.

Y es doloroso.. muy doloroso. Al multiplicar la infra x 3... puedo llegar a obtener una mejora en rendimiento teórica (será mucho menos en la realidad) del:
- Con un nodo, en una unidad de tiempo guardo un dato. En 2 unidades de tiempo guardo 2 datos.
- Con 3 nodos, en dos unidades de tiempo guardo 3 datos.

Tengo una mejora teórica del 50% en rendimiento al triplicar la infra. (Es una jodienda... me gustaría tener un 300% de mejora... pero no es así)

Volviendo a los master:
Me llega el datoA... lo voy a guardar en 2 nodos... pero solo 1 ha recibido la solicitud. 
Ese nodo debe comunicarse con otro nodo al menos para que guarde el datoA.

Quién le pone el ID al dato? El nodo que recibe la petición? NO... Si eso fuera así, si 2 nodos reciben peticiones en paralelo, podrían asignar el mismo ID a 2 datos diferentes.
Debe haber un responsable de asignar IDs únicos a los datos que llegan al cluster.
Ese es el nodo master. Y REPITO: EL NODO MAESTRO... De los 3 que se exige que tenga un cluster, solo uno de ellos es el NODO MAESTRO ACTIVO. Los otros 2 están en espera por si falla el activo. Dicho de otra forma, son máquinas que estoy tirando a la basura... pero que son necesarias para garantizar la alta disponibilidad del cluster.

    Nodo1* <---
     ^v
    Nodo2  <--- Balanceador de Carga <--- Clientes
     v^
    Nodo3*  <--- 

No es la única responsabilidad de los master... pero si la más importante. Otra responsabildad es determinar en que nodos se van a guardar los datos que llegan al cluster.

Para que un nodo se pueda convertir en master, debe recibir apoyo de al menos la mitad + 1 de los nodos master del cluster (quorum). Por eso vamos siempre con nodos master en número impar (3,5,7...)
El mínimo es 3... pero en clusters grandes se recomienda 5... y me la juego a que se caen 2... y el cluster sigue funcionando.

Como es un chorreo lo de tener nodos que no haceen nada... hay un truco de configuración que puede hacerse... nombrar maestros de mentirijilla. Son maestros pero que no pueden ejercer como tales. Solo dan voto!
Ese rol habitualmente se lo enchufamos a algún nodo de tipo data.

 2 nodos maestros elegibles
 3 nodos data (uno de ellos maestro de mentirijilla)

Los nodos maestros tienen una responsabilidad gigante. Si un nodo maestro no responde, el cluster entero deja de estar operativo. Y ... una mala query puede hacer que el nodo maestro se quede sin recursos y deje de responder, si es que el nodo maestro además de maestro es data.
Los maestros son lo más sagrado del cluster. No quiero joder con ellos... quiero protegerlos.

Una buena instalación de ES/OS tendrá nodos maestros dedicados (al menos 2), que no hagan nada más que ser maestros.
No necesitan tantos recursos como los nodos data... pero si necesitan estabilidad y disponibilidad.

Y son nodos que no voy a exponer nunca al exterior. No van a recibir peticiones de clientes... ni van a indexar datos... ni van a responder consultas.

OS/ES usan 2 puertos para la comunicación entre nodos: Uno para comunicaciones con clientes y otro para comunicaciones internas entre nodos. A los maestros no les mandamos ninguna petición del exterior por el puerto de clientes.

Cuando empiezo con ES/OS suelo empezar con un cluster de 3 nodos que hagan de todo (master, data, ingest, coordinador de consultas). Pero en cuanto el cluster crece un poco... separo los roles.

La siguiente instalación que monto:
   2 nodos maestros dedicados
   2 nodos con el resto de roles (data, ingest, coordinador de consultas... uno de ellos con maestro de mentirijilla)

A partir de ahí ya es monitorización... La monitorización es la que me dirá como debo ir creeciendo el cluster.
Si me interesa más nodos de tipo data.
Si me interesan más nodos de tipo coordinador de consultas.

El siguiente paso más natural es separar los nodos data de los nodos coordinadores de consultas.

    Maestro1    -+-- Data 1
                 |
    Maestro2.   -+-- Data 2
                 |
                 +-- Data 3
                 |
                 +-- Coordinador1 <--+ 
                 |                   | BC.  <- Son los que reciben consultas públicas
                 +-- Coordinador2 <--+ 

Y ahora imaginemos el paseillo y el trabajo que ocurrirá al hacer una búsqueda.

El coordinador1 recibe la petición de búsqueda:
- Dame los documentos que contengan tales cosas... COORDINADOR1
- COORDINADOR1 -> pregunta al maestro activo quien datos que puedan encajar con la búsqueda (INDICES)
- El maestro contesta al coordinador1 diciéndole en que nodos data puede encontrar esos datos
- El coordinador 1 pregunta a los nodos data que le interesan por esos datos: DATA 1, DATA 2
- DATA 1 y DATA 2 responden al coordinador1 con los documentos que cumplen

> Pregunta. 

En los data 1 y data 2, lo que tendremos son Lucenes cargados en memoria con los índices invertidos.
Cuando el coordinador1 les pregunta por los documentos que cumplen, esos nodos data harán la búsqueda en memoria y devolverán los identificadores de los documentos que cumplen la condición... 
Devuelven los datos ordenados? Lucene devuelve datos ordenados... pero los suyos.
Ahora, un trabajo esencial del coordinador1 es unificar los resultados de los nodos data y ORDENARLOS ENTRE SI. Cada subconjunto ha llegado ordenado internamente... pero no entre si.
Qué tal se le da a un ORDENADOR ordenar datos? COMO EL CULO! Es de la peores cosas (desde un punto de vista computacional) que puede hacer un ordenador. Los algoritmos de ordenación son de O(n log n). Y si tengo muchos datos... se me puede disparar la latencia de la consulta.

Los ordenadores (también llamados COMPUTADORES) son horribles ordenando datos. De hecho, en España les llamamos ORDENADORES influenciados por los gabachos (le llaman l'ordinateur' que significa el que procesa ordenes).

Ese trabajo destroza un nodo... sobre todo si la búsqueda devuelve muchos datos.
Si ese trabajo se lo dejo a los maestros.. corro un riesgo enorme de que el maestro deje de responder y el cluster se caiga.
Si ese trabajo se lo dejo a los data... corro el riesgo de que el data deje de responder y no se puedan hacer más consultas ni más indexaciones.

En cuanto mi cluster crece un poco... separo los nodos coordinadores de consultas de los nodos data.

Esto daría lugar a un cluster del tipo:

    2 maestros dedicados + 2/3 nodos data (uno de ellos maestro de mentirijilla) + 2 nodos coordinadores de consultas

    Ese es un cluster de partida serio. A partir de ahí, la monitorización me dirá si necesito más nodos data o más nodos coordinadores de consultas.

---
Lo de los 3 maestros es el mismo concepto que en Kubernetes.
En kubnernetes cuando nodos hacen de master, deben ser al menos 3 para evitar el brain split.
De hecho en Kubernetes, el control plane son muchas hgerramientas:
- etcd <--- Esta es la herramienta (la BBDD) que me exige tener 3 replicas para evitar el brain split
- controller manager
- scheduler
- api server
- core dns
- ...
---

## Índices en ES/OS (que no son el mismo concepto que en Lucene)

En Lucene, un índice es un conjunto de ficheros (segmentos) que contienen un conjunto lógico de datos indexados.
En ES/OS, un índice es un conjunto de lucenes, que guardan documentos relacionados entre si de forma lógica... a decisión de quién diseña el sistema o los índices.

> Quiero crear un índice para guardar documentos de tipo recetas de cocina
Y tengo varios nodos de data... pongamos que tengo 3 nodos data.
Digo yo!... pues ya que tengo 3 nodos.. voy a hacer que los 3 puedan guardar datos de recetas de cocina... y así reparto la carga entre los 3 nodos.
Lo que tendría que hacer entonces es poner 3 lucenes (1 en cada nodo data) para guardar los documentos de recetas de cocina.

    INDICE recetas = LUCENE1 (nodo data1) + LUCENE2 (nodo data2) + LUCENE3 (nodo data3)

    Cada uno de esos lucenes guarda un conjunto de recetas de cocina diferentes a los de los otros lucenes.
    Es decir, las recetas guardadas en el lucene1 no están en el lucene2 ni en el lucene3.

    Cada uno de esos lucenes en OS/ES se llama SHARD (fragmento)
        SHARD = LUCENE
    Y un índice en OS/ES está compuesto por un determinado número de SHARDS PRIMARIOS, que son esos que hemos comentado ahí arriba.

    Adicionalmente, para garantizar la alta disponibilidad, puedo crear COPIAS de esos SHARDS PRIMARIOS. Es decir más Lucenes que guardan los mismos datos que los shards primarios.
    Esos lucenes(o shards) copias se llaman REPLICAS.

    En un momento dado del tiempo de cada shar primario puedo tener 0, 1 o más replicas. De entre todas las copias de un shard sólo una de ellas está activa / primaria. Las demás (réplicas) están en espera por si falla la activa.

    El dividir un índice en muchos shards primarios me aporta? ESCALABILIDAD... Pero ojo!
    Lo que tengo es ESCALABILIDAD DE ESCRITURA... Puedo hacer más operaciones de indexación en paralelo.
    La escalabilidad de LECTURA es más compleja... mucho más compleja.
    El tener más shards primarios me hace poder recuperar los datos más rápido?
    No necesariamente. Si voy a reecuperar 1 documento, si... pero si voy a recuperar muchos documentos... Al tener más primarios donde tener que ir a buscarlos se va a incrementar el tiempo de CONSOLIDACION de los resultados (el que hacía el nodo coordinador de consultas).

    Otra cosa con la que hay que tener especial cuidado es con la forma de determinar en qué shard primario se van a guardar los documentos que llegan al índice. Eso es lo que en OS/ES se llama ROUTING.

    Por defecto, un documento tiene un ID. El cliente puede mandar ese ID o OS/ES lo generará automáticamente caso que el cliente no lo mande. Y OS/ES usa ese ID para determinar en qué shard primario se va a guardar el documento. Se aplica un algoritmo que trata de repartir uniformemente los documentos entre los shards primarios.
    OJO! No es un round robin...

    Qué hace ES:
    Lo que hace es aplicar un algoritmo de HUELLA (HASH) al id del documento: "FEDERICO" 

    hash("FEDERICO") -> Número entero

    See aplica lo mismo que para el DNI. Se divide ese número entero entre el número de shards primarios del índice... y el resto de esa división es el shard primario donde se guardará el documento.

    Ese resto puede estar entre qué valores? 0-(N-1) siendo N el número de shards primarios del índice.
    Los shards de un índice se denominan: shard0, shard1, shard2, ..., shardN-1

    Los nodos maestros llevan un control de en que nodo data está cada shard primario y sus réplicas.
        shard0/indiceA -> nodo data1
        shard1/indiceA -> nodo data2
        shard2/indiceA -> nodo data3
        shard0(copia1)/indiceA -> nodo data2
        shard1(copia1)/indiceA -> nodo data3
        shard2(copia1)/indiceA -> nodo data1
    
    Si los IDs que me llegan (o que genero) son aleatorios... perfecto... los documentos se repartirán uniformemente entre los shards primarios... crucemos los dedos!
    En caso que no se haga un reparto equilibrado de los documentos entre los shards primarios... puedo tener problemas de rendimiento. Puede haber un shard con un 10% de los documentos y otro shard con un 90% de los documentos.
    Y cuando hago una búsqueda , el del 90% tarda mucho más en responder que el del 10%.
    Y el nodo con el shard del 10% le tengo aburrió que acabó hace mucho... y el usuario como un pendejo esperando a que le llegue la respuesta. Mientras que si tuvieran un reparto más equilibrado, ambos shards acabarían más o menos a la vez.

    Lo que puedo hacer es definir un campo de routing personalizado.
    Por ejemplo, en el caso de las recetas de cocina, puedo definir que el campo de routing sea el campo "Tipo de comida" (Italiana, Mexicana, Española, etc)
    
    Al usar este campo como campo de routing,me aseguro que todas las recetas de comida italiana van a un shard primario... todas las recetas de comida mexicana a otro shard primario... etc.

    Si luego se hacen muchas búsquedas por tipo de comida... ideal... ya que solo tengo que ir a un shard primario... y me quito el trabajo de coordinación de resultados entre shards primarios.

    Y esto puede dar lugar a desequilibrios enormes entre shards primarios.
    Me puede interesar o no... depende del caso de uso.
    Depende si hay más búsquedas o más indexaciones. Y de si las búsquedas son por el campo de routing o no.
    Y la respuesta a esto SOLO LA OFRECE MONITORIZACION... NO HAY RECETAS MAGICAS en OPENSEARCH/ELASTICSEARCH.

    Esto es algo importante a la hora de definir los índices en OS/ES: Determinar la forma óptima de hacer el routing de los documentos entre los shards primarios. Dicho esto.. o lo tengo muy claro... o lo dejo por defecto (ID del documento).

> El tener más réplicas... qué aporta? Las réplicas me aportan ALTA DISPONIBILIDAD y ESCALABILIDAD DE LECTURA.
  Al tener más réplicas, puedo leer de más sitios en paralelo -> Más operaciones de lectura en paralelo = Meejor rendimiento en lecturas.

NOTAS IMPORTANTES:
- El número de réplicas se puede cambiar en caliente (en producción)... a lo que me dé la gana.
- El número de shards primarios NO se puede cambiar en caliente... Es un dato que se define al crear el índice y ya no se puede cambiar. Al menos a un valor arbitrario de mi interés...
  Lo unico que podré hacer a posteriori es:
  - Dividir cada shard primario en 2 shards primarios (split)
  - Juntar varios shards primarios en 1 shard primario (shrink)
  - Reindexar el índice en uno nuevo con el número de shards primarios que me interese. <- MUY PESADO

Teniendo 4 shards primarios, puedo pasar a 2 (shrink) o a 8 (split). Pero no puedo pasar a 5 o 6 shards primarios.

# Distribución de shards primarios y réplicas en los nodos data

Núnca puedo tener más replicas de un shard que nodos en el cluster. 
ES/OS no van a ubicar 2 copias del mismo shard en el mismo nodo data (considera que es inutil = single point of failure)
Lo que si puedo tener es más shards primarios que nodos data.

Tengo 3 datas.
> Creo un índice que tiene 5 primarios, con 1 réplica.
Cómo se distribuirán los shards primarios y sus réplicas entre los nodos data?

    Data1 shard0 shard3    shard1' shard2'
    Data2 shard1 shard4    shard0'
    Data3 shard2           shard3' shard4'

> Si creo un índice con 2 primarios y 3 réplicas de cada uno:

    Data1 shard0 shard1'
    Data2 shard1 shard0'
    Data3 shard1' shard0'

    Pero habría otra réplica de cada shard que no se podría ubicar en ningún nodo data... por lo tanto ES/OS SI me dejaría crear ese índice... pero consideraría que el cluster no está sano (yellow state): UNHEALTHY

1 primario y 2 réplicas:

    Data1 shard0
    Data2 shard0'
    Data3 shard0''

    Al hacer la búsqueda la podría hacer en cualquier de los 3 nodos data.
    No va a ocurrir que parte de los datos se saquen de un nodo y parte de otro...
    Lo que si ocurrirá es que mientras un nodo está ocupado, tengo otros 2 nodos libres que pueden atender otras peticiones sobre el mismo shard... y tengo más throughput de lectura.

---

> Tengo 3 nodos data. Y creo un índice con 3 shards primarios y 1 réplica de cada uno de ellos.

    Data1 shard0*    shard1'
    Data2 shard1*    shard2'
    Data3 shard2*    shard0'

Imaginad que ahora, el nodo Data2 se cae!

    Data1 shard0*    shard1*
    Data3 shard2*    shard0'

  Los shards que tenía ese nodo quedan sin ubicación: shard1    shard2'
    El cluster pasa a estar en estado YELLOW (UNHEALTHY)
  Qué decisión va a tomar ES/OS?
  - A priori esa reasignación la hace automáticamente pasado un tiempo de gracia (configurable).

        Data1 shard0*    shard1*     +shard2'  <-Copia desde el shard2* (que está en el data3)
        Data3 shard2*    shard0'     +shard1'  <-Copia desde el shard1* (que está en el data1)

    ESTO ES ALGO A EVITAR A TODA COSTA EN PRODUCCIÓN.... más aún si uso kubernetes!
    Pero en general SIEMPRE !
    No queremos esa reasignación!

    El problema es otro... Si se me cae un nodo data... y me pongo en ese momento a reasignar shards primarios y réplicas.... lo que básicamente se traduce en copiar datos de unos nodos a otros:
    - Cuando me pongo a copiar el shard2* del nodo data3 al nodo data1... y ese shard tiene 10Gbs de datos... 
      SE ME CAE EL CHIRINGUITO ENTERO... Cluster DEAD ... RIP Se murió!

## Índices.

Qué índices creo? Cuántos índices me interesa tener en un cluster?

Podríamos pensar que si guardo recetas me interesa tener un solo índice "recetas" donde guardar todas las recetas.

Pero que otras alternativas tengo?
- Podría crear índices diferentes en base a algún campo de las recetas... por ejemplo "recetas_italianas", "recetas_mexicanas", "recetas_españolas", etc.
- Podría crear índices diferentes en base a la fecha de creación de las recetas... por ejemplo "recetas_2023", "recetas_2024", etc.

Una gracia que tienee ES/OS es que podemos hacer búsquedas que usen varios índices a la vez... de forma muy sencilla:
ALIAS

Todo índice tiene un nombre único en el cluster.
Pero un índice además puede tener uno o varios ALIAS (nombres alternativos).
Y es más, esos ALIAS no tienen porque ser únicos. Puedo tener varios índices con el mismo alias.

    Índice: recetas_italianas   Alias: recetas
    Índice: recetas_mexicanas   Alias: recetas
    Índice: recetas_españolas   Alias: recetas

Si hago una búsqueda sobre el alias "recetas", ES/OS buscará en los 3 índices a la vez.

La decisión de cúantos índices voy a crear está condicionada principalmente por?
EL MANTENIMIENTO DE LOS INDICES!

Los índices requieren de mantenimiento periódico:
- Optimización de índices (merge de segmentos en Lucene)
- A veces quiero congelar un índice (read only) para que no se le pueda escribir más datos
- A veces quiero cerrar un índice (offline) para que no se use ya en búsquedas 

Estas cosas son las que van a condicionar la forma en la que voy a crear los índices.

Si tengo por ejemplo datos de logs de servidores... lo más normal es crear un índice por un determinado rango de fechas (diario, semanal, mensual, etc)... en base a la cantidad de datos que genere.

Sé que una vez que ha pasado un mes, esos datos ya no se van a modificar más... por lo tanto puedo congelar ese índice y reorganizarlo para optimizarlo para lecturas.
Quizás no me interesa tener esos datos en las búsqedas habituales cuando pasan 6 meses... por lo tanto puedo cerrar ese índice y dejarlo offline.
Quizás si quiero poder seguir hacien búsquedas.. pero ya no me interesa el mismo detalle en los datos... por lo tanto puedo hacer un reindexado de ese índice en otro nuevo con menos detalle (menos campos, menos precisión en los datos numéricos, etc).

    Indice:
    datos_enero_2024
    datos_febrero_2024
    datos_marzo_2024
    ... 
    datos_diciembre_2024

    alias datos_de_los_ultimos_12_meses -> apunta a los 12 índices anteriores

    datos_enero_2025  -> le meto el alias datos_de_los_ultimos_12_meses
    datos_enero_2024 -> le quito el alias datos_de_los_ultimos_12_meses y lo cierro
    Y en cuanto creo el índice de febrero 2024, el índice de enero_2024 lo congelo.. y lo reorganizo... y le hago un shrink... y lo dejo listo para lecturas.
    Y a lo mejor a los 6 meses hay campos que ya no me interesan... y defino políticas de ciclo de vida de los índices (ILM) para que se reindexen automáticamente en índices nuevos con menos detalle.


Estas cosas son las que tengo que tener en cuenta principalmente a la hora de definir los índices en ES/OS.

Hay otras:
- En un sistema multitenant, quizás me interese tener un índice por cada cliente (por seguridad)
---

# Instalación es de ES/OS en kubernetes

Kubernetes es una herramienta (TOTALMENTE DIFERENTEE A ALGO COMO DOCKER COMPOSE) que lo que nos permite es definir y operar (en automático) entornos de producción mediante un lenguaje declarativo (basados en contenedores).

Básicamente es lo antes hacíamos con el VMware vSphere (o Hyper-V, o Xen, o KVM, etc) pero con contenedores en lugar de máquinas virtuales. Y usando un lenguaje declarativo en lugar de interfaces gráficas / lenguaje imperativo.

Los conceptos en kubernetes son los de entornos de producción:
- Balanceadores de carga        Service
- Entradas DNS
- Volumenes                     Persistent Volume
- Reglas de firewall            Network Policies
- Proxies reversos              Ingress Controller (Ingress = Regla de configuración del proxy reverso)

Una gracia de Kubernetes es que al operar con contenedores, los contenedores son muy muy mucho más 

## Contenedor?

Es un entorno aislado (dentro de un kernel Linux) donde ejecutar procesos.
Me permite ejecutar procesos de forma aislada del resto de procesos que corren en el host.. cosa que también puedo conseguir al ejecutar máquinas virtuales.
Pero los contenedores usan una estrategia totalmente diferente. En un contenedor NO PUEDO POR DEFINICIÓN instalar un SO. Los contenedores usan el Kernel de SO del host. Mientras que las MV llevan su propio kernel de SO independiente del host.

Eso hace que las MV sean mucho más pesadas que los contenedores (Gbs vs Mbs o Kbs)

Cuando trabajo con un cluster de kubernetes. y quiero HA, lo que monto son varios pods (réplicas) con el/los mismos contenedores. Y monto un balanceador de carga por delante que reparte las peticiones entre los pods (service).

Qué pasa si un nodo se cae que tenía uno de los pods?
Kubernetes lo mueve a otro nodo automáticamente. Relmente esto es un eufemismo... Lo que hace es crear un nuevo pod en otro nodo... y el pod que estaba en el nodo caído se pierde.

Como hace para que ese nuevo pod mantenga la personalidad (datos, configuraciones...) del pod que se ha perdido?
Montándole el mismo volumen persistente (Persistent Volume) que tenía el pod que se ha perdido.

Esto es lo que en kubernetes podemos configurar mediante un StatefulSet (OS/ES es un StatefulSet no son Deployment).

Los datos siempre van FUERA del pod/contenedor.
Hay posibilidad de que se pierdan los datos? MUY REMOTA... MUY MUY REMOTA. Los datos los tendré en una o varias cabinas de almacenamiento con redundancia (RAID, replicación, etc).

Lo que si es más fácil es que el programa (OS/ES) se quede colgao!
Y en ese caso, mato el proceso y kubernetes lo vuelve a levantar en el mismo nodo (o en otro nodo si quiero).
O si se cae el nodo, kubernetes lo vuelve a levantar en otro nodo.
Pero le enchufo los mismos volúmenes persistentes y el programa sigue funcionando como si nada.

No quiero que por caersee un nodo, el OS/ES se ponga a reubicar shards primarios y réplicas... que es lo que ocurriría si un nodo data se cae en una instalación tradicional (si además tengo guardados los datos en local en cada nodo data). De lo contrario es algo que NO QUIERO QUE OCURRA... bajo ningún concepto.
Lo único que quiero es levantar un nuevo nodo de OS DATA sustituto en otro nodo que se ha jodido.

---

# Nota general sobre TODO lo que os voy a contar en el curso.

JAVA es desastroso en su gestión de la memoria RAM... Pregunta eso es malo? NO... es un feature de JAVA (una característica)

Cuando JAVA surge, se decide hacer un lenguaje que gestione mal la RAM (es intencionado). Para que íbamos a crear un lenguaje con esa sintaxis y que gestionase bien la RAM.. ya exisía es lenguaje. Se llama C++.

HAcer un programa en C++... con una buena gestión de RAM es MUY COMPLEJO. 
Y el objetivo de JAVA es hacer un lenguaje sencillo de usar... y que gestione la RAM por ti.... Metiendo al RECOLECTOR DE BASURA (Garbage Collector) que se encarga de liberar la RAM que ya no se usa.

Java genera un montón de mierda en RAM (Basura = Garbage) y de vez en cuando el Garbage Collector se pone a trabajar para liberar esa RAM. 

Al crear un software me pregunto.. lo hago en C++ o lo hago en JAVA?
Y digo:
- Si lo hago en C++, necesito 300 horas de desarrollor con conocimientos avanzados de programación en C++ para hacer un software que gestione bien la RAM. Desarrollador que me sale a 60€/hora = 18.000€
- Si lo hago en JAVA, necesito 200 horas de desarrollo con conocimientos medios de programación en JAVA para hacer un software que gestione mal la RAM. Desarrollador que me sale a 50€/hora = 10.000€
Diferencia: 18.000 - 10.000 = 8.000€

Cuanto cuestan 2 pastillas de RAM para el servidor= 1500€
Esta claro: Lo hago en JAVA.

Y voy a fuerza bruta! MAS RAM!

Este mismo concepto aplica en los CLOUDS.
Antes tenía a un DBA que optimizaba mi BBDD.. la dejaba finita, e iba como un tiro.
Hoy en día le pido al AWS que me ponga una BBDD y la gestiones... y me quito del DBA.
Y va a ir tan finita como cuando la gestionaba el DBA? NO... necesitaré más infra... pero me sale más barato que el DBA.

Fuerza bruta!

Y un poco lo mismo nos para con el OS/ES.
Tenemos formas de optimizar mucho el uso de RAM/CPU/Almacenamiento... pero a costa de horas!
Y en muchos casos, sale más barato tirar de fuerza bruta (más RAM/CPU/Almacenamiento) que optimizar.
Sube RAM y no jodas... que como tengamos que empeezar a meterle mano a ver de dondee ahorro, me llevo 1 mes... Y sale más caro!
El problema en las organizaciones es que en un caso la cuenta la paga sistemas y en otro la paga desarrollo.
Y nadie quiere pagar la cuenta. Y aquí estáis de los 2 bandos.
PERO MAGIA NO HAY!

ES/OS Tragan... y se degradan... y si no afino con las configuraciones... se degradan más rápido.
Y hay un problema adicional. El día 1 no voy a dar con la tecla.. por mucho que me empeñe... Es imposible por mucha experiencia que tenga.
SOLO LA MONITORIZACION ME DARÁ LA RESPUESTA.... Respuesta que cambiará con el tiempo... porque los datos y las consultas cambian con el tiempo.
Y necesitaré ir acomodiando la configuración del cluster con el tiempo.

Y una de dos:
- O tengo un equipo de sistemas que se encargue de monitorizar y ajustar la configuración del cluster (ADMINISTRANDO EL ES/OS = DBA DE ES/OS)
- O subo recursos y no me preocupo de nada... y si se degrada el rendimiento, subo más recursos.

---

### Algoritmo de huella (hash)?

En el DNI, la letra es una huella (hash) del número.
Si tengo el DNI 23.000.000 y quiero saber la letra que le corresponde:

    23.000.000 | 23
               +------------
             0   1.000.000 <- Pero esto me da igual
             ^
             Este resto es lo que nos interesa.. que puede estar entre 0-22
    
    El ministerio publica una tabla con las letras que corresponden a cada resto.
        0 -> T
        1 -> R
        2 -> W
        ...
        21 -> A
        22 -> E

Un algoritmo de huella es una funcion que:
- Al recibir un dato siempre devuelve el mismo resultado
- Desde el resultado es imposible deducir el dato original (se considera un resumen/huella del dato original)
- Debe haber "poca" probabilidad de que 2 datos diferentes den el mismo resultado (colisión)
  En el caso de los DNIS, la probabilidad de coición es de 1 entre 24 = 4.16%
  Para el ministerio de interior es aceptable... 
  En informática usamos algoritmos de huella con colisiones mucho menores (SHA256, SHA512, etc)
  Estamos hablando de 1 entre billones o más. 
---

### El almacenamiento es caro o barato hoy en día?

En un entorno de producción es LO MAS CARO QUE EXISTE CON DIFERENCIA.

Digo yo... joder... si me compro yo un HDD de 2Tbs en casa para guardar fotos y me soplan solo 60€ en el Mediamarkt
... Ese mismo espacio de almacenamiento en un entorno de producción:
- En casa uso HDD de baja calidad: Western blue. En la empresa ese HDD se multiplica x5-x10 en precio por usar HDD de alta calidad empresarial
- Cada cado lo guardo al menos 3 veces!
   Esos 2Tbs necesito 3 x 2 = 6Tbs x 5o x 10 del precio del HDD casero
- Y ahora backups x 3
Y resulta que esos 2Tbs en la empresa se convierten en 5.000-10.000€ con facilidad.
---

# Estados en un cluster de ES/OS

- GREEN: Todo OK: todos los shards primarios y réplicas están asignados a nodos (todos los datos son accesibles y estoy garantizando la alta disponibilidad que se solicitó)
- YELLOW: Algún shard réplica no está asignado a ningún nodo (esto implica que aunque todos los datos son accesibles, no garantizo la alta disponibilidad solicitada)
- RED: Algún shard primario no está asignado a ningún nodo (esto implica que hay datos que no son accesibles)



---
Indexador de documentos
BBDD No relacional
Motor de búsqueda
Analizar datos
Sistema de monitorización