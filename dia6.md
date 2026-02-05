

# Búsquedas:

Tengo 3 shards (3 lucenes)
Hago una busquedsa. Primero lucene me da 5, segundo 6 y tercero 4. Entonces tengo 15 resultados. 
El coordinador debe ordenar esos 15 resultados.

Si quiero solo 10 resultados, los 10 primeros, no necesito ordenar los 15 resultados, sino solo los 10 primeros. Entonces el coordinador solo ordena los 10 primeros resultados de cada shard. Lo que hace el coordinador es ir leyendo los resultados de cada shard, y va guardando los 10 mejores resultados. Cuando ha llegado a 10, deja de leer resultados. Entonces el coordinador solo ha ordenado 10 resultados, no los 15 resultados haían salido de los shards.

Son procedimientos optimizados que usa para evitar ordenaciones grandes: TOP_RANKING.

Si busco en la página tropecientos... el coordinador debe ordenar todo ... hasta llegar a esa página tropecientos... y los algoritmos de srot son nlogn.... 
A más lejos que sea la página que quiero buscar, más resultados tengo que ordenar., y mucho más va a tardar.. el tiempo no es siquiera lineal.

# Esto es si fuera lineal el tiempo, pero es peor que lineal, es nlogn. Entonces a más lejos que esté la página que quiero buscar, más resultados tengo que ordenar, y mucho más va a tardar.
Página 1: 1 segundo.  
Página 2: 2 segundos
Página 3: 3 segundos

En la realidad:
Página 1: 1 segundo
Página 2: 3 segundos
Página 3: 10 segundos

---

# Sinónimos

Hay 2 formas de usar sinónimos:
- Al indexar: el sinónimo se expande al indexar, entonces el documento se indexa con el término original y con el sinónimo. Esto hace que la búsqueda sea más rápida, pero el índice es más grande.
- Al buscar: el sinónimo se expande al buscar, entonces la consulta se expande con el término original y con el sinónimo. Esto hace que la búsqueda sea más lenta, pero el índice es más pequeño. Y no solo eso. Este es guay porque me permite añadir sinónimos nuevos sin tener que reindexar. En el caso de los sinónimos al indexar, si quiero añadir un nuevo sinónimo, tengo que reindexar todo el índice. Como no solemos tener 800 sinónimos de una palabra, pues tampoco empeora mucho el rendimiento.

Sinónimos : patata, papa

Receta 1: Tortilla de patatas

El índice con sinónimos sería: tortilla(1), patata(3), papa(3)
El índice sin sinónimos sería: tortilla(1), patata(3)

Al buscar, si me dicem "papa":
- Si no aplico sinónimos, porque ya los apliqué en el índice, entonces el resultado es la receta 1.
- Si no aplique los sinonimos en el índice, entonces debo aplicarlos en la búsqueda...
  Y buscar en el indice no solo "papa" sino también "patata". Entonces el resultado es la receta 1.
  Si no lo hago, entonces no me va a salir ningún resultado.

Es la diferencia entre los tipos de datos: synonym y synonym_graph. 
- El tipo de dato synonym es para sinónimos al indexar
- El tipo de dato synonym_graph es para sinónimos al buscar


El uso de sinónimos tiene su sitio de aplicación... que no es cualquiera.

Si quiero buscar videos: "persona nadando en un lago"
Y hay un video con       "Felipe flotando en el mar"
Con sinónimos eso no lo saca en la vida!
En cambio con búsquedas vectoriales, sí que lo saca. Porque el vector de "persona nadando en un lago" es muy parecido al vector de "Felipe flotando en el mar". Entonces con búsquedas vectoriales sí que lo saca. Esas dos frases, semanticamente son muy parecidas, aunque no tengan ni una sola palabra en común. 

Tengo otro escenario: Noticias de "Emiliano García Page", también conocido como "El presi"... esto búsquedas vectoriales.. ni de coña! En cambio sinónimos es ideal!

De nuevo, un concepto que ya hablamos: Dependiendo del caso de uso y de la funcionaldiad que quiero conseguir, y de cómo los usuarios van a usar mi sistema, así los índices / mappinmgs / analizers que voy a usar. 

         dormir
   gato  comer  
el perro ladrar    -> Embeddings (ser vivo, accion, agua, temperatura)



Word2vec y técnicas similares son el inicio. Nos ayudaban a codificar palabras en vectores, Es decir, a crear un diccionario semantico de palabras, donde cada palabra se codificaba en un vector.

Esto, hoy en día se hace a nivel de frase, a nivel de párrafo, a nivel de documento... redes neuronales profundas basadas en ese modelo transformador.


Opensearch usa luego un algoritmo de búsqueda k-NN para buscar los vectores más parecidos a un vector de consulta.

