# Restful APIs

## De la Web Tradicional a las APIs RESTful

### El Contexto: Del Navegador a la Comunicación entre Máquinas

Hasta ahora hemos trabajado en el ámbito clásico de la ingeniería web. En este modelo, seres humanos (o robots indexadores) interactúan con el servidor a través de un navegador web que actúa como cliente. El patrón habitual consiste en que el servidor procesa una petición, carga datos, y devuelve HTML renderizado para mostrar la información, utilizando formularios HTML para introducir, editar o borrar datos.

Sin embargo, la ingeniería web abarca un ecosistema mucho más amplio. En muchos contextos, los clientes que se conectan a nuestros servidores no son navegadores manejados por personas, sino aplicaciones de todo tipo: aplicaciones móviles (iOS/Android), Single Page Applications (SPAs) corriendo en el navegador, u otros servidores backend que actúan como clientes de nuestros servicios.

En estos casos, necesitamos sistemas que no sirvan HTML (orientado al consumo visual humano), sino datos crudos en un formato legible por máquina (machine-readable).

### Monolitos vs Microservicios

Este cambio de paradigma nos lleva a hablar de la arquitectura de software. Cuando servimos HTML directamente desde nuestro backend, solemos tener una arquitectura monolítica. Pero al separar los datos (backend) de la presentación (frontend), abrimos la puerta a los microservicios.

En lugar de tener una aplicación gigante que hace de todo, dividimos el sistema en servicios pequeños, independientes y especializados (por ejemplo: un servicio de usuarios, un servicio de facturación, un motor de recomendaciones) que se comunican entre sí utilizando estos formatos legibles por máquina.

### Formatos de Intercambio de Datos

Históricamente, la transmisión de información entre procesos se hacía mediante mensajes de texto plano con estructuras rígidas o mediante archivos binarios.

- Los archivos binarios son extremadamente eficientes, pero ocultan los detalles de su estructura y no son legibles por humanos, requiriendo herramientas específicas para descifrarlos.
- Los archivos de texto plano iniciales tenían el problema de la falta de estandarización.

Con el tiempo, la industria evolucionó:

- XML y SOAP: Hace un par de décadas, se estandarizó el uso de XML (primo hermano del HTML) para estructurar datos, dando lugar a protocolos como SOAP. Aunque robusto, XML es muy "verboso" (ocupa mucho espacio para transmitir poca información) y complejo de parsear.
- JSON: Actualmente, es el rey indiscutible para las APIs web. Es un formato de texto ligero, menos verboso que XML, fácil de leer para los humanos y nativo para lenguajes como JavaScript.
- gRPC / Protocol Buffers: Para comunicaciones donde se requiere una eficiencia extrema (comúnmente entre microservicios internos del backend), se utilizan formatos binarios modernos como gRPC.

Hoy en día, las comunicaciones entre servicios se realizan principalmente en JSON para la gran mayoría de casos (APIs públicas, comunicación con frontend/móviles), o gRPC cuando buscamos latencias ultrabajas entre servidores.

### RESTful APIs

Una RESTful API (Representational State Transfer) es un servicio que expone una interfaz a través del protocolo HTTP. Nos comunicamos con ella basándonos en rutas o identificadores (URLs) y operaciones o verbos HTTP, definiendo el cuerpo de las peticiones (bodies) normalmente en JSON, y utilizando cabeceras (headers) para metadatos como la autenticación o el tipo de contenido.

Podemos mapear las operaciones típicas de base de datos (CRUD) directamente a los verbos HTTP de una API REST:

| Verbo HTTP | Operación CRUD | Descripción                                                  |
| ---------- | -------------- | ------------------------------------------------------------ |
| GET        | Read           | Recupera información de un recurso (ej: obtener un usuario). |
| POST       | Create         | Crea un nuevo recurso (ej: registrar un nuevo usuario).      |
| PUT        | Update         | Reemplaza un recurso existente por completo.                 |
| PATCH      | Update         | Modifica parcialmente un recurso existente.                  |
| DELETE     | Delete         | Elimina un recurso específico.                               |

### La clave de REST: Ausencia de Estado (Stateless)

A diferencia del uso de un servidor web HTML convencional con Flask, una API RESTful debe ser stateless (sin estado).

