
# Sinónimos en OpenSearch

Una forma es trabajar con listas de sinónimos que configuramos en los settings del índice. Se pueden referenciar conjuntos de sinónimos en los analizadores.

La forma guay de resolver etso en general va por otro lado... similar a como lo hacen los LLM (Deep learning). <- Embedding + vector search.

Cuando le mando un mensaje a ChatGTP, entiende lo que le pregunto?
Pues es jodido el tema... porque de hecho si que nos entienden.
Para entender lo que decimos, lo primero que necesitan sería entender el significado de cada palabra.
Y son capaces de eso? YA VES !
Entienden perfectamente el significado de cada palabra.
La pregunta más bien es: Qué entendemos nosotros por significado de una palabra?

El significado de una palabra viene determinado por otras palabras, y la relación que guarda con ellas.
Los humanos (en ocasiones) además de ssto, vinculamos las palabras a experiencias, emociones, sensaciones, que capturamos con nuestros sensores(sentidos). Los modelos de lenguaje no pueden hacer esa parte, pero si pueden hacer la parte de las relaciones entre palabras.
Y con eso suele ser suficiente.

Significado de Perro <- Animal, Mascota, Fiel, Ladra, Cuatro patas, Peludo, Amigo, Protector, Caza, Comer, Pasear, Jugar, Collar, Correa, Veterinario, Raza, Cachorro, Adulto, Viejo

Igual que los humanos tenemos diccionarios de sinónimos, los modelos de lenguaje también los tienen,
Representan cada palabra como un vector (n-dimensional), donde cada dimensión representa una característica que los humanos a priori no somos capaces de entender... pero ellos si.

Leyendo corpus gigantescos de documentos, son capaces de ver palabras que se usan en contextos similares, y por tanto deducir que esas palabras tienen significados similares.

                Es un ser vivo   Es blando
    perro               1           0.5
    gato                1           0.5
    comer               0.5         0
    piedra              -1         -1
    plastelina          -1          1

De esta forma, dependiendo del diccionario de sinónimos que usemos, la computadora es capaz de entender realmente de lo que le estoy hablando.

Lo primero que hacen, una vez sustituyen las palabras por sus vectores, es hacer un análisis sintáctico y semántico de la frase.
Los LLM que usamos hoy en día (en general todas las redes neuronales que usamos hoy en día) parten de una arquitectura llamada Transformer (2017-Google- Attention is all you need -> Traducción automática de textos: Bert).

             C.P.Nuc. Sujeto   Cópula o atributo (referida al sujeto)
             -----------    ------------
    El perro de mi primo es muy juguetón
    -------------------- ---------------
    Sujeto              Predicado


Muchos modelos de búsqueda hoy en día, lo que hacen es llevar esto al extremo... con los vectores de búsqueda. En lugar de tener vectores a nivel de palabra, generan vectores a nivel de documento o párrafo. Para entender de qué leches habla un documento, generan un vector que representa el significado global del documento.

Hoy en día, en este tipo de herramientas lo que se suelen usar son esos vectores a nivel de documento para hacer búsquedas semánticas.
Lo que hacemos es buscar documentos que tengan un vector similar al vector de la consulta que hace el usuario. (dicho de otra forma, que su resta sea pequeña -> 0).

Esto es lo guay!
El trabajar con listas de sinónimos es una forma muy básica de hacer esto. Y casi siempre insuficiente.

Una cosa es ser capaz de entenderme, y otra, una vez que me han entendido generar una respuesta coherente y veraz.

El nivel de entendimiento de una persona estudiada de 40 años, no es el mismo que el de un niño de 5 años.


---

Tengo 1 índice con 3 shards . Les hacemos una réplica o 2? En un cluster de 3 nodos?
NUNCA 2.
Las réplicas las hago para HA...
Pero en cluster de 3 nodos, cúantos me puedo permitir que caigan? 1.
Siempre quedan 2 nodos activos. Si uno de esos cae, el otro cae. No es maestro... no tiene quorum.. no hay cluster.
Solo tengo el sistema funcionando (HA) cuando al menos hay 2 nodos.
Si hago 1 réplica tengo asegurado que al menos 1 de esos nodos va a tener una copia de los datos.
Si hago 2 réplicas, no me aporta nada en HA, porque si caen 2 nodos, el sistema cae. Y me penaliza un huevo en escritura y en volumen de datos.

