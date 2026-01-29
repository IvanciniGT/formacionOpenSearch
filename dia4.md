

Si desde fuera quieren acceder, tienes que dar la IP de algún nodo del cluster + el puerto NodePort asignado.
Lo normal es poner un balanceador delante (VIPA) que reparta las peticiones entre los nodos del cluster.
Esa VIPA la puede gestionar el propio kubernetes (MetalLB) o un balanceador físico (F5, Netscaler, etc), o te la puede gestionar el VMware Sphere (con su propio balanceador).

HOST 1 - esXI
    nodo-maestro1 kubernetes                  192.168.1.100:30920
                                              0.0.0.0:30920       <-- esto se limita por firewall
    nodo-worker1 kubernetes                   192.168.1.101:30920
                        labels: infra
            nodo2-elastic
    nodo-worker2 kubernetes                   192.168.1.102:30920
                        labels: apps

HOST 2.  esxi -> OFFLINE
    nodo-maestro2 kubernetes                  192.168.1.103:30920
    nodo-worker3 kubernetes                   192.168.1.104:30920
                        labels: infra
    nodo-worker4 kubernetes                   192.168.1.105:30920
                        labels: apps

HOST 3.  esxi
    nodo-maestro3 kubernetes                  192.168.1.106:30920
        ha-proxy
    nodo-worker5 kubernetes                   192.168.1.107:30920
                        labels: infra
            nodo1-elastic
            nodo3-elastic 
    nodo-worker6 kubernetes                   192.168.1.108:30920
                        labels: apps


AFINIDAD KUBERNETES:
Información que le damos al scheduler (pistas) acerca de dónde se deben ubicar los pods.

3 nodos de elastic... multi-role -> 3 pods
    nodo1-elastic
    nodo2-elastic
    nodo3-elastic

    AFINIDAD: Solo en nodos de tipo infra
    ANTIAFINIDAD A NIVEL DE POD: Que 2 pods del mismo tipo no puedan estar en el mismo nodo.

Las afinidades en algunos casos pueden irme para otro propósito.
Si tengo repertorio de nodos en el ES, no todos los nodos requieren las mismas características de hardware.

Puedo tener nodos que requieran GPU (Machine Learning)... y no en todos los nodos tengo GPU que son caras!
Si guardo datos en disco, quiero nodos con discos rápidos (NVMe)... o no!



---


    El Opensearch, a los clientes les atiende por puerto 9200 (HTTP REST).

    Se monta un balanceador (Service -> ClusterIP) atiende peticiones internas al cluster

    Si quiero que el ES/OS sea accesible desde fuera... Opciones:
        - Ingress <----
        - NodePort
        - LoadBalancer (si el entorno lo soporta - MetalLB)

    Si el OS lo quiero accesible desde fuera, pero no conectado a internet / usuarios.. RED PRIVADA
        - 2 Ingress                         <----
        - NodePort / Firewall
        - LoadBalancer con IP en red privada.


    OKD: Route
        3 configuraciones SSL
            - edge           Entre cliente y HAPROXY (Ingress Controller/Proxy reverso) se usa SSL (certificado propio - wildcard-). Entre HAPROXY y OS no hay SSL (http)
            - passthrough    USAR HTTPS DIRECTO ENTRE CLIENTE Y OS (Usa el certificado de OS).
                                El problema de esto es que el certificado de OS debe ser válido (no autofirmado y emitido por una CA reconocida por el cliente)
            - re-encrypt     EL ROUTE HACE TERMINACIÓN SSL CONTRA CLIENTE CON CERTIFICADO PROPIO (WILDCARD) Y VUELVE A ENCRIPTAR HASTA OS (Usa el certificado del OS.. puede ser autofirmado.. solo hay que registrarlo)

En cualquier caso, el OKD mete un ingress controller (HAProxy) que hace de proxy inverso entre el cliente y el OS.

Eso no es balanceo!

---

    Cluster de Kubernetes

        Tiene dentro una red virtual.
        Y en ella los pods... que pueden hablar entre si.

        Lo que expongo fuera del cluster.
            - apps         ---> Ingress    https://app1.miempresa.com         <-- clientes
            - dashboards   ---> Ingress    https://dashboards.miempresa.com   <-- clientes
            - opensearch   ---> Ingress2   https://opensearch.miempresa.com   <-- privado


        El ingress controller1 / proxy reverso abre puerto 80/443 en una IP de la red de la empresa / pública
        El ingress controller2 / proxy reverso abre puerto 80/443 en una IP de la red privada de la empresa

        Eso es lo más limpio.
        ---
        Otra que tengo es abrir un servicio NodePort / LoadBalancer para el OS. Y meter reglas de firewall.... 
        Y que solo pueda acceder a la IP/Puerto desde la red privada.

