
# Formación de un cluster

seed_hosts. Cada nodo del cluster tiene en la lista de seed_hosts a los nodos maestros (en kubernetes, ponemos el service que apunta a los nodos maestros). Esto ves válido apra nodos que NO SEAN MAESTROS.
Para nodos maestros eso no vale! El balanceador me podría poner en comunicación conmigo mismo (loopback) y no formar cluster

El Elastic, antiguamente la formación de un cluster se hacía mediante un mensaje de tipo broadcast que se lanzaba por parte de todos los nodos en una red. Eso daba problemas... y o cambiaron por un procedimiento unicast (comunicaciones de nodo a nodo) en el que cada nodo conoce de antemano la lista de nodos que pueden formar parte del cluster. Estos nodos se llaman seed hosts (anfitriones semilla).


    Maestro1  <<       <<   Data1          Coordinador1   <<<   BC      <<<    Clientes
                   BC
    Maestro2  <<       <<   Data2          Coordinador2   <<<

                            Data3
                                                                ^^
                                                            OpenSearch Dashboards (Kibana) <<< BC <<< Clientes
                                                                                              VIPA
                    opensearch-masters

        Maestro1:   maestro1.opensearch-masters
        Maestro2:   maestro2.opensearch-masters

Cuando un nodo del tipo que sea, se representa a un maestro, el maestro ya le presenta al resto de integrantes del cluster.

---

# Instalación en kubernetes.

En kubernetes tenemos el concepto de pod.
Nosotros nunca creamos pods dentro de un cluster de kubernetes.. eso lo hace kubernetes.
Nosotros definimos plantillas de pods:
- Deployment      Plantilla de pod + # inicial de réplicas
- StatefulSet     Plantilla de pod + # inicial de réplicas + Plantillas de PVC + Identidad (pero no solo el nombre del pod, sino también la identidad de red!)
- Daemonset       Plantilla de pod, de la que kubernetes genera un pod por cada nodo del cluster de kubernetes.
                  - U uso por ejemplo de un Deployment sería desplegar un contenedor que existiera en cada nodo del cluster de kubernetes, que establezca la propiedad max_map_count en cada nodo del cluster de kubernetes.


## Balanceador de carga en Kubernetes

Service: ClusterIP = IP Fija de la red interna del cluster + Balanceo de carga entre los pods + Entrada en el DNS interno del cluster de kubernetes.


---

# Java y la RAM

Java hace un destrozo con la RAM.... es una feature de JAVA.

```java
String texto = "Hola";     // Esto lo que hace es asignar la variable al texto "Hola"
```

Si fuera C, si que asignaría el texto "Hola" a la variable texto.
Pero en JAVA, lo que se asigna es la variable texto a un objeto que contiene el texto "Hola".

// 1 - Crear un objeto en memoria RAM que contiene el texto "Hola" (String)
// 2 - Crear una variable llamada texto que puede apuntar a datos de tipo String
// 3 - (=) Asignar a la variable texto la dirección del objeto en memoria RAM que contiene el texto "Hola"
 La variable es como un postit.. la pegamos en la RAM al lado del objeto que contiene el texto "Hola".


```java
texto = "Adios";     // Esto lo que hace es reasignar la variable al texto "Adios"
```
// 1 - Crear un objeto en memoria RAM que contiene el texto "Adios" (String)
       Ese nuevo texto "Adios" se crea en una zona de la RAM diferente al texto "Hola" o en la misma (sobreescribiendo el texto "Hola")? En una nueva zona de la RAM
       En este momento en RAM tenemos 2 objetos de tipo texto "Hola" y "Adios"
       Y AQUI ESTA EL DESMADRE DEL USO DE LA RAMA EN JAVA!
// 2 - La variable texto ya existe, no hay que crearla... Lo que hacemos es reasignar la variable texto a otro objeto en memoria RAM. Despego el postit de donde estaba pegado (junto al objeto "Hola") y lo pego junto al objeto "Adios".

