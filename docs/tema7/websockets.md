# Websockets

**WebSockets es un protocolo de comunicación que permite establecer una conexión bidireccional y en tiempo real entre un cliente y un servidor sobre un único socket TCP**. A diferencia de las peticiones HTTP tradicionales, donde el cliente envía una solicitud, recibe una respuesta y cierra una conexión, WebSockets permite que tanto el cliente como el servidor envíen y reciban datos de manera continua sin necesidad de realizar múltiples solicitudes. Es decir, se mantiene un canal de comunicación abierto de tal forma que el envío de datos se produce más rápidamente y con menor latencia.

El uso de WebSockets es fundamental en aplicaciones que requieren actualizaciones en tiempo real, como:

- Chats en vivo: Permite que los mensajes se envíen instantáneamente entre usuarios.
- Notificaciones en tiempo real: Para alertas o actualizaciones instantáneas en una aplicación.
- Juegos en línea: Facilita la comunicación en tiempo real entre los jugadores y el servidor.
- Sistemas de monitoreo: Como paneles de control que muestran datos en tiempo real.
- Colaboración en línea: Aplicaciones como editores de texto compartidos o pizarras interactivas dependen de WebSockets para sincronizar cambios entre usuarios.
- Transmisión de datos en vivo: Plataformas de trading, apuestas deportivas o telemetría en aplicaciones de IoT utilizan WebSockets para ofrecer datos instantáneos.

## Gestión de eventos en Websockets

Una de las características más importantes de WebSockets es la capacidad de gestionar eventos personalizados, lo que permite una comunicación más estructurada y eficiente. En lugar de simplemente enviar y recibir mensajes sin diferenciación, los WebSockets pueden manejar eventos específicos según las necesidades de la aplicación.

- **Eventos predefinidos**: La mayoría de las implementaciones de WebSockets ofrecen eventos como open (cuando se establece la conexión), message (cuando se recibe un mensaje) y close (cuando se cierra la conexión).
- **Eventos personalizados**: Se pueden definir eventos específicos para cada aplicación, como nuevo_mensaje, usuario_conectado o actualizar_estado.
- **Broadcasting**: Permite enviar mensajes a múltiples clientes conectados simultáneamente, útil para notificaciones globales o salas de chat.
- **Autenticación y autorización**: Se pueden gestionar eventos de validación para permitir o denegar conexiones de clientes en función de credenciales o permisos.
- **Manejo de desconexiones**: WebSockets permite detectar cuándo un cliente pierde la conexión, lo que es clave para gestionar sesiones en tiempo real y reconexiones automáticas.

La estructura basada en eventos hace que WebSockets sea una herramienta poderosa para aplicaciones dinámicas y reactivas, optimizando la experiencia del usuario y reduciendo la sobrecarga del servidor en comparación con soluciones tradicionales basadas en polling o peticiones periódicas.


---

Flask, por sí solo, no incluye soporte nativo para WebSockets, pero se puede integrar mediante extensiones como Flask-SocketIO, que permite gestionar conexiones WebSocket con facilidad. Para ello, lo primero es instalar la dependencia:

```bash
pip install flask flask-socketio
```

Una vez instalado, se debe actualizar el fichero `requirements.txt` con la nueva dependencia instalada:

```bash
pip freeze > requirements.txt
```


En nuestro caso, podemos usar un websocket para mostrar ne vivo la ocupación del rocodromo. Para ello, deberemos desarrollar lo siguiente en el archivo `app.py`:

```python
from flask_socketio import SocketIO, emit
import random
import time

...

socketio = SocketIO(app)

...

# Function to emit random occupancy
def emit_occupancy():
    while True:
        occupancy = random.randint(1, 100)
        socketio.emit('update_occupancy', {'occupancy': occupancy})
        time.sleep(5)

@socketio.on('connect')
def handle_connect():
    socketio.start_background_task(emit_occupancy)

...

# Ejecutar la aplicación
if __name__ == '__main__':
    socketio.run(app, debug=True)
```