---

Si quiero montar una monitorización centralizada.

Cluster por tenant... me puede interesar tener una monitorización centralizada. Tengo un programa que periódicamente hace peticiones a los distintos clusters OS me avisa si alguno falla / o se desmadra.



---


# Instalación en un entorno de producción tradicional

     HOST 1                                                    INGRESS CONTROLLER
        app1(instancia1)    <<                                      vvvvv
     HOST 2                         Balanceador de carga      <<< PROXY INVERSO    <<<  PROXY  <<< CLIENTE
        app1(instancia2)    <<          NGINX/HAPROXY               NGINX
                                        Balanceo físico: F5         Apache httpd
                                        NETFILTER                   haproxy

                                                                    Reglas
                                                                    Cuando te hagan una petición por https a:
                                                                        https;//app1.miempresa.com -> mandala al BC376
                                                                        https;//app2.miempresa.com -> mandala al BC458
                                                                    En un apache, es lo que llamamos los virtual hosts.
La forma de configurar esas reglas cambia de proxy reverso a proxy reverso.
En kubernetes estandarizan la forma de dar esas instrucciones a través de los ingress (REGLA DE PROXY REVERSO)

Netfilter es un componente del KERNEL DE LINUX... Todo paquete de red que entra o sale del servidor LINUX pasa por netfilter.

Os suena IPTABLES? IP TABLES SOLO ES UNA FORMA DE DAR INSTRUCCIONES A NETFILTER.

El proxy protege al cliente 
Cuando el cliente quiera hacer una petición a algun sitio, es interceptada por el proxy.
Y el proxy es el que hace la petición al servidor destino. Y captura el resultado (de paso lo espia -seguridad, filtros-) y manda la respuesta al cliente

El proxy reverso protege al servidor
Cuando el cliente hace una petición al servidor, la petición es interceptada por el proxy inverso
Y le dioce al cliente.. tu ahí! Que ya voy yo.
El proxy inverso  captura el envío (de paso lo espia -seguridad, filtros-), hace la petición al servidor, y manda la respuesta al cliente.

La diferencia del ROUTE del OKD al INGRESS de kubernetes es que el ROUTE hace 2 cosas adicionales:
- Gestión de DNS (resolución de nombres)
- Gestión de certificados SSL (terminación SSL)
  
---

# Pruebas en Kubernetes e impacto en la configuración del OS

Hay 3 tipos de pruebas que se configuran en el Kubernetes:

    startupProbe:
        prueba concreta: comando / URL / TCP

    livenessProbe: 

    readinessProbe:

Cuando kubernetes arranca un contenedor de un pod, empieza a ejecutar las pruebas de StartUp (arranque)
Si esas pruebas fallan, kubernetes reinicia el pod.

Cuando esas pruebas se han superado, empiezan a ejecutarse las pruebas de Liveness (vida), para ver si el programa sigue corriendo en un estado sano. Si la prueba no se supera, kubernetes reinicia el pod.

Luego está la pruena de readiness (mira si el proceso está en un estado compatible con atender peticiones de los clientes). Si la prueba de readiness falla, kubernetes NO REINICIA EL POD: Lo que hace en su lugar es sacarlo de balanceo (lo quita del servicio asociado).


- BBDD... arranca... pero... la pongo a hacer un backup en frio /templado... no puede atender peticiones de clientes.
  La BBDD está sana? quiero reiniciarla? NO
  La BBDD está lista para atender peticiones de clientes? NO


Por defecto, Docker y cualquier otro gestor de contenedores (containerd, crio), cuando arrancan un contenedor, lo comienzan a monitorizar... Cómo se hace eso? mirando el proceso 1 del contenedor.

Si ese proceso cae, en funcion de la configuracion que tenga puesta en el gestor de contenedores, puede tratar de reiniciarlo.


---

Configuración que viene por defecto en OS al despliegr en kubernetes:
        startupProbe:
          failureThreshold: 30
          initialDelaySeconds: 5
          periodSeconds: 10
          tcpSocket:
            port: 9200
          timeoutSeconds: 3

- Le estamos dando al OS para que arranque 5 segundos. 3 que espero (timeoutSeconds) + 10 segundos que espero entre pruebas (periodSeconds) * 30 intentos (failureThreshold) = 13 * 30 = 390 segundos + 5 segundos de initialDelaySeconds = 395 segundos = 6 minutos y 35 segundos.
Eso es lo que damos para que el OS abra el puerto 9200 y responda a la prueba TCP.

Esta prueba es un poco cutre... bastante! El puerto puede estar abierto .. y el cluster no estar operativo.
Una prueba más decente es hacer una petición http y que conteste un 200.