El objeto "Hola" queda huérfano de variable. Ninguna variable apunta a ese objeto en memoria RAM. Y desde JAVA ya no hay forma de poder volver a recuperar ese dato "Hola". Ese dato se convierte en basura (GARBAGE).
En algún mopmento entrará (o no) el GARBAGE COLLECTOR de JAVA y liberará esa zona de la RAM que ocupa el objeto "Hola" para que pueda ser reutilizada por otros objetos.

Esto da un patrón característico del uso de RAM en las apps JAVA (y que veremos en OpenSearch):
- Una vez la app está en uso normal, veremos las gráficas de memoria y parecerán la hoja de un cuchillo... subidas y bajadas constantes en el uso de RAM. ESO ES PERFECTO! Eso es lo que se espera de una app JAVA.

Se van creando datos de trabajo... y se quedan ahí acumulados como basura... Cuando hay mucha, el garbage collector libera esa RAM.

      /|  /|  /|
     / | / | / |
    /  |/  |/  |

JAVA. necesita más RAM de la que a priori sería necesario (por ejemplo si el programa lo hubiera hecho en C)... para basura!. Cuanto más le ponga, menos tendrá que entrar el garbage collector a liberar RAM, que consume recursos.
Hay que encontrar un buen equilibrio. Ni queremos tirar RAM, ni queremos que el garbage collector esté cada segundo liberando RAM.... si esto ocurre, el rendimiento cae en picado.

En el caso de JAVA... además hay otra consideración... que ocurre lo mismo en Kubernetes! Nos sale en este caso por partida doble!

Las apps java se ejecutan en una JVM (Java Virtual Machine).... Ahí pone: Virtual Machine... es una máquina virtual.... como si fuera una VM de VirtualBox o VMWare.
A esa máquina le configuramos memoria RAM: -Xms512m -Xmx2g
-Xms512m  => Memoria RAM inicial que se reserva para la JVM
-Xmx2g    => Memoria RAM máxima que puede usar la JVM

En el caso de JAVA la recomendación es CLARA: SIEMPRE poner Xms = Xmx... salvo que tenga muy claro el comportamiento interno de la app JAVA.... y entendiendo perfectamente la implicación que eso tiene, y en que casos me puede servir.

Ponemos esos dos valores iguales entre si para asegurarnos que cuando la app vaya a coger la memoria que estimo que va a necesitar, esa memoria ya está reservada en el sistema operativo y que no haya problemas.

En el caso de Opensearch/ElasticSearch la recomendación es poner entorno a la mitad de la RAM del sistema operativo a la JVM (el resto es para SO + CACHES de disco + otros procesos).

Si no dejo espacio en RAM al SO para caches de disco, el rendimiento de Opensearch/ElasticSearch cae en picado.

## NOTA ADICIONAL (Adelantándonos a temas de kubernetes)

En docker, a los contenedores no solemos limitarles el acceso a recursos (RAM/CPU) salvo que sea necesario.
Pero en kubernetes, es TOTALMENTE OBLIGATORIO EN CUALQUIER ENTORNO DE PRODUCCION poner límites de RAM y CPU.

Eso son los "resources" que se establecen a nivel de pod (realmente los establecemos en la plantilla del pod, en el deployment o statefulset).