---
                Escenario 1           Escenario 2        Escenarios intermedios
Shard1 ACME
                    50%                 10%    
Shard2 Globex
                    50%                 90%

Escenario 1: Ideal... Tengo balanceo en escritura y al hacer lectura solo leo de 1 shard.
Escenario 2: Ruina!   No tengo balanceo en escritura y en lectura tengo que leer un shard gigante en RAM.
Escenarios intermedios.. son los que hay valorar a ver si compensa o no. (Menos de 60-40 me compensa fijo.. ni lo miro!)

---

# Receta

PUT /recetas_nuevo/_doc/1
{
    "nombre": "Tortilla de patatas",
    "ingredientes": [
        {"nombre": "patatas", "cantidad": 4, "unidad": "unidad"},
        {"nombre": "huevos", "cantidad": 6, "unidad": "unidad"},
        {"nombre": "cebolla", "cantidad": 1, "unidad": "unidad"},
        {"nombre": "aceite de oliva", "cantidad": 100, "unidad": "ml"},
        {"nombre": "sal", "cantidad": 1, "unidad": "cucharadita"}
    ],
    "tiempo_preparacion": 45,
    "dificultad": "media"
}

// Que devuelve OS/ES al recuperar el documento:
GET /recetas_nuevo/_doc/1
{
  "_index": "recetas_nuevo",
  "_id": "1",
  "_version": 1,
  "_seq_no": 0,
  "_primary_term": 1,
  "found": true,
  "_source": {
    "nombre": "Tortilla de patatas",
    "ingredientes": [...],
    "tiempo_preparacion": 45,
    "dificultad": "media"
  }
}

Campos:
_index:     Nombre del índice donde está el documento
_id:        ID del documento (le asigné yo... si no lo hubiera hecho OS/ES le asigna uno UUID)
_version    Versión del documento. Se genera en automático cada vez que se actualiza el documento. De forma incremental.
_primary_term:  Número de término primario. Se incrementa cada vez que el shard primario cambia (fallo nodo, reasignación shards, etc)
_found:     Booleano que indica si se ha encontrado el documento o no.
_source:    Contenido original del documento cuando se solicita la indexación.... OJO , ES/OS no usan ESTO para buscar, sino los índices invertidos que genera a partir de este contenido.
_seq_no:    Número de secuencia en el shard que se incrementa por cada operación de escritura (indexación, actualización, borrado) que afecta al shard. Al guardar/indexar cada documento, se guarda en él el número de secuencia que tenía el shard en ese momento. Y el número es incrementado posteriormente.
            Para qué sirve... CONTROLAR CONCURRENCIA OPTIMISTA (Optimistic Concurrency Control - OCC)
            Es decir, si 2 personas intentan simultáneamente actualizar el mismo documento.
            No es mágico.. lo tenemos que explicitar nosotros a la hora de hacer la actualización.
            El dato lo tengo para usarlo... pero necesito decirle a ES/OS que lo use.
            Si no lo digo, ES/OS no lo usa.