Esto no significa que el cliente no maneje información, sino que el servidor no guarda el estado de la sesión del cliente entre distintas peticiones en su memoria. No hay un objeto session tradicional guardado en el servidor. Cada petición HTTP que hace el cliente debe contener absolutamente toda la información necesaria (por ejemplo, un token de autenticación en los headers) para que el servidor la entienda y la procese de forma independiente a peticiones pasadas o futuras.

### Ejemplo 1 - Interoperabilidad

Imagina un servicio web (un agregador) que se dedica a recomendar actividades interesantes para el fin de semana. Este servicio necesita hablar con proveedores locales para consultar su oferta. Por ejemplo, podría comunicarse con nuestro rocódromo para ver qué vías y bloques tiene disponibles.

En lugar de descargar el HTML de la web del rocódromo y parsearlo manualmente (algo frágil y que no escalaría) o usar una IA para extraer los datos (lo cual consumiría demasiados recursos), le pedimos al rocódromo que exponga su información a través de una API RESTful.

El agregador (cliente) pediría la información de las vías con una simple petición HTTP:

HTTP

```shell
GET /api/v1/vias HTTP/1.1
Host: api.rocodromolocal.com
Accept: application/json
```

Y el servidor del rocódromo contestaría puramente con los datos solicitados:

HTTP

```shell
HTTP/1.1 200 OK
Content-Type: application/json

{
  "total_vias": 2,
  "vias": [
    {
      "id": 101,
      "nombre": "El Techo del Mono",
      "tipo": "bloque",
      "dificultad": "V4",
      "disponible": true
    },
    {
      "id": 102,
      "nombre": "La Fisura Infinita",
      "tipo": "deportiva",
      "dificultad": "6b+",
      "disponible": false
    }
  ]
}
```

Esta estructura de datos en **JSON** actúa como un lenguaje universal o "lengua franca". Se puede usar en cualquier aplicación, y absolutamente cualquier lenguaje de programación moderno tiene herramientas nativas para entenderlo y procesarlo.

### Ejemplo 2 - Múltiples tipos de clientes

Pensemos ahora en nuestra propia plataforma. Además del agregador externo que nos consume, nosotros tenemos:

- Una aplicación web (Frontend en React o Angular).
- Una aplicación móvil nativa para Android (escrita en Kotlin).
- Una aplicación móvil para iOS (escrita en Swift).
- Una experiencia de realidad virtual para ver las vías en 3D (desarrollada con Unity en C#).

Si usáramos el patrón tradicional de Flask devolviendo vistas renderizadas, sería una pesadilla mantener el código: tendríamos que generar HTML para la web, quizás XML para una app, y formatos a medida para el resto.

La magia de las APIs es la **separación de responsabilidades**. Es infinitamente más sencillo y escalable si nuestro servidor backend _solo_ se dedica a exponer y recibir JSON. De esta forma, construimos **una única API** que sirve como fuente de la verdad, y delegamos en cada cliente (la web, el móvil, las gafas VR) la responsabilidad de decidir cómo interpretar, renderizar y dibujar esos datos en la pantalla del usuario.

### Ejemplo 3 - Microservicios (Máquina a Máquina)

El ecosistema de las APIs no se limita a "servidor hablando con aplicación de usuario". En sistemas modernos y en **Ingeniería de Datos**, es muy común encontrar escenarios donde ni siquiera hay humanos en los extremos de la comunicación. Hablamos de procesos backend y servidores coordinándose de forma autónoma (comunicación B2B o _Machine to Machine_).

Por ejemplo, al registrar un usuario en nuestro rocódromo, nuestra API principal podría hacer peticiones en segundo plano a otros microservicios internos:

1. Envía un JSON por `POST` al **Servicio de Facturación** para que genere la suscripción mensual.
2. Envía un JSON al **Servicio de Notificaciones** para que dispare un email de bienvenida.
3. Envía los datos brutos a un **Pipeline de Ingesta de Datos** para que los analistas puedan medir métricas de registro en tiempo real.

Todos estos sistemas son piezas de software independientes que colaboran entre sí utilizando APIs RESTful como su mecanismo de comunicación estándar.