En kubernetes se establecen 2 valores para cada recurso (RAM/CPU):
- requests: Es la cantidad del recurso que sea que se garantiza al pod. La tiene en cuenta el Scheduler de kubernetes a la hora de decidir en que nodo del cluster de kubernetes poner el pod.
- limits:   Es la cantidad máxima del recurso que kubernetes le permitiría usar al pod caso que haya excedente.

            REAL             SIN COMPROMETER      USO            LIBRE 
    NODO1: 20GB   10CPU      0GB    0CPU       15GB    7CPU     0GB    0CPU
      data1 10GB  5CPU                         1GB     1CPU
      data2 10GB  5CPU                         10GB    5CPU
    NODO2: 20GB   10CPU      20GB   10CPU
      data 3...

    Despliego un DATA(data1). Ese data tiene:
        resources:
            requests:      <--- SE USAN EN EL SCHEDULER al determinar DONDE SE UBICA EL POD
                memory: 10GB
                cpu: 5
            limits:
                memory: 15GB <--- Si hay hueco, se le permite usar hasta 15GB si los necesita
                cpu: 10      <--- Si hay CPU libre, se le permite usar hasta 10 CPU si los necesita
    Despliego ahora un data2.. igual que el data1
    Despliego ahora un data3

Estamos usando LINUX: UN SO de tiempo compartido. Si un proceso necesita más CPU de la que tiene asignada, el SO se la puede dar si hay CPU libre en el sistema. Y si no lo hay, puede encolar las peticiones de CPU que esté haciendo otras procesos y repartir la CPU entre todos los procesos que la estén pidiendo.

A un programa le puedo quitar RAM? NO

Le manda una señal SIGTERM para que se cierre correctamente. (Vete muriendo... si no responde en un timeout predefinido: 30s) le manda una señal SIGKILL (mátalo YA!)

RECOMENDACION EN KUBERNETES:
- En todo pod (salvo que tenga muy claro el comportamiento de la app y las implicaciones) poner el REQUEST DE RAM IGUAL AL LIMIT DE RAM. (Como en JAVA -Xms = Xmx)
- En CPU, poner EL LIMIT DE RAM a lo que me de la gana... oye.. si hay hueco... coño aprovecha!

Son pocos los casos en Kubernetes en los que interesa poner REQUEST < LIMIT en RAM.
Ejemplos de esto sería por ejemplo el contenedor que establece el max_map_count en cada nodo del cluster de kubernetes (Daemonset)... que solo se usa al inicio... y luego no consume RAM.
Un provisionador de volumenes dinámicos (como el de AWS EBS o GCE PD)... que solo se usa puntualmente para crear los volúmenes... y luego no quiero que bloquee RAM.

    En estos casos, pongo el limit a lo que el programa realmente necesita y el request a una cantidad ínfima (por ejemplo 64MB) para que no me bloquee RAM en el nodo de kubernetes.

Hay otro tema importante... A veces nos volvemos locos con cuánto pongo de RAM y cuánto pongo de CPU...
La realidad es que da igual!
La CPU escala en horizontal... Si necesito más cpu, más nodos! (Hasta cierto punto... ya dijimos que más nodos = sobrecoste (overhead en RAM y Disco)... pero tampoco nos ponemos tan finos... a no ser que las cifras sean muy alta... entonces lo miramos con más detalle).
Y la RAM depende de CPU.
Lo crítico, especialmente en un cluster es que el ratio RAM/CPU sea el adecuado <- MONITORIZACIÓN!
Inicialmente , pondré lo que sea (me lo voy a sacar la manga).
Más CPU que ponga... más peticiones que podré procesar en paralelo y por ende más RAM necesitaré.
Menos CPU... menos peticiones en paralelo... menos RAM.
Hay una cantidad mínima de memoria que necesito para que el sistema arranque y dé un rendimiento adecuado (buena cache)... a partir de ahí, + RAM es inutil... si no hay CPU suficienmte como para meter (absorber) trabajo que use esa RAM.

Si no consigo buen RATIO, puede pasarme que tire RAM a kilos en el cluster (bloquee nodos de kubernetes) y que el rendimiento del cluster sea una mierda porque no tengo CPU suficiente para aprovechar esa RAM.


---

# Instalando OpenSearch en Kubernetes

Kubernetes trabaja con Archivos de manifiesto -> YAML 
Esos archivos se escriben usando un lenguaje declarativo.

---

# Lenguaje declarativo