De entrada... un tema importante es cómo gestiona internamente ES/OS las actualizaciones de documentos.
Y lo hace de forma ligeramente diferente a lo que ocurre en las bases de datos tradicionales.
En una BBDD tradicional, cuando actualizo un registro, lo que hago es localizar el registro en disco (página/bloque) y modificar los datos "in situ", reemplazando los datos antiguos por los nuevos. En ealgunas ocasiones, si en la página/bloque no hay espacio suficiente para los nuevos datos (Tenía "Hola" y ahora quiero guardar: "Hola mundo y todos los seres de la tierra"), puede que no haya hueco en el bloque y tenga que mover el registro a otro bloque/página del disco donde si que haya espacio suficiente... Realmente no se mueve el registro, sino que se marca el registro antiguo como desactualizado y se crea un nuevo registro en otro bloque/página con los nuevos datos. Este proceso se denomina ROW MIGRATION (migración de fila). En Oracle, podemos usar un atributo al definir una tabla (CRITICO) llamado PCTFREE, que indica el porcentaje de espacio libre que debe quedar en cada bloque/página para futuras actualizaciones. Si ponemos un valor alto, evitamos las migraciones de fila, pero desperdiciamos espacio en disco. Si ponemos un valor bajo, ahorramos espacio en disco, pero podemos tener más migraciones de fila.
Puede ser incluso peor... si parte de los datos si le entran pero parte no, puede optar por hacer un ROW CHAINING (encadenamiento de filas). Es decir, guarda parte de los datos en el bloque/página original y el resto en otro bloque/página, creando un enlace entre ambos. Esto también penaliza el rendimiento, ya que para leer el registro completo, hay que leer varios bloques/páginas del disco.

Si hacemos un volcado de una página de datos de Oracle a disco lo vemos guay... esos punteros y toda la estructura de un bloque de Oracle.

En general, la BBDD trata de preservar el dato en su ubicación original en la medida de lo posible, para evitar tener que andar moviendo datos por el disco.

OS siempre hace ROW MIGRATION. ese concepto.. aunque no se llama así.

Es decir, quien indexa es el Lucene... y los datos los guarda en archivos de segmentos. APPEND ONLY.
Es decir, cuando actualizo un documento, no voy a buscar el segmento donde está el documento original y lo modifico "in situ".
Lo que se hace es guardar la nueva versión del documento como un nuevo documento en un nuevo segmento.
Y en otro ficheo, lucene apunta que el documento original ya no es válido (está obsoleto).

Es decir, cada vez que actualizo un documento el dato se duplica en disco... Si lo actualizo 5 veces... sse quintuplica en disco.
Y luego vienen las búsquedas... y Lucene tiene que andar filtrando los documentos, de acuerdo al archivo de _tombstones (documentos obsoletos) antes de entregar los resultados.

ESTO ES OPENSEARCH = ELASTICSEARCH.
Hay una operación de mantenimiento que lanzamos de vez en cuando que se llama MERGE (fusión de segmentos).
Esto mejora el rendimiento de las búsquedas, porque reduce el número de segmentos que tiene que leer Lucene. Y además, elimina los documentos obsoletos, liberando espacio en disco. Pero evidentemente, no es gratis... consume recursos de CPU, memoria y disco. Por eso no se hace en tiempo real, sino que se programa para que se ejecute en momentos de baja carga del sistema. 

Si opto por congelar un índice, puedo aprovechar ese proceso de MERGE para hacerlo... incluso para hacer el shrinking (reducir el número de shards) del índice antes de congelarlo.

## Lo del seq_no.

Cuando voy a actualizar un documento, miro que versión tengo del documento (número de secuencia).

Es decir, flujo:

    Un usuario hace clic en "editar" en un documento.
    - Cargo sus datos en RAM (y me anoto el seq_no que tiene ese documento en ese momento, así como su _version)
    - Muestro el resto de datos en pantalla para que el usuario los edite.
    - El usuario edita los datos y hace clic en "guardar".
    - Mando al backend los datos editados, junto con el seq_no y la _version que tenía el documento cuando lo cargué.
    - Con eso compongo un nuevo documento para mandar a OS/ES.
    - Pero antes de mandarlo, le digo a OS/ES: "Oye, actualiza este documento, pero solo si su seq_no es igual a X (el que tenía cuando lo cargué) y si la _version es igual a Y (la que tenía cuando lo cargué)".
      Porque si alguno de esos datos ha cambiado, es que otro usuario ha actualizado el documento en OS mientras yo lo editaba. Y podría pisarle sus datos.
    - Con la version de ES pasa como con el ID... ES/OS la gestiona en automático, siempre que yo no la toque... que puedo tocarla.
      Alguien podría cargar una nueva version del doucmento, pero con la misma _version 
      En general no es problema, ya que en cualquier caso se habría incrementado el seq_no.

      POST /recetas_nuevo/_update/1
      {
          "doc": {
              "tiempo_preparacion": 50
          },
          "if_seq_no": 0,
          "if_primary_term": 1
      }

      También lo podemos hacer en el requestParams:
        POST /recetas_nuevo/_update/1?if_seq_no=0&if_primary_term=1
        {
            "doc": {
                "tiempo_preparacion": 50
            }
        }


