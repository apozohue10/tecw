# Producción en sistemas web

En esta sección vamos a hablar de algunos aspectos relacionados con la **producción** de aplicaciones web. 

## Virtualización

La **virtualización** permite ejecutar **varias máquinas virtuales (VMs)** en un solo servidor físico. Cada VM incluye su propio **sistema operativo**, aplicaciones y recursos virtualizados.

- Cada VM es **independiente**, con su propio kernel.
- Consumen recursos del host: CPU, memoria, disco.
- Permiten ejecutar diferentes sistemas operativos en el mismo servidor físico.
- Se gestionan con **hipervisores**:
  - Tipo 1: directamente sobre hardware (ej. VMware ESXi, Hyper-V)
  - Tipo 2: sobre un sistema operativo anfitrión (ej. VirtualBox, VMware Workstation)

### Contenedores

Los **contenedores** son una forma más ligera de aislar aplicaciones, compartiendo el **mismo kernel del host** pero con un entorno aislado para cada aplicación.

- Se ejecutan sobre el sistema operativo del host.
- Contienen solo la aplicación y sus dependencias.
- Son **muy ligeros y rápidos** de arrancar.
- Se gestionan con herramientas como **Docker** o **Podman**.

## Comparación rápida

| Característica          | Virtualización (VM)            | Contenedores                |
|-------------------------|-------------------------------|----------------------------|
| Sistema operativo       | Cada VM tiene el suyo         | Comparte kernel del host   |
| Peso / recursos         | Alto                          | Ligero                     |
| Arranque                | Lento                         | Muy rápido                 |
| Portabilidad            | Buena                         | Excelente                  |
| Aislamiento             | Fuerte                        | Moderado                   |


## Escalabilidad en Servidores Web

Cuando una aplicación web comienza a recibir más tráfico o necesita procesar más datos, el servidor que la ejecuta puede convertirse en un **cuello de botella**. En ese momento aparece el problema de la **escalabilidad**, es decir, la capacidad de un sistema para aumentar su rendimiento cuando crece la carga de trabajo.

En el ámbito de los servidores web existen dos estrategias principales para escalar un sistema:

- **Escalabilidad vertical**
- **Escalabilidad horizontal**

---

### Escalabilidad vertical

La **escalabilidad vertical (scale up)** consiste en **aumentar los recursos hardware de una única máquina**. La aplicación sigue ejecutándose en el mismo servidor, pero dicho servidor dispone de más capacidad de cómputo.

Esto puede implicar, por ejemplo:

- añadir más núcleos de CPU
- aumentar la memoria RAM
- utilizar discos más rápidos (SSD o NVMe)
- mejorar la capacidad de red

Supongamos que tenemos un servidor que ejecuta una aplicación web desarrollada con Flask. Inicialmente el sistema podría estar desplegado en una máquina con:

- CPU: 2 cores  
- RAM: 4 GB  

Si el tráfico aumenta y el servidor empieza a saturarse, podríamos migrar la aplicación a una máquina más potente:

- CPU: 8 cores  
- RAM: 32 GB  

Desde el punto de vista del software **la aplicación no cambia**, simplemente se ejecuta en una máquina con más recursos.

Este enfoque tiene varias ventajas. Es una estrategia **muy sencilla de aplicar**, ya que no requiere modificar la arquitectura de la aplicación ni introducir nuevos componentes en el sistema. Además, no es necesario implementar mecanismos de distribución de carga entre múltiples servidores. No obstante conlleva importantes limitaciones:

- **Existe un límite físico** en el hardware que puede tener una máquina ([ver más](https://calculator.aws/#/createCalculator/ec2-enhancement?nc2=h_pr_calc)). 
- El precio de manetener un servicio con un servidor muy potente puede ser **mayor** que el de varios servidores equivalentes más modestos. Si montamos nuestro propio servidor, el coste de una máquina con 8 cores y 32 GB de RAM puede ser **mucho más elevado** que el de cuatro máquinas con 4 cores y 16 GB de RAM cada una. Si montamos el servidor en la nube, aunque el coste de una instancia de 8 cores y 32 GB de RAM vs 2 instancias de 4 cores y 16 GB de RAM puede ser idéntico. Pero si tenemos picos y valles de tráfico, poder apagar y encender servidores según la demanda puede ser más rentable que mantener un servidor muy potente siempre encendido.
- A medida que se añaden más recursos, el rendimiento no siempre mejora de forma lineal debido a cuellos de botella internos (por ejemplo, la comunicación entre CPU y RAM).
- Los servidores más potentes suelen ser **mucho más caros**.
- El sistema sigue teniendo un **punto único de fallo**: si ese servidor se detiene, todo el servicio deja de estar disponible.


### Escalabilidad horizontal

La **escalabilidad horizontal (scale out)** adopta una estrategia diferente. En lugar de mejorar una única máquina, el sistema se amplía **añadiendo más servidores que ejecutan la misma aplicación**.

Por ejemplo, en lugar de disponer de un único servidor muy potente, podríamos tener varios servidores más modestos:

- Servidor 1: CPU 4 cores, RAM 16 GB  
- Servidor 2: CPU 4 cores, RAM 16 GB  
- Servidor 3: CPU 4 cores, RAM 16 GB  
- Servidor 4: CPU 4 cores, RAM 16 GB  

Cada servidor ejecuta **la misma aplicación web**, y las peticiones de los usuarios se reparten entre ellos.

Este enfoque permite escalar de forma mucho más flexible. Si el tráfico aumenta, simplemente se pueden **añadir más servidores al sistema**. En teoría, este modelo permite crecer casi sin límite.

Otra ventaja importante es la **tolerancia a fallos**. Si uno de los servidores deja de funcionar, los demás pueden seguir atendiendo peticiones, lo que mejora la disponibilidad del sistema.

No obstante, esta arquitectura también introduce mayor complejidad. Es necesario coordinar múltiples servidores, gestionar la distribución de las peticiones y garantizar que los datos compartidos (por ejemplo sesiones o cachés) se mantengan coherentes entre las distintas instancias del sistema.


### Balanceadores de carga

Cuando una aplicación se ejecuta en varios servidores surge una pregunta fundamental: **¿cómo se decide qué servidor debe atender cada petición?**

Para resolver este problema se utilizan los **balanceadores de carga (load balancers)**. Estos sistemas actúan como un punto de entrada único para los clientes. Todas las peticiones llegan primero al balanceador, y este se encarga de redirigir cada una al servidor adecuado.

Además de distribuir el tráfico, los balanceadores de carga suelen proporcionar otras funcionalidades importantes:

- Detección automática de servidores caídos
- Terminación de conexiones HTTPS
- Gestión de sesiones
- Protección frente a sobrecargas

De esta manera, el balanceador no solo reparte el trabajo entre los servidores, sino que también contribuye a mejorar la robustez y seguridad del sistema.


### Algoritmos de balanceo

Para repartir las peticiones entre servidores existen diferentes estrategias.

Uno de los algoritmos más simples es **Round Robin**, en el que cada petición se envía al siguiente servidor de una lista. Por ejemplo, la primera petición va al servidor 1, la segunda al servidor 2, la tercera al servidor 3, y así sucesivamente. Este método es fácil de implementar y funciona bien cuando todos los servidores tienen capacidades similares y las peticiones requieren un tiempo de procesamiento similar.

Otro método común es **Least Connections**, donde cada nueva petición se asigna al servidor que tiene **menos conexiones activas en ese momento**. Este enfoque puede ser más eficiente cuando algunas peticiones requieren más tiempo de procesamiento que otras.

También existen estrategias basadas en **hash de la dirección IP del cliente**, que permiten enviar siempre las peticiones de un mismo usuario al mismo servidor.

> #### El problema de las sesiones
>
> Cuando una aplicación se ejecuta en varios servidores aparece un problema habitual: **la gestión de las sesiones de usuario**.
>
> Imaginemos que un usuario inicia sesión en un servidor concreto. Si la siguiente petición de ese mismo usuario es enviada a otro servidor distinto, ese servidor podría no conocer la sesión previamente creada.
>
> Para resolver este problema se utilizan diferentes técnicas:
>
> - **Sticky sessions**, donde el balanceador intenta enviar siempre al mismo usuario al mismo servidor.
> - Almacenar las sesiones en un **sistema compartido**, como una base de datos o una caché distribuida (por ejemplo Redis).
> - Utilizar **tokens sin estado**, como los JWT, donde la información de autenticación se incluye directamente en cada petición.

---

En términos generales, la **escalabilidad vertical** consiste en aumentar la potencia de una sola máquina, mientras que la **escalabilidad horizontal** implica añadir más servidores al sistema. Los **balanceadores de carga** permiten distribuir las peticiones entre estos servidores y constituyen una pieza clave en las arquitecturas modernas.

Aunque ambos enfoques pueden utilizarse, los sistemas actuales tienden a favorecer arquitecturas **horizontalmente escalables**, ya que permiten construir servicios más robustos, distribuidos y tolerantes a fallos.

## Monitoreo de Máquinas en Sistemas Web

Cuando desplegamos aplicaciones web en producción, **no basta con que el servidor funcione**, sino que debemos **vigilar su salud y recursos**. En la sección anterior vimos cómo escalar un sistema añadiendo más servidores, pero para que esto funcione correctamente es fundamental **monitorear el estado de cada máquina**. El monitoreo de máquinas permite detectar cuando un servidor está saturado o con bajo uso, cuándo se acerca a sus límites de recursos o cuándo hay algún proceso que consume más de lo esperado. esto permite tomar decisiones informadas sobre cuándo escalar o desescalar el sietema tanto de forma manual como automática.

Algunas métricas básicas que se suelen recolectar:

| Recurso      | Métrica típica                 |
|--------------|--------------------------------|
| CPU          | porcentaje de uso, núcleos ocupados, procesos que consumen más recursos  |
| Memoria      | memoria total, memoria utilizada, memoria disponible, swap    |
| Disco        | espacio libre, velocidad de lectura/escritura, inodos           |
| Red          | tráfico de entrada y salida, errores de paquetes, saturación de interfaces   |
| Procesos     | Número de procesos activos, consumo por proceso, estado de servicios críticos        |

Estas métricas permiten **detectar cuellos de botella** y anticiparse a problemas de rendimiento.

Existen muchas herramientas, tanto locales como distribuidas, para medir el estado de la máquina:

- **htop / top** → supervisión interactiva de CPU y memoria.
- **iostat / vmstat / sar** → métricas de disco y CPU históricas.
- **netstat / iftop** → tráfico de red y conexiones.
- **Prometheus Node Exporter** → exporta métricas de hardware a Prometheus.
- **Grafana** → paneles para visualizar métricas de múltiples máquinas.
- **Nagios / Zabbix** → alertas y monitorización centralizada.

El monitoreo no sirve solo para mirar gráficos, sino para **actuar cuando algo falla**. Con esto podemos:

- **Prevenir caídas del sistema**.
- **Planificar ampliaciones de recursos**.
- **Detectar anomalías antes de que afecten al usuario**.

### Buenas prácticas

1. Medir **frecuencia y tendencias**: no solo un instante.
2. Definir **umbrales de alerta realistas**.
3. Usar **múltiples métricas juntas** (CPU + memoria + disco).
4. Registrar métricas históricas para analizar problemas recurrentes.
5. Automatizar notificaciones y respuestas a problemas críticos.

---

El monitoreo de máquinas es la **base de la observabilidad**: antes de ver errores de aplicaciones o problemas de red, siempre hay que asegurarse de que los servidores están sanos y trabajando dentro de sus límites.

## Replicacion y sharding de bases de datos

Cuando una aplicación web crece, no solo los servidores web necesitan escalar. También la **base de datos** puede convertirse en un cuello de botella. Para abordar este problema se utilizan dos estrategias principales en sistemas distribuidos:

- **Replicación**
- **Particionamiento (sharding)**

Ambos mecanismos permiten mejorar la **disponibilidad, el rendimiento y la escalabilidad** de los sistemas de bases de datos modernos.


### Replicación

La **replicación** consiste en **mantener copias de los mismos datos en varios servidores al mismo tiempo**.

En lugar de almacenar la base de datos en una única máquina, se crean varias réplicas distribuidas en distintos nodos del sistema.

Esto permite:

- **Alta disponibilidad**: si un servidor falla, otro puede seguir proporcionando el servicio.
- **Mayor rendimiento en lecturas**: las consultas pueden repartirse entre varias réplicas.
- **Reducción de latencia**: los datos pueden situarse más cerca de los usuarios.
- **Mantenimiento sin interrupciones**: backups, reconstrucción de índices o actualizaciones pueden realizarse sin detener el sistema.

Existen 3 modelos de replicación:

- **Lider-seguidor**: un nodo actúa como lider, recibiendo todas las escrituras, mientras que los nodos seguidores replican los datos del lider y se encargan de las lecturas.
- **Líder múltiple**: varios nodos pueden recibir escrituras y replicarse entre sí, lo que permite una mayor flexibilidad pero también introduce complejidad en la gestión de conflictos.
- **Sin lider**: todos los nodos pueden recibir escrituras y replicarse entre sí.

### Particionamiento (Sharding)

Mientras que la replicación **duplica los datos**, el **particionamiento** los **divide entre varios servidores**.

Cada servidor almacena **solo una parte del conjunto total de datos**. Se puede particionar la base de datos de diferentes maneras, por ejemplo:

- **Por rango**: cada partición almacena un rango específico de valores (por ejemplo, usuarios con ID entre 1 y 1000 en un servidor, entre 1001 y 2000 en otro, etc.).
- **Por hash**: se aplica una función de hash a un campo específico (por ejemplo, el ID de usuario) para determinar en qué partición se almacenan los datos.

Existen dos formas principales de gestionar el particionamiento.

- **Particionamiento fijo (algorítmico)** . El número de particiones está definido de antemano y una función determina en qué nodo se almacenan los datos.
- **Particionamiento dinámico** . Las particiones se ajustan automáticamente según el tamaño del dataset y la carga del sistema. Pero implica tener un servicio de localización que indique en qué nodo se encuentra cada dato, lo que añade complejidad a la arquitectura.

---