Entra dentro de lo que llamamos paradigmas de lenguajes de programación.
Estamos muy acostumrbados a usar lenguajes imperativos.

Hoy en día, vivimos en el mundo de la AUTOMATIZACION.
El trabajo como SYSADMIN ya no es administrar sistemas!
El trabajo del SYSADMIN es AUTOMATIZAR la administración de sistemas!
Dicho de otra forma: Crear o configurar programas que administren sistemas por nosotros.

Como sysadmins, nos han cambiado de curro.. ahora somos PROGRAMADORES (de scripts/programas de automatización).

Como parte de esa adminsitración de sistemas tenemos el despliegue de aplicaciones en servidores o su operación diaría (monitorización, backups, actualizaciones, reinicio, etc)

Esas operaciones, no las hacemos ya nosotros... las hace KUBERNETES por nosotros.
Kubernetes es quién opera el entorno de producción de la empresa!
Mi misión es darle instrucciones a KUBERNETES para que sepa como quiero que opere mi entorno de producción.

Lo que vamops a estar haciendo son programas que se ejecutan en KUBERNETES para que KUBERNETES opere mi entorno de producción.

Durante décadas, hemos usado lenguajes IMPERATIVOS para programar: BASH, PYTHON, PERL, JAVA (90%)

Cada día más odioamos el paradigma imperativos.

Eso de paradigma no es sino un nombre "un tanto hortera" que los desarrooladores damos a las distintas formas de usar un lenguaje de programación.

Pero no es algo propio de los lenguajes de programación. en los lenguajes naturales (español, catalán, inglés...) también existen distintos paradigmas.

> Felipe, IF(Si) hay algo que no sea una silla debajo de la ventana: (CONDICIONAL)
> Quítalo.   IMPERATIVO (Orden)
> IF(no hay silla debajo de la ventana): (CONDICIONAL)
> Felipe, if not silla (silla ==false) GOTO IKEA , Compra silla!
> Felipe, pon una silla debajo de la ventana.   IMPERATIVO (Orden)

El lenguaje imperativo hace que me olvide de mi objetivo, y que me centre en cómo conseguir llegar a ese objetivo, dependendiendo de los potenciales estados INICIALES en los que me pueda encontrar.

> Felipe, debajo de la ventana tiene que haber una silla. Es tu responsabilidad.
Esto no es imperativo... es DECLARATIVO.
Delego la responsabilidad en otra persona (KUBERNETES) de conseguir lo que yo quiero (tener una silla debajo de la ventana). Con total independencia del estado inicial (puede que ya haya una silla... o 2, una lámpara, un sofá... o nada debajo de la ventana).


El lenguaje declarativo, lleva asociado un concepto llamado IDEMPOTENCIA!

Da igual cuantas veces hagas una operación (ejecutes un programa), y da igual el estado inicial en el que te encuentres... el resultado final siempre será el mismo.

Todas las herramientas que están triunfando a día de hoy lo hacen por usar un lenguaje declarativo (y ofrecer idempotencia - en mayor o menor grado):
- Terraform
- Kubernetes
- Ansible
- Docker (compose)
- Spring
- Angular
  

Con kubernetes declaro/manifiesto (en un archivo de manifiesto) cómo quiero que sea mi entorno de producción y cómo quiero que sea operado. Y esas instrucciones se las cargo a kubernetes para que él se encargue de conseguir/mantener ese estado deseado por mi.

Esos ficheros son complejos... muy muy muy complejos! Pero lo son no por el formato.. sino por lo que contienen... La descripción de un entorno de producción, que es compleja por naturaleza.

Y además, pasa otra cosa... y es que al final.. más o menos todos los entornos de producción son parecidos... y al final, mi fichero de manifiesto es primo hermano del fichero de manifiesto del de la empresa en la que trabajaba mi amigo.