Sin entrar en eso! 6 minutos y 35 segundos.. es suficiente? Depende del tamaño del cluster, de la configuración de plugins y de que no le pille una actualización al OS de por medio.

Quizás estoy con un volumen de un pod anterior viejo, y actualizo imagen (despliegue nuevo... pero mismos volumenes)... El OS actualiza ficheros (puede ser, depende de la versión) y tarda más en arrancar.

    Puede ser que el faileureThreshold lo tenga que subir a 300, al menos para actualizaciones de imagenes (major upgrades).

En el caso del OS.. no están viniendo pruebas de Liveness, solo de readyness.. Eso implica que kubernetes NUNCA reiniciará un pod del OS por estar en un estado no sano. ESTO NO LO QUIERO
Si el proceso puede estar corriendo.. pero si le hago un HTTP y no contesta... el pod está en un estado no sano. ME LO CRUJO. Le dare un buen timeout a que no conteste... pero si no contesta -> REINICIO (AUTOMATICO) No quiero a alguien entrando a las 3:00am a reiniciar pods del OS.

Cuidado... un buen timeout... no si en 2 seg no contesta... Pero si en 2 segundos no contesta 20 veces... mal está la cosa!


---

Instalacion con el operador:

Lo primero es instalar el operador de Opensearch en el cluster de kubernetes.
Eso es el equivalente a meter un humano que sepa un huevo de Opensearch a vivir dentro del cluster de kubernetes.
A partir de ese momento, nosotros no le pedimos ya a kubernetes nada relativo a Opensearch... se lo pedimos al operador.


    # En un entorno tradicional: Quiero montar un Opensearch
    - Tengo que montar máquinas virtuales / físicas
    - Instalar el programa bien configurado
    - Montar un balanceador de carga delante
    - Crear volumenes de almacenamiento
    - ...

    # En un entorno tradicional, donde hubiera contratado a un tio/a que sepa un huevo de Opensearch
    Mi forma de comunicación cambia. Ya no hablo de máquinas virtuales / físicas / balanceadores / almacenamiento.
    Ahora le digo:
        - Quiero un cluster de Opensearch con 3 nodos maestros y 5 nodos data
        - Quiero que el cluster tenga un volumen de almacenamiento de 2TB
        Problema del operador ahora es el montaje del cluster en mi entorno: Crear las máquinas virtuales / físicas , volumenes, balanceadores, etc 

Y cómo hago eso de hablar con el operador y no con kubernetes directamente?
Lo que hacfemos es, en los archivos de manifiesto que cargamos (los carga HELM) en kubernetes, en lugar de usar obejtos de kubernetes (Deployment, StatefulSet, Service, etc) que son gestionados por kubernetes, usamos objetos propios del operador de Opensearch (OpensearchCluster, OpensearchDashboards, etc) que son gestionados por el operador (CRD = Custom Resource Definition).

El operador , lo instalamos también mediante HELM (chart del operador).

En este caso no usamos una plantilla de despliegue (chart) de Opensearch, 
                usamos una plantilla de despliegue (chart) del operador de Opensearch.

Una vez cargado el oeprador y habilitados los nuevos CRDs, podemos:
- Crear y cargar un manifiesto (YAML) con objetos del operador (CRDs): OpensearchCluster, OpensearchUser, OpensearchRole, etc
- Usar otra plantilla de HELM que genera ficheros de manifiesto con objetos del operador (CRDs) y los carga en kubernetes. (esto solo puedo usarlo si previamente he instalado el operador en el cluster de kubernetes).



---

LA HA ES CARA ! se acepta.

 2 nodos de ES/OS:
  Escenario 1: 
   Nodo 1: 66%
   Nodo 2: 66%
   Nodo 3: 66%

   Necesito escalar? ESTOY ACOJONADO , YA VES QUE SI NECESITO ESCALAR !!!! TOTALMENTE . INSOSTENIBLE !!!
   No estoy dando HA... La escalabilidad es genial... siempre que cumpla con la HA... uy esa es cara de cojones!


Esto es jugar por lo bajo. 4 máquinas y admiten que se caigan 3 de ellas y siguen dando servicio.
Esto significa que el 75% de los recursos de la máquina están el 99,99% del tiempo en la basura! SIN USAR


Solo teneis 3 nodos... La idea con ese esquema es que si se cae 1 TODO SIGA FUNCIONANDO! Si no, no ofrezco HA.
Vuestra HA es a 1 nodo... ni siquiera a 2 nodos (para eso necesito 4 máquinas al menos.)

Ninguna máquina puede tener comprometido más del 60% de los recursos del nodo. Si es así... NO HAY HA.