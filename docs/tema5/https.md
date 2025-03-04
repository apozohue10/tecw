# HTTPs

**HTTPS (HyperText Transfer Protocol Secure) es la versión segura de HTTP**. Se basa en el uso de TLS (Transport Layer Security) para cifrar la comunicación entre el cliente y el servidor, garantizando confidencialidad, integridad y autenticación. El cifrado TLS protege los datos de los usuarios contra ataques de intercepción y manipulación, como el robo de información confidencial o la inyección de código malicioso. **Este cifrado se realiza antes de llegar a la capa de transporte en el modelo OSI**. Para establecer una conexión segura, se requiere un certificado SSL/TLS válido y renovado, emitido por una autoridad de certificación (CA) confiable.


Cuando un cliente accede a un servidor mediante HTTPS, el proceso de establecimiento de una conexión segura sigue estos pasos:

1. Solicitud de conexión: El cliente inicia la conexión segura enviando un "Client Hello" al servidor, incluyendo una lista de cifrados y versiones TLS compatibles.
2. Respuesta del servidor: El servidor responde con un "Server Hello", eligiendo un cifrado compatible y enviando su certificado digital.
3. Verificación del certificado: El cliente verifica el certificado del servidor, asegurándose de que es válido y está firmado por una autoridad de certificación (CA) confiable.
4. Intercambio de claves: Se genera una clave de sesión utilizando técnicas criptográficas como el intercambio de claves Diffie-Hellman o RSA.
5. Cifrado de la comunicación: A partir de este punto, los datos se transmiten de forma cifrada.
6. Transferencia de datos segura: Todas las solicitudes y respuestas entre el cliente y el servidor están protegidas contra ataques de intercepción y manipulación.


[IMAGEN DE HTTPS flow]



Implementación de HTTPS en Flask

Para habilitar HTTPS en una aplicación Flask, se pueden seguir estos pasos:

1. Obtener un certificado SSL/TLS

Para implementar HTTPS, se necesita un certificado SSL/TLS. Se puede obtener de las siguientes maneras:

- Usar un certificado de una autoridad de certificación (CA) confiable, como Let's Encrypt.
- Generar un certificado autofirmado (solo recomendado para entornos de desarrollo).

Ejemplo de generación de un certificado autofirmado:

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

Esto permite generar el certificado en sistemas tipo Ubuntu. Para Windows se debe buscar la forma equivalente de generar los certificados.

2. Configurar Flask para usar HTTPS

Una vez obtenido el certificado, crearemos una carpeta dentro de nuestro proyecto llamada certs y copiaremos los archivos cert.pem y key.pem en ella. A continuación configuramos Flask para usar HTTPS:

```python
...
if __name__ == '__main__':
    app.run(ssl_context=('certs/cert.pem', 'certs/key.pem'))
``` 

Luego para ejecutar la aplicación, simplemente ejecutamos el comando de flask sin el modo debug e indicando los certificados:

```bash
flask run --cert=app/certs/cert.pem --key=app/certs/key.pem
```

No obstante, esta opción se recomienda no usarla mientras se esta desarrollando y se recomienda volver al modo debug para ver los errores de forma más clara.

---

La estructura de ficheros nos queda de la siguiente manera:

```plaintext
rocodromo/
│
├── app/                        # Carpeta principal de la aplicación
│   ├── assets/                 # Contiene ficheros estáticos que no pueden accederse directamente a través de una URL
│       └── images              # Contiene imágenes de carácter privado
│   ├── certs/                  # Contiene los certificados SSL/TLS
│   ├── models/                 # Contiene la estructura de los datos. MODELO de MVC
│       ├── via.py              # Modelo de las vias
│       ├── __init__.py         # Para importar modelos
│       ├── migrations          # Contiene las migraciones de la base de datos
│           ├── env.py          # Configuración de la migración de Alembic que define como se ejecutan las migraciones
│           ├── script.py.mako  # Plantilla para los scripts de migración
│           ├── alembic.ini     # Configuración de Alembic
│           └── versions/       # Contiene los archivos de migración generados
│       └── seeders             # Contiene los seeders de la base de datos
│           ├── fillVia.py      # Seeder para rellenar las vias
│           └── __init__.py     # Para importar seeders
│   ├── blueprints/             # Contiene las rutas de acceso y ciertas lógicas. CONTROLADOR de MVC
│       ├── via.py              # Controlador de las rutas de vias
│       ├── bloque.py           # Controlador de las rutas de bloques
│       └── __init__.py         # Para importar blueprints
│   ├── public/                 # Contiene ficheros que pueden accederse directamente a través de una URL
│       ├── stylesheets         # Contiene los estilos CSS divididos por páginas
│       ├── javascripts         # Contiene los ficheros javascript divididos por páginas
│       ├── images              # Contiene imágenes y logos
│       └── favicon.ico         # Favicon de la aplicación
│   ├── templates/              # Contiene las vistas de la aplicación, es decir los ficheros HTML. VISTA de MVC
│       ├── via/                # Contiene las vistas para las vias
│       ├── bloque/             # Contiene las vistas para las bloques
│       └── layout.html         # Contiene el layout base de la aplicación
│   ├── app.py                  # Inicializa la aplicación y las configuraciones
│   ├── database.py             # Contiene el objeto de conexión a la base de datos
│   └── handle_files.py         # Middlewares para gestionar ficheros
├── requirements.txt            # Lista de dependencias del proyecto
└── env/                        # Entorno virtual
```


---

Generalmente, **se recomienda usar servicios como proxys inversos (Ngix, Apache) en entornos en producción para gestionar HTTPS en lugar de Flask**. Usar un proxy inverso para gestionar HTTPS en lugar de Flask mejora el rendimiento, la seguridad y la escalabilidad del servidor. Los proxies inversos, como Nginx o Apache, están optimizados para manejar miles de conexiones concurrentes con menor consumo de recursos, mientras que Flask no está diseñado para gestionar tráfico a gran escala. Además, un proxy inverso permite servir archivos estáticos de forma eficiente, reducir la carga en la aplicación y actuar como balanceador de carga en sistemas escalables.

En términos de seguridad, un proxy inverso proporciona una implementación robusta de TLS con soporte para HSTS, cifrados seguros y protección contra ataques DDoS o de fuerza bruta. También facilita la gestión de certificados SSL con herramientas como Let's Encrypt y permite redirecciones automáticas de HTTP a HTTPS. En producción, delegar HTTPS a un proxy inverso es una práctica estándar para mejorar la fiabilidad y reducir la complejidad en el servidor Flask.

Usando un proxy inverso permite que Flask **se encargue exclusivamente de la lógica de la aplicación web** como el manejo de rutas, el procesamiento de datos o la conexión con bases de datos entre otros, mientras que el proxy inverso gestiona el tráfico, la seguridad y la eficiencia del servidor.

No obstante, ciertos aspectos de seguridad siguen dependiendo de Flask como puede ser la autenticación y autorización de usuarios que veremos a continuación.