Los fabricantes de software, a día de hoy, hacen 2 cosas:
- Publican su software en IMAGENES DE CONTENEDOR (un archivo comprimido - tar- que contiene una estructura de carpetas compatible con POSIX y un montón de programas YA INSTALADOS DE ANTEMANO en esas carpetas... con una configuración por defecto.)
  En la imagen me viene un programa ya instalado de antemano por alguien (normalmente el fabricante)... con una configuración por defecto... que solemos adaptar a nuestro escenario mediante ficheros de configuración inyectados/variables de entorno.
- Plantillas de despliegue de su aplicación en un cluster de kubernetes.
  Esas plantillas, se suelen crear en un formato concreto (CHARTS)... que son procesadas por un programa llamado HELM (gestor de paquetes para kubernetes, como apt lo es para debian/ubuntu o yum para rhel). 

Una cosa es una app... y otra cosa es desplegar esa app en un entorno de producción (HA/Escalabilidad/Seguridad mejorada).

Entorno de producción:
- Vaults
- Volúmenes de almacenamiento
- Balanceadores de carga
- Proxies reversos
- Reglas de firewall (comunicación)
- ...

## HELM

Es un programa que:
- Toma una plantilla de despliegue (CHART) de una aplicación concreta (por ejemplo OpenSearch)
- Toma un fichero de parametrización (valores concretos para mi entorno de producción)
- Genera los archivos de manifiesto YAML que necesito para desplegar esa aplicación en mi cluster de kubernetes.
- Y los aplica en el cluster de kubernetes 

Además, gestiona las versiones de los despliegues (actualizaciones de la app):
- Si tiene que quitar cosas de en medio (por ejemplo quitar reglas de firewall que ya no son necesarias)
- Si tiene que añadir cosas nuevas (por ejemplo añadir nuevas reglas de firewall)
- Si tiene que cambiar cosas (por ejemplo cambiar el tamaño de los volúmenes de almacenamiento)
- ... el se apaña! en general bien! a veces se compleja la cosa y hay que ayudarle un poco.

Una cosa es la versión de mi app: OpenSearch 3.4.0
Otra cosa es la versión de la plantilla de despliegue (CHART) de OpenSearch 3.4.0: opensearch-helm-chart 17.5.2
    - En esta versión se ha corregido un bug, con respecto a la versión anterior.. se ha dado de alta una nueva regla de firewall que antes se nos había olvidado
Y otra cosa es la versión de mi despliegue concreto, que he generado con esa plantilla de despliegue (CHART) y mi fichero de parametrización (valores concretos para mi entorno de producción).
- He pasado del 17avo despliegue al 18avo despliegue, porque he cambiado el tamaño de los volúmenes de almacenamiento de datos de 100GB a 200GB. O he cambiado de proveedor cloud para los volumenes (AWS a GCP).

La infra, hoy en día la trato como si fuera código (Infrastructure as Code - IaC) -> CON VERSIONES, que son independientes de las versiones de la app que estoy desplegando.

---

# Opensearch.

La gente de Opensearch nos ofrecen 2/3 formas de desplegar Opensearch en kubernetes:
* Plantilla de despliegue (CHART) oficial de Opensearch.
  Esta plantilla nos permite definir un entorno de producción básico de Opensearch en un lenguaje muy similar al de kubernetes (YAML).

    PRODUCTO: Un cluster de Opensearch en kubernetes

