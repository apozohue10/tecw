# Seguridad en servidores web

La seguridad en servidores web es un aspecto fundamental del desarrollo de aplicaciones web. Un servidor web mal configurado o sin medidas de seguridad adecuadas puede ser vulnerable a diversos ataques, comprometiendo la integridad, confidencialidad y disponibilidad de los datos. Por ejemplo, algunas amenazas son:

- **Inyección SQL**: Un atacante puede ejecutar código SQL malicioso si los datos no son correctamente validados o parametrizados.
- **Ataques de fuerza bruta**: Intentos repetidos de autenticación para adivinar credenciales.
- **Divulgación de información**: Configuraciones incorrectas pueden exponer información sensible del servidor.
- **Cross-Site Scripting (XSS)**: Se inyecta código JavaScript en páginas vulnerables para robar información del usuario.
- **Cross-Site Request Forgery (CSRF)**: Un atacante puede engañar a un usuario autenticado para que realice acciones involuntarias.


En general, la seguridad gira entorno a la protección de los datos de los usuarios y la prevención de accesos no autorizados. Para ello, es necesario implementar medidas de seguridad en diferentes niveles, como la red, el sistema operativo, el servidor web y la aplicación. En nuestro contexto, nos centraremos en las medidas de seguridad a nivel de servidor web donde se pueden adoptar diferentes prácticas para proteger la aplicación web:

- Usar HTTPS: Protege la transmisión de datos mediante cifrado TLS. Para ello se suelen usar certificados SSL/TLS válidos y renovados.
- Validación y saneamiento de entradas: Prevenir inyecciones validando los datos que recibe el servidor. En este caso, el uso de ORMs como SQLAlchemy ayuda a prevenir inyecciones SQL.
- Uso de autenticación segura. Como uso de hashing seguro para contraseñas
- Protección contra CSRF: Utilizar tokens CSRF en formularios y solicitudes sensibles. 
- Protección frente a ataques de fuerza bruta: Limitar el número de intentos de acceso y bloquear direcciones IP sospechosas.

A lo largo de este tema veremos algunos de estos aspectos en detalle y cómo implementarlos en servidores web basados en Flask.