K-NN = K-Nearest Neighbors. Es un algoritmo de búsqueda que busca los k vecinos más cercanos a un vector de consulta.
No es solo resta de vectores, hay dimensiones que pueden pesar más que otras.
---

OCR. Reconocdor de carateres: Solo numeros -> Lector de matriculas de coche / escaner númerico de id de producto.
FOTO de un numero -> Digito


    +--+--+--+--+--+--+--+--+
    |  |  |  |  |  |  |  |  |
    +--+--+--+--+--+--+--+--+
    |  |  |  |XX|  |  |  |  |
    +--+--+--+--+--+--+--+--+
    |  |  |XX|XX|  |  |  |  |
    +--+--+--+--+--+--+--+--+   Esto es un 1
    |  |  |  |XX|  |  |  |  |
    +--+--+--+--+--+--+--+--+
    |  |  |  |XX|  |  |  |  |
    +--+--+--+--+--+--+--+--+
    |  |  |  |XX|  |  |  |  |
    +--+--+--+--+--+--+--+--+

Que el pixel 12 está marcado guarda relación con que el dígito sea un 1?
 8 x8 = 64 datos (1/0) -> 64 dimensiones
 64 variables, que puedo poner en un vector -> [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0...]
 Cómo de eso saco si la foto es un 1 o un 2 o un 3? -> Aprendizaje automático.


Resolverlo tal cual es imposible... ya que no hay relación entre el pixel 12 y el dígito que es la foto.Ç

PERO    ... puedo hacer un paso intermedio:
    - Programa 1: Mira si hay más pxiles en la parte de arriba que en la de abajo: 0 / 1 : true /false
    - Programa 2: Mira si hay igual pxiles en la parte de la arriba que en la de abajo: 0 / 1 : true /false
    - Programa 3: Mira si hay más pxiles en la parte de la izquierda que en la de la derecha: 0 / 1 : true /false
    - Programa 4: Mira si hay igual (95%) pxiles en la parte de la izquierda que en la de la derecha: 0 / 1 : true /false

Si el programa 1 me da 0
Si el programa 2 me da 1
Si el programa 3 me da 0
Si el programa 4 me da 1      Qué significa esto?Es simetrica la imagen -> 1, 0, 8
                              Podría ser un 4? 7? 9, 3? NO, no son simetricos.
Imaginad que me dice que está más cargado arriba que abajo: que numeros puede ser? 9, 7
6? 3? 0? 8?

Igual que esos 4 programas, puedo hacer otros cuandos.. que comparen diagonales.
Que miren si los datos se concentran en el centro o en los bordes... etc.
Y con toda esa información ya si puedo hacer el programa.


    64 datos

    P1  ----> Perceptron1 -> 10 variables (0/1) -> 10 dimensiones <-- esto son los embedings
    P2. \---> Perceptron2
    ...  \--> Perceptron3
    P6

    Es una forma además de codificar la información, de reducir la dimensionalidad. 

    De hecho, muchos algoritmos de redes neuronales generativas hacen esto.

    Buscar programas que de 64 datos pasen a 10... y despues de esos 10 de nuevo a 64... de forma que el resultado final (64) sea lo más parecido posible a los originales. Una vez conseguido, me quedo solo conlos segundos programas. Me invento 10 datos aleatorios y se los paso a ellos. Y tengo algo que parece una foto de las originales... pero es nueva.


--

# Usos de la RAM:

En el Oracle cuando configuro una instancia le establezco el PGA y el SGA
El SGA es la compartida entre proecesos (donde se carga la cache de páginas, la cache de planes de ejecución de queries...)
    Dentro del SGA:
     - Buffer Cache: Donde se cargan las páginas de datos que se han leído de disco. Si una página ya está en el buffer cache, no hay que leerla de disco, lo que es mucho más rápido.
     - Shared Pool: Donde se cargan los planes de ejecución de las queries. Si una query ya se ha ejecutado antes, el plan de ejecución ya está en el shared pool, lo que hace que la ejecución de la query sea mucho más rápida.
     - Redo Log Buffer: Donde se cargan los redo logs antes de escribirlos en disco. Esto es para asegurar la durabilidad de las transacciones.
El PGA es la memoria privada de cada proceso (donde se cargan los datos de cada query, los datos de cada sesión...)

Esop lo controla Oraclle... Se asegura de no llenar más la cache del tamaño que le he establecido, y de no llenar más la memoria privada de cada proceso del tamaño que le he establecido. Y dejar memoria para otras cosas.

## En OS 

Mismo modelo. Una cosa es cuanta RAM tengo (lo que doy de heap a la JVM y lo que dejo para el sistema operativo), y otra cosa es cómo uso esa RAM internamente... 
Y eso es gestión de OS:
- Queries
- Indices
- Búsquedas vectoriales
- Modelos ML