* Operador de Opensearch (que instalo mediante una plantilla de despliegue - CHART - de HELM).
  El operador me ofrece nuevos tipos de objetos (CRD) que puedo usar para definir mi entorno de producción de Opensearch en un lenguaje aún más alto nivel que el anterior.
    Básicamente, un operador es otro hombrecillo/mujercilla que vive dentro de mi cluster de kubernetes y se encarga de automatizar otras tareas (en nuestro caso relacionadas con Opensearch).
    Yo hablo con este individuo... y él a su vez habla con kubernetes para que las cosas estén como yo quiero.

    PRODUCTO: Un operador que gestiona clusters de Opensearch en kubernetes
              Pero inicialmente no tengo ningún cluster de Opensearch desplegado.

    A ese operador le digo ahora:
        - Quiero un cluster de Opensearch con 3 nodos maestros, 3 nodos datos y 2 nodos coordinadores.
        - Quiero otro cluster de Opensearch con 5 nodos maestros, 5 nodos datos y 3 nodos coordinadores.
        - Quiero en el primer cluster un índice llamado logs-2024 con 5 shards y 1 réplica.
        Y el operador se encarga de hablar con kubernetes para que todo eso esté como yo quiero.

    Cómo le doy esas instrucciones al operador? De nuevo 2 opciones: 
        - Trabajar con ficheros de manifiesto de kubernetes (usando objetos CRD que el operador me ha instalado en kubernetes)
        * Usar un nuevo chart de HELM que me ha proporcionado Opensearch, apra pasarle instrucciones al operador en un lenguaje aún más alto nivel.


---

# Ficheros de manifiesto en kubernetes

```yaml
kind:           Namespace # Tipo de objeto que estamos DECLARANDO
apiVersion:     v1  # La librería que gestiona ese tipo de recursos en kubernetes
                    # Habitualmente es de la forma:    libreria/version

metadata:
    name:

spec:
```

Kubernetes de serie viene con unas cuantas librerías (core, apps, networking.k8s.io, rbac.authorization.k8s.io, etc)... Y viene con unos 50 tipos de objetos distintos (Pod, Service, Deployment, StatefulSet, ConfigMap, Secret, Ingress, Role, RoleBinding, etc). Con eso... hacemos mucho...

Pero, una gran gracia de kubernetes es que nos permite instalarle nuevas librerías que ofrecen nuevos tipos de objetos que podemos decinir/usar dentro de nuestro cluster de kubernetes.

Esos tipos de objetos que no son parte del estandar de kubernetes, se llaman CRD (Custom Resource Definition).

Los programas (plugins) que instalan esas nuevas librerías y nos permiten gestionar nuevos tipos de objetos en kubernetes se llaman OPERADORES.

> Para yo desplegar un cluster de OS en un kubernetes, necesito:
> 3 statefulsets (maestros, datos, coordinadores)
        Plantilla de despliegue + Plantilla de Volumen + Número inicial de réplicas 
> 3 services (maestros, datos, coordinadores)
        Balanceadores de carga internos del cluster de kubernetes       
> 1 deployment (opensearch dashboards)
        Plantilla de despliegue + Número inicial de réplicas
> 1 service (opensearch dashboards)
        Balanceador de carga interno del cluster de kubernetes
> 1 ingress para el acceso externo a opensearch dashboards
        Regla de proxy reverso
> Otro ingress para el acceso externo a opensearch (API REST)
        Regla de proxy reverso
> Varios secrets (certificados TLS, usuarios/contraseñas)
        Vaults donde kubernetes guarda datos sensibles
> Varios configmaps (configuración de opensearch, configuración de opensearch dashboards)
        Carpeta compartida en red con configuraciones que inyectamos en los programas
> PodDisruptionBudget (para evitar que se caigan demasiados nodos a la vez)
        Politica de alta disponibilidad
> ResourceQuota (para limitar el uso de recursos del namespace donde se despliega opensearch)
> LimitRange (para limitar el uso de recursos por pod dentro del namespace donde se despliega opensearch)
> NetworkPolicy (para limitar las comunicaciones entre pods dentro del namespace donde se despliega opensearch)
        Reglas de firewall
> ... decenas de cosas

Claro... Kubernetes espera que le hable en estos términos.
Puedo hacerlo... o puedo instalar un programa que me permita hablar un lenguaje de aún más alto nivel... como por ejemplo el OPERADOR DE OPENSEARCH.

