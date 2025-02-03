# Historia de Internet

La evolución de Internet ha cambiado por completo la manera en que nos relacionamos, trabajamos y compartimos información. Lo que comenzó como una simple red para investigadores, hoy conecta a miles de millones de personas y dispositivos alrededor del mundo. En este proceso, es clave el desarrollo de los servidores web, los cuales han permitido crear multitud de aplicaciones como redes sociales, periodicos o la banca online. En esta sección vamos a ver cómo ha sido ese proceso y cómo estos cambios han dado forma al Internet que usamos todos los días.

## Orígenes

Internet tiene sus raíces en Arpanet, una red de investigación que se estableció en enero de 1983. Desde sus inicios, la idea era conectar ordenadores utilizando el protocolo TCP/IP, un sistema de comunicaciones que ha permitido la creación de una red global. TCP/IP es fundamental porque soporta aplicaciones cliente-servidor, que se comunican a través de una interfaz de sockets. 

El modelo **cliente-servidor** es la base de las aplicaciones distribuidas en Internet. Este modelo divide las tareas entre dos partes: los **servidores**, que proveen recursos o servicios, y los **clientes**, que solicitan y consumen estos recursos. Las primeras aplicaciones sobre Internet fueron herramientas sencillas como **telnet** (una terminal virtual para acceder a otros sistemas), **FTP** (para la transferencia de archivos) y el **correo electrónico**.

En 1989, **Tim Berners-Lee** propuso una nueva idea: la **World Wide Web (WWW)**, un espacio interconectado de información basado en documentos hipertexto. Estos documentos poseen un identificador único y están conectados entre sí mediante **hiperenlaces**. Los usuarios pueden acceder a estos documentos a través de navegadores web, mientras que los servidores web están encargados de servir esos contenidos estáticos. Para ello, Berners-Lee  desarrolló las tres tecnologías esenciales para la Web:

- **URL**: Para identificar recursos de manera unívoca.
- **HTML**: Un lenguaje de marcado que estructura y presenta documentos en la Web.
- **HTTP**: El protocolo que permite la transferencia de información entre el cliente y el servidor.

Desde la creación de la Web, la tecnología ha evolucionado de manera constante. Por ejemplo, el protocolo **HTTP** ha ido mejorando con versiones como HTTP/1.1, **HTTPS** (que agrega seguridad mediante SSL) y **HTTP/2**. Además, **JavaScript** ha experimentado grandes cambios desde sus primeras versiones, con el nacimiento de **ES6** y **ES7**. A su vez, la estructura de los documentos ha pasado de HTML y CSS básicos a las potentes versiones **HTML5** y **CSS3**, que permiten una interacción más rica y moderna.

También han surgido nuevos estándares en los navegadores, como el soporte para pestañas y extensiones, mientras que los **servidores web** han evolucionado hacia modelos más dinámicos, capaces de interactuar con bases de datos y ofrecer contenidos personalizados.

## Evolución páginas web

### Década de 1990:
En los primeros años de la Web, **JavaScript** apenas existía y la lógica de las aplicaciones web residía casi completamente en el lado del servidor. En este periodo, las páginas web eran principalmente **estáticas** y se servían usando el método **HTTP GET**, mientras que los formularios se enviaban con **HTTP POST**. Cada vez que un usuario interactuaba con una página, la página completa se recargaba, lo que hacía que la experiencia fuera bastante lenta y poco eficiente. 

<div class="img-center">
    <img src="/_images/introduccion/1990.png" alt="Páginas de 1990" />
</div>

### Década de 2000:
A medida que Internet maduraba, surgieron nuevas tecnologías como **AJAX** (Asynchronous JavaScript and XML), que permitió la comunicación asíncrona entre el cliente y el servidor a través de **XMLHttpRequest**. Esto hizo posible actualizar partes de una página web sin necesidad de recargarla completamente, mejorando así la experiencia de usuario. Sin embargo, en este periodo, la mayor parte de la lógica de las aplicaciones seguía residiendo en el servidor.

<div class="img-center">
    <img src="/_images/introduccion/2000.png" alt="Páginas de 2000" />
</div>

<br>

### Década de 2010 y más allá:
Con el paso del tiempo, las aplicaciones web evolucionaron hacia lo que hoy conocemos como **SPA** (Single Page Application), un tipo de arquitectura en la que la mayoría de la lógica de la aplicación se ejecuta en el **cliente**. Esto permite una experiencia más fluida y rápida, ya que la mayoría de la interacción ocurre sin tener que recargar la página. El servidor se limita a almacenar datos y proporcionar APIs para la comunicación.


<div class="img-center">
    <img src="/_images/introduccion/200X.png" alt="Páginas a partir del 2000" />
</div>

<br>

El uso de tecnologías como **AJAX**, **Websockets**, **RTSP** y **WebRTC** ha permitido una comunicación en tiempo real, transformando la Web en una plataforma capaz de soportar aplicaciones más interactivas, como videollamadas, chats en vivo y juegos en línea. Además, el auge de **frameworks JavaScript** como **React**, **Vue.js** y **Angular** ha facilitado la creación de aplicaciones web modernas y escalables.

<div class="img-grid-line">
    <img src="/_images/introduccion/react.png" style="width: 200px;" alt="Frameworks de Javascript: React" />
    <img src="/_images/introduccion/vue.png" style="width: 200px;" alt="Frameworks de Javascript: Vue.js" />
    <img src="/_images/introduccion/angular.png" style="width: 200px;" alt="Frameworks de Javascript: Angular" />
</div>

En resumen, Internet y las tecnologías web han recorrido un largo camino desde sus inicios, con mejoras constantes que han permitido una mayor interactividad, velocidad y personalización en la experiencia de usuario.