RESUMIENDO: Actualizar un documento = Marcar un documento como obsoleto + crear un nuevo documento con los datos actualizados.

Si tengo un único índice... como os imaginaís... esto se puede convertir en un caos.
Según pasa el tiempo, me voy a encontrar que la mitad o más de los documentos del índice están obsoletos.

Si tengo un sistema que tiene periodos con menos carga (por la noche, fines de semana, etc), y tengo solo un índice puedo programar el proceso de MERGE para que se ejecute en esos momentos. Y backups... y hago backups de todo/completo.
Y tardará más... pero también tiene su punto bueno:  RESTORE. Solo hago restore de un índice completo.

Distinto es si tengo un sistema con mucha carga, y no tengo momentos de baja carga.
Y me puede interesar tener varios índices "rotativos" (por ejemplo uno por día, semana, mes, etc).
Y voy cerrando índices antiguos y abriendo nuevos.
Y qué pasa con las búsquedas? cuando hay datos que aparecen con distintas versiones en distintos índices? Tengo que filtarlos. Desde OS lo podemos hacer con un collapse

    GET /recetas_nuevo/_search
    {
        "query": {
            "match_all": {}
        },
        "collapse": {
            "field": "_id",
            "size": 1",
            "sort": { "_fecha": "desc" }
        }
    }

    Me da resultado preciso... pero me penaliza el rendimiento de la búsqueda. DECISIONES. Es simple.

    Esto no hace que los índices ocupen menos espacio en disco... 


---

    Tengo la tabla Empresas < Telefonos     (1-N) Es el mismo ejemplo de los ingredientes de las recetas. No los tenemos normalizados.
    Tengo la tabla Facturas - Estado        (1-1)
    Tengo la tabla Empresas x Direcciones   (N-M)

Por qué no usar Object o Nested en un único índice?


---

# Receta + Ingredientes

Receta la guardo en un índice RECETAS
Los ingredientes en otro índice INGREDIENTES
En el índice INGREDIENTES, en los documentos meto una relación con la receta a la que pertenecen (recipe_id)

{
    "nombre": "Tortilla de patatas",
    "recipe_id": 1,
    "_version": 5,
    "ingredientes": [
        {"nombre": "patatas", "cantidad": 4, "unidad": "unidad"},
        {"nombre": "huevos", "cantidad": 6, "unidad": "unidad"},
        {"nombre": "cebolla", "cantidad": 1, "unidad": "unidad"},
        {"nombre": "aceite de oliva", "cantidad": 100, "unidad": "ml"},
        {"nombre": "sal", "cantidad": 1, "unidad": "cucharadita"}
    ],
    "tiempo_preparacion": 45,
    "dificultad": "media"
}


{
    "nombre": "Tortilla de patatas mejorada",
    "recipe_id": 1,
    "parent_id": 1,
    "ingredientes": [
        {"nombre": "sal del himalaya", "cantidad": 1, "unidad": "cucharadita"}
    ],
    "tiempo_preparacion": 45,
    "dificultad": "media"
}

Dame recetas de tortilla, mezclando ingredientes de una y de sus referencias.

---


# Versiones a la hora de instalar OpenSearch que necesitamos tener en cuenta:

- Una cosa es la versión del motor de OpenSearch (por ejemplo 3.3.4)
- Otra cosa es la plantilla de operador (chart de helm) que usemos para desplegar OpenSearch en Kubernetes (por ejemplo 17.0.1)
  Esa plantilla se habrá diseñado para desplegar un operador que genere clusters de OpenSearch de una versión concreta (por ejemplo 3.3.4)
  Igual que el chart de helm 17.0.0 que también despliega OpenSearch 3.3.4