- Quiero un cluster con 2 maestros, 3 datos y 2 coordinadores
- Tu sabrás! Pones... pues lo normal en una instalación de OpenSearch en producción!
- Lo que pone to'el mundo:
   - Vaults para guardar los secretos
   - Volúmenes de almacenamiento para datos
   - Balanceadores de carga internos
   - Reglas de firewall (9300-9400, 5601)
   - ...

El operador me ofrece nuevos tipos de objetos: Por ejemplo el objeto: Cluster de Opensearch

```yaml
kind:           OpensearchCluster
apiVersion:     opensearch.org/v1
metadata:
    name:       mi-cluster-opensearch
spec:
    general:
        version:    3.4.0
    nodes:
        - name:       maestros
          roles:      [master]
          replicas:   2
          resources:

          ...
```

Los operadores, que no son sin más programas que montamos en un cluster, a su ver, solemos montarlos mediante plantillas de despliegue (CHARTS) de HELM.

---


# DEVOPS

Una cultura, un movimiento, una filosofía en pro de LA AUTOMATIZACION!

# AUTOMATIZAR

Automatizar es crear una máquina (o cambiar el comportamiento de una mediante programas) para que haga lo que antes hacía un humano.

Puedo automatizar el lavado de la ropa (lavadora.. que incluso puedo cambiar su comportamiento con programas "de lavado": ropa delicada, ropa muy sucia, ropa de color, ropa blanca, etc). Ya no hace falta un humano rascando ropa contra una tabla de lavar.

Enm nuestro escenario, la máquina ya la tenemos = COMPUTADORAS
Para nosotros automatizar = CREAR PROGRAMAS que hagan que la computadora haga lo que antes hacía un humano.

---

Los volumenes pueden trabajar:
- A nivel de fichero (NFS, CIFS/SMB)
- A nivel de bloque (iSCSI, Fibre Channel, EBS, GCE PD, etc)
- A nivel de objeto (S3, Swift, etc)

Por ejemplo, si estoy con un Oracle/MariaDB/PostgreSQL... esas apps para su BBDD tienen un fichero de datos gordo o muchos ficheros pequeños? Son ficheros gordos los datafiles.
Y de esos ficheros a veces cambiamos solo un pequeño trozo (un par de páginas de 8KB)... pero el resto del fichero no cambia. En este tipo de casos es mejor trabajar a nivel de bloque.

Un app server que montase, donde los usuarios pueden subir ficheros, que la app los guarda en una carpeta local, eso me interesaría monatrlo a nivel de fichero (NFS, CIFS/SMB).

Los almacenamientos a nivel de archivo se comparten fácilmente en red a varios servidores (NFS, CIFS/SMB).
Todos ellos en modo lectura/escritura concurrente (varios servidores pueden leer y escribir a la vez en el mismo recurso compartido).

Los almacenamientos a nivel de bloque, normalmente se montan en un solo servidor (salvo que use sistemas de ficheros clusterizados - GFS2, OCFS2, etc).

OpenSearch al contrario que otras apps de bases de datos, no trabaja con ficheros gordos... trabaja con esos archivos de segmentos de 64MB... y va creando nuevos archivos de segmentos conforme se van necesitando.
Aqui un almacenamiento a nivel de bloque no aporta nada... se puede montar perfectamente un almacenamiento a nivel de fichero (NFS, CIFS/SMB) y OpenSearch trabajará perfectamente con él.


---


Los volumenes de datos, no son los únicos volumenes que vamos a tener en un ES/OS en kubernetes.
Qué otros volumenes necesitamos? Backups! <- Almacenamiento a nivel de fichero

Nos interesa que ese volumen sea compartido entre todos los nodos del cluster de kubernetes (varios pods de ES/OS) para que todos puedan escribir backups ahí.

En ES, los backups de índices (snapshots) son lógicos... no físicos.

Eso requiere en los nodos del cluster tener el nfs-common instalado (cliente NFS).
