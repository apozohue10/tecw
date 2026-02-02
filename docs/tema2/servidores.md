# Servidores



## Tipos de servidores

Un servidor es un sistema que proporciona información y servicios a otros dispositivos denominados clientes. Dependiendo de los datos que se quieran ofrecer, un servidor se comunica a través de distintos métodos y protocolos. Las formas más comunes son:


<ul>
  <li><b>A través de una Página Web (HTTP/HTTPS).</b> Un servidor web (como <b>Apache</b>, <b>Nginx</b> o <b>Flask</b>) responde a las solicitudes de los usuarios con archivos <b>HTML, CSS y JavaScript</b>. Los usuarios acceden al contenido a través de un navegador web. Los clientes (navegadores por lo gneeral) realizan peticiones mediante métodos HTTP como <b>GET, POST, PUT, DELETE.</b></li>

  <li><b>Mediante una API (REST, GraphQL, SOAP).</b> Un servidor expone una API que devuelve datos en formatos como <b>JSON</b> o <b>XML</b>. Los clientes (apps, navegadores, otros servidores) realizan peticiones mediante métodos HTTP como <b>GET, POST, PUT, DELETE</b>.</li>

  <li><b>Con WebSockets para Comunicación en Tiempo Real.</b> A diferencia de HTTP, <b>WebSockets</b> permite una conexión bidireccional persistente. Se usa en aplicaciones como chats o juegos online.</li>

  <li><b>Por Archivos Descargables (FTP, SFTP, HTTP).</b> Un servidor de archivos permite que los usuarios descarguen contenido como documentos, imágenes o software. <b>FTP y SFTP</b> son protocolos comunes para la transferencia de archivos.</li>

  <li><b>Streaming de Datos (RTSP, HLS, WebRTC).</b> Para la transmisión de vídeo y audio en tiempo real, se usan protocolos como RTSP, HLS y WebRTC. <b>RTSP</b> se utiliza en cámaras de seguridad en vivo. <b>HLS</b> se emplea en plataformas como YouTube y Twitch. <b>WebRTC</b> permite videollamadas en el navegador sin necesidad de plugins.</li>

  <li><b>Correos Electrónicos (SMTP, IMAP, POP3).</b> Los servidores de correo como <b>Postfix</b> o <b>Exim</b> permiten el envío y recepción de correos electrónicos. <b>SMTP</b> se usa para el envío de correos, mientras que <b>IMAP/POP3</b> se usan para recibirlos.</li>

  <li><b>Por Bases de Datos Remotas (SQL, NoSQL).</b> Servidores como <b>MySQL, PostgreSQL o MongoDB</b> permiten consultas de datos desde clientes remotos. Se utiliza en aplicaciones web que necesitan almacenar y recuperar datos estructurados o no estructurados.</li>

  <li><b>Por Redes Peer-to-Peer (P2P, Torrent, Blockchain).</b> En lugar de depender de un único servidor, los datos se distribuyen entre múltiples nodos. <b>Torrent</b> permiten descargas descentralizadas. <b>Blockchain</b> se usa para almacenamiento de datos en criptomonedas y contratos inteligentes.</li>
</ul>

---

En esta asignatura desarrollaremos un servidor web que utilizará el protocolo HTTP para servir páginas. Además, este servidor gestionará la entrega de archivos estáticos, como imágenes, a través del mismo protocolo. También estableceremos una conexión con un servidor de bases de datos, empleando los protocolos adecuados para la comunicación. Para ello usaremos un framework de Python llamado **Flask**.