- Otra cosa es la versión de mi despliegue concreto de OpenSearch (y del Operador)
  Puedo tener el despliegue v   1 de mi cluster, que usa la plantilla de operador 17.0.1, que a su vez despliega OpenSearch 3.3.4
  Y mañana desplegar la versión 2 de mi cluster, que usa la plantilla de operador 17.0.1, que a su vez despliega OpenSearch 3.3.4
                             ^^^^^^^^^^
                             Esta es la que gestiona HELM por mi, al usar el chart de despliegue que trabaja contra el operador de OpenSearch.
Para mi, la versión de mi despliegue es mi fichero values.yaml + la versión del chart de helm que use (17.0.1)

---

# Service account en Kubernetes

Respresenta un usuario "programa" (CUENTA DE SERVICIO) que puede conectarse con el api-server de kubernetes para hacer operaciones.
Tiene su nombre y un token de acceso (contraseña). Y a ese service account se le asigna roles (Role o ClusterRole) que le dan permisos para hacer operaciones en el cluster de kubernetes.

Hay programas que instalamos dentro del cluster que a su vez hablan con el cluster, para hacer operaciones dentro de él!
Como el operador de OpenSearch.
Ese programa corre dentro del cluster, pero a su vez, necesita hablar con el api-server de kubernetes para hacer operaciones (crear pods, servicios, etc).

Por contra, hay muchas aplicaciones de dentro de cluster que no necesitan hablar con el api-server de kubernetes.
Monto un NGINX para servir una web... no necesito que hable con el api-server de kubernetes.

Es algo que puedo configurar a nivel de POD / Container.

Kubernetes solo exige que los programas que corren dentro del cluster y que necesitan hablar con el api-server de kubernetes, tengan definido un service account.
Openshift es diferente en este sentido (y OKD igual). En Openshift, TODOS los pods que corren dentro del cluster tienen asociado un service account, aunque no lo usen.

---

Hay 2 tipos de tags en imágenes de contenedores:
- Fijos     3.3.4
- Variables latest , 3.4 , 3
Variables, significa que en un momento dado, puede apuntar a una versión concreta, pero en el futuro puede apuntar a otra versión diferente.

            HOY         MAÑANA
    latest  3.3.4       4.7.0 < Este ni de coña! Puede subir a cualquier versión mayor y romper compatibilidades
    3       3.3.4       3.4.1 < Este puede traer nuevas funcionalidades que no necesitamos (no las estamos usando) con nuevos bugs
    3.3     3.3.4       3.3.5 < ESTE ES GUAY . Me trae la funcionalidad que necesito, pero siempre con la mayor cantidad posible de bugs arreglados: PATCH
    3.3.4   3.3.4       3.3.4 < Este es muy conservador... y no está mal

Esto va en conjunto con el imagePullPolicy del pod/container:
- Never: Nunca descarga la imagen del repositorio. Asumo que ya está descargada en el nodo.
- IfNotPresent: Descarga la imagen solo si no está ya descargada en el nodo.
    si tengo el tag: 3.3 y pongo IfNotPresent, si ya tengo un 3.3.4 descargado, no descarga el 3.3.5 nuevo. Ya que ya tengo un 3.3 en el nodo.
- Always: Siempre descarga la imagen del repositorio, aunque ya esté descargada en el nodo
   Siempre que tenga que recrear el pod (escala, nodo cae, etc) descarga la última versión del tag que tenga definido, aunque ya la tenga (útil para tags variables)... aunque cuidado... Si tengo que descargar imagen, eso tarda un rato (quizás solo 20 segundos... pero es un rato). Durante ese tiempo el pod no está disponible.
   Si monto un cluster ACTIVO/PASIVO si es un problema.

Soporta Kubernetes clusters Activo/Pasivo? En kubernetes es básicamente replicas: 1
---

Los  passworsd nunca van en texto plano en los values.yaml. Todo chart que se precie debe permitierme trabajar con secretos de kubernetes.
Y esos secrets no los genero con archivos de manifiesto, los creo por consola / comando (que luego se automatiza vía script/pipeline).

$ kubectl create secret generic opensearch-usuario --from-literal=password=F0rmac1on2025% --from-literal=username=admin -n opensearch