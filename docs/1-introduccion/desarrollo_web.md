# Arquitecturas de aplicaciones Web

Como se ha comentado anteriormente, generalmente las comunicaciones se establece siguiendo el modelo cliente/servidor. La implementación de dicho modelo se puede hacer de diferentes formas en función de las necesidades de escalabilidad, rendimiento y mantenimiento de cada contexto. Un esquema muy sencillo sería aquel en la que el servidor web ofrece ficheros estaticos que son descargados por el cliente. Los ficheros estaticos podrían ser ficheros HTML, imágenes, audios, videos, etc. Pero existen otros esquemas más avanzados:

<ul>
    <li><strong>Arquitectura con balanceador de carga</strong>. 
        Utiliza un <strong>balanceador de carga</strong> para distribuir las solicitudes entre múltiples servidores. 
        Permiten mejorar el rendimiento y evita sobrecargas en un único servidor, pero se debe tener cuidado con la sincronización de los datos. 
        Es clave para garantizar <strong>alta disponibilidad y tolerancia a fallos</strong>.
    </li>
    <li><strong>Arquitectura basada en microservicios</strong>. 
        En este caso, la aplicación se divide en pequeños <strong>servicios independientes</strong> que se comunican mediante APIs. 
        Cada microservicio tiene su propia lógica de negocio y base de datos. 
        Por ejemplo, una aplicación de compras podría tener microservicios como el sistema de autenticación, la pasarela de pago, 
        catálogo de productos, etc. Este tipo de arquitecturas permite escalar aquellos microservicios que puedan suponer un cuello de botella 
        y de esta forma hacer un uso más eficiente de los recursos. Además, facilita despliegues más flexibles y la actualización independiente de componentes.
    </li>
    <li><strong>Arquitectura peer-to-peer (P2P)</strong>. 
        No depende de servidores centrales; cada nodo actúa como cliente y servidor al mismo tiempo, lo que permite favorecer la 
        <strong>descentralización y la resistencia a fallos</strong>. Es común en aplicaciones como redes de intercambio (Torrent) 
        de archivos o criptomonedas.
    </li>
    <li class="nested_list"><strong>Arquitectura con caché</strong>. 
        Se puede utilizar <strong>caché en diferentes niveles</strong> para mejorar el rendimiento y reducir la carga en los servidores. 
        Estos elementos permiten agilizar la obtención del recurso. Hay varios tipos de caché:
        <ul>
            <li class="nested_list"><strong>Caché en el cliente</strong>: Almacena datos en el navegador o aplicación para reducir solicitudes repetitivas.</li>
            <li class="nested_list"><strong>Caché en el servidor</strong>: Guarda respuestas generadas para acelerar futuras peticiones.</li>
            <li class="nested_list"><strong>Caché en la base de datos</strong>: Usa sistemas como <strong>Redis o Memcached</strong> para evitar consultas repetitivas.</li>
        </ul>
    </li>
</ul>

<br>
Un modelo clásico es la arquitectura en tres capas (Three-Tier Architecture), que divide la aplicación en una capa de presentación (frontend), una capa de lógica de negocio (backend) y una capa de datos (base de datos), facilitando la modularidad y el mantenimiento.

## Arquitectura en 3 capas

Uno de los modelos más clásico es la arquitectura en tres capas (Three-Tier Architecture), que divide la aplicación en:

- **Capa de presentación (frontend)**: Capa de visualización y presentación con la interfaz del servicio. Se encuentra en el lado del cliente.
- **Capa de lógica (backend)**: Capa lógica con las reglas de orquestación de la respuesta a las peticiones. Se encuentra en el lado del servidor.
- **Capa de datos (base de datos)**: Capa de persistencia que almacena los datos en una base de datos. Se suele encontrar en el lado del servidor.
<br>

<div class="img-center">
    <img src="_images/introduccion/3capas.png" alt="Arquitectura en 3 capas" />
</div>

<br>
Los distintos elementos se comunican entre si usando distintos protocolos como HTTP, WebSockets, RTSP, WebRTC, etc. Como se verá más tarde, esta arquitectura guarda un paralelismo con un patrón para el desarrollo de aplicaciones llamado **MVC**.

## Desarrollo web

Para desarrollar servidores web, se pueden utilizar diferentes entornos, frameworks, librerías y lenguajes de programación. Cada aplicación web puede ser construida con tecnologías variadas, como Python con Flask o Django, Node.js con Express, Ruby on Rails, o incluso lenguajes como Java o PHP. La flexibilidad de elegir entre estos entornos permite a los desarrolladores adaptar la tecnología a las necesidades específicas de la aplicación. En Internet, cada aplicación usa su propio entorno, y es aquí donde los **protocolos** y **estándares** juegan un papel crucial. Protocolos como **HTTP** permiten que aplicaciones construidas con diferentes tecnologías puedan comunicarse entre sí. Estos protocolos proporcionan un marco común que facilita la interoperabilidad y asegura que los datos sean intercambiados correctamente entre clientes y servidores, independientemente del entorno que tengan. La siguiente imagen recoge algunos de las decenas de entornos que permiten el desarrollo y despliegue de servicios web.


<div class="img-center">
    <img src="_images/introduccion/entornos.png" alt="Entornos de desarrollo web" />
</div>

<br>

A través de estos frameworks, se pueden abordar las capas de presentación y lógica de la arquitectura de 3 capas. Para la capa de datos, existen multitud de bases de datos, tanto SQL como NoSQL. Los frameworks o lenguajes de programación antes descritos poseen de librerías que permiten establecer conexiones con las distintas bases de datos. Y al igual que ocurre entre cliente y servidor, existe una serie de protocolos específicos de cada base de datos que permiten su conexión.

Para finalizar, comentar que para desarrollar aplicaciones web, existen diversos **Entornos de Desarrollo Integrados (IDE)** y **herramientas de desarrollo** que facilitan la escritura, depuración y optimización del código. A continuación, se mencionan algunos de los más utilizados:

- **Visual Studio Code** (gratuito): Es uno de los editores más populares debido a su ligereza, extensibilidad y soporte para una gran cantidad de lenguajes y tecnologías. Puedes obtenerlo de manera gratuita en su página oficial: [Visual Studio Code](https://www.visualstudio.com/es/).
  
- **WebStorm** (pago, con licencia gratuita para educación): Este IDE está diseñado especialmente para el desarrollo web y soporta tecnologías como JavaScript, TypeScript, y frameworks como Angular y React.

- **Developer Tools** (Herramientas para desarrolladores de navegadores como Chrome, Firefox, Safari, Edge, etc.): Estas herramientas permiten depurar y analizar las aplicaciones web directamente desde el navegador. Entre las funciones más útiles se encuentran:
  - **Visor de código fuente**: Permite visualizar y explorar el código HTML, CSS y JavaScript de una página web.
  - **Consola JavaScript**: Ideal para ejecutar scripts y depurar errores en tiempo real.
  - **Analizador de elementos HTML y CSS**: Muestra cómo están estructurados los elementos HTML y permite realizar ajustes de CSS de forma visual y directa.

Estas herramientas son esenciales para el desarrollo web, ya que facilitan la depuración, optimización y adaptación de las aplicaciones a distintos entornos y plataformas.