La función `emit_occupancy` que se encarga de generar un número aleatorio entre 1 y 100 cada 5 segundos y emitirlo a través del socket con el evento `update_occupancy`. Para esta tarea se han requerido 2 librerías:

- La librería `random` se utiliza para generar un número aleatorio entre 1 y 100.
- La librería `time` se utiliza para introducir una pausa de 5 segundos entre cada emisión.

Se ha creado un evento `connect` que se activa cuando un cliente se conecta al servidor. Este evento se encarga de iniciar un hilo en segundo plano que ejecuta la función `emit_occupancy`. De esta manera, la función se ejecuta de forma continua y no bloquea el hilo principal de la aplicación.

En cuanto a la última línea, estamos habilitando al servidor para escuchar peticiones http y websockets. Las peticiones de websockets siguen el estandar URL y tendrán un aspecto similar a:

```
ws://localhost:5000/socket.io/?EIO=3&transport=websocket
```

Donde:
- `ws://` indica que se trata de una conexión websocket.
- `localhost:5000` es la dirección y puerto del servidor.
- `/socket.io/` es la ruta del socket que se crea en nuestro servidor automaticamente
- `?EIO=3&transport=websocket` son parámetros que se añaden a la URL para establecer la conexión.


A continuación, generaremos un nuevo fichero html llamado `occupation.html` en la carpeta `templates` donde se mostrará la ocupación del rocódromo en tiempo real:

```html
{% extends 'layout.html' %}
{% block content %}
<body>
    <h1>Ocupación en Vivo</h1>
    <p>Ocupación actual: <span id="occupancy">0</span></p>
    <script src="https://cdn.socket.io/4.0.0/socket.io.min.js"></script>
    <script type="text/javascript">
        document.addEventListener("DOMContentLoaded", function() {
            var socket = io();

            socket.on('connect', function() {
                console.log('WebSocket connection established');
            });

            socket.on('update_occupancy', function(data) {
                document.getElementById('occupancy').innerText = data.occupancy;
            });

            socket.on('disconnect', function() {
                console.log('WebSocket connection closed');
            });

            socket.on('connect_error', function(error) {
                console.log('WebSocket error: ' + error);
            });
        });
    </script>
</body>
{% endblock %}
```

En nuestro caso, usaremos una librería externa de javascript que podemos importar dentro de nuestra página web con el comando script. En este caso, estamos importando la librería `socket.io` que nos permitirá establecer la conexión con el servidor a través de websockets. Esta librería se encargará de gestionar la conexión y los eventos que se produzcan en ella. Automaticamente establede la conexión mediante la URL que hemos definido en el servidor, es decir `ws://localhost:5000/socket.io/`.

Esta librería funciona mediante eventos, es decir, cuando se produce un evento en el servidor, se ejecuta una función en el cliente. En este caso, estamos escuchando los eventos `connect`, `disconnect`, `connect_error` y `update_occupancy`. 

Los 3 primeros son eventos predefinidos por la librería `socket.io` que se producen cuando se establece la conexión, se cierra la conexión o se produce un error en la conexión. 

El último evento, `update_occupancy`, es un evento personalizado que hemos definido en el servidor para enviar la ocupación del rocódromo. Los eventos personalizados permiten crear un identificador único para cada tipo de evento y enviar información específica asociada a ese evento. Asi de esta manera, podriamos crear más eventos (como por ejemplo la ocupación de la sala de musculación) y gestionarlos de manera independiente en el lado del cliente.

Asimismo deberemos crear la ruta en el archivo `app.py` para renderizar el fichero `occupation.html`:

```python
@app.route('/occupation')  
def occupation():
    return render_template('occupation.html')
```

Y en el archivo `layout.html` añadir un enlace a la nueva ruta en el navigation bar:

```html
<nav>
...
<a href="/occupation">Ocupacion</a>
...
</nav>
```

Con esto, podremos observar como cada 5 segundos se actualiza la ocupación del rocódromo en la página `occupation.html` en tiempo real. 