# Microservicios y clientes

## Contexto

Los servicios web no solo son servidores que exponen información, sino que dichos servidores, que exponen servicios, en algún momento pueden actuar como clientes de datos de otros servicios.

En la web tradicional, la relación solía ser unidireccional y estática: el navegador del usuario siempre era el cliente y nuestro código (Flask, por ejemplo) siempre era el servidor. Sin embargo, en el ecosistema moderno de las APIs RESTful y la arquitectura basada en la nube, esta línea se difumina por completo.

Nuestro servidor backend, encargado de procesar las peticiones de los usuarios, a menudo necesita información, capacidad de cómputo o funcionalidades que no posee localmente. En ese preciso instante, nuestro servidor cambia de rol: deja de ser un servidor pasivo, construye una petición HTTP (usando librerías como `requests` o `httpx` en Python) y se conecta a otra API, actuando como un cliente puro y duro.

Veamos tres ejemplos claros de cómo nuestro backend interactúa como cliente en diferentes escenarios:

### **Ejemplo 1 - Consumo de APIs Externas (Servicios de Terceros o SaaS)**

Imaginemos que en nuestro rocódromo queremos empezar a cobrar una suscripción mensual a los usuarios para que puedan acceder a instalaciones premium. Gestionar tarjetas de crédito, cumplir con normativas de seguridad (PCI DSS) y lidiar con los bancos es un dolor de cabeza gigantesco.

¿La solución? Delegar esto a un servicio especializado como Stripe o PayPal.
Cuando un usuario desde su app móvil pulsa "Pagar suscripción", la app envía una petición a nuestra API. En ese momento, **nuestra API se convierte en cliente**:

1. Nuestro servidor de Flask toma los datos básicos del usuario y construye un DTO interno.
2. Hace una petición `POST` (como cliente) a la API externa de Stripe mandando un JSON con la intención de pago.
3. Stripe valida los datos y nos devuelve un JSON con un identificador de transacción.
4. Nosotros (nuestro servidor) recibimos ese JSON, lo procesamos, guardamos el estado en nuestra base de datos, y finalmente respondemos a la app móvil de nuestro usuario.

### **Ejemplo 2 - Comunicación Interna entre Microservicios (Backend a Backend)**

A medida que la aplicación del rocódromo crece, nos damos cuenta de que nuestro código en Flask (el monolito) se ha vuelto inmanejable. Decidimos dividir el sistema en dos microservicios independientes: un "Servicio de Vías" (que gestiona las rutas, grados y chapas) y un "Servicio de Usuarios" (que gestiona el registro, login y roles). Ambos corren en servidores (o contenedores) distintos.

Si un usuario quiere crear una nueva vía, envía un JSON al _Servicio de Vías_. Pero, ¿cómo sabe este servicio si el ID del usuario que viene en el JSON realmente existe o si tiene permisos de creador?

1. El _Servicio de Vías_ detiene momentáneamente el procesamiento de su controlador.
2. Actúa como cliente y hace una petición `GET http://api-usuarios.local/v1/users/{user_id}` al _Servicio de Usuarios_.
3. El _Servicio de Usuarios_ le responde con un JSON que contiene el DTO del usuario (`UserDto`).
4. El _Servicio de Vías_ valida que el rol es correcto ("admin" o "route_setter"), guarda la vía en su propia base de datos y responde al frontend.
   Aquí vemos a dos máquinas hablando en JSON sin ninguna intervención humana en el proceso.

### **Ejemplo 3 - Webhooks (Eventos Asíncronos y Cambio de Roles)**

Existen situaciones donde la comunicación máquina a máquina no puede ser inmediata (síncrona). Volvamos al ejemplo del proveedor de pagos externos (Stripe). A veces, un pago no se confirma al instante (por ejemplo, si el banco del usuario requiere una doble verificación en su app bancaria).

En este escenario, nosotros no podemos dejar a nuestro servidor de Flask esperando (bloqueado) a que el banco conteste. Aquí entran en juego los Webhooks, que son esencialmente "APIs inversas".

1. Nosotros le decimos a Stripe: _"Aquí tienes la orden de pago. Cuando el banco confirme que tienes el dinero, avísame a esta URL: `https://api.rocodromolocal.com/v1/webhooks/pagos`"_.
2. Horas más tarde, el pago se completa.
3. Ahora **Stripe actúa como cliente** y hace una petición `POST` a nuestro servidor (que vuelve a actuar como servidor), enviándonos un JSON con los detalles del pago completado.
4. Nuestra API recibe el DTO del evento, actualiza el perfil del usuario a "Premium" y le envía un email de bienvenida.

**Conclusión del paradigma**
Para construir estas interacciones en Python, dejaremos de lado herramientas como Postman (que nos sirven a nosotros como humanos para probar) y utilizaremos librerías cliente dentro de nuestro código. Entender que tu aplicación no es el centro del universo, sino un nodo más en una inmensa red de APIs hablando entre sí en JSON, es el paso definitivo para dominar la arquitectura de software moderna.

## Creando un cliente de datos

La mayoría de lenguajes de programación y de frameworks de ingeniería web nos permiten montar servidores (Flask o FastAPI) y nos permiten tener clientes de datos, es decir, configurar un objeto que se encargue de hacer peticiones HTTP a un servidor, y comprender la respuesta, implmementando los requisitos del protocolo HTTP correctamente.

En el mundo del desarrollo backend y la analítica de datos, es fundamental que nuestro código sepa comportarse no solo como un servidor que atiende peticiones, sino también como un **cliente** que va a buscar información a otros servidores.

Vamos a ver tres formas de conseguir esto en Python, yendo desde la abstracción más alta (Pandas) hasta las herramientas de red más puras (`requests` y `httpx`).

### 1. Pandas: El enfoque de la Analítica de Datos

Para quienes vienen del mundo de la ciencia de datos, `pandas` es la herramienta estrella. Lo interesante de Pandas es que abstrae por completo la complejidad de la red. No tienes que preocuparte por configurar cabeceras ni parsear strings; le pasas una URL y él hace la petición HTTP por debajo, descarga los datos y los convierte directamente en un `DataFrame` (una tabla).

```python
import pandas as pd

# Supongamos que esta URL expone un JSON con las vías de nuestro rocódromo
url_api = "https://api.rocodromolocal.com/v1/vias"

# Pandas abre un cliente HTTP interno, hace un GET, lee el JSON y lo mapea
df_vias = pd.read_json(url_api)

# Ya podemos usar toda la potencia de Pandas para analizar los datos
print(df_vias.head())
print(f"La altura media de las vías es: {df_vias['altura'].mean()} metros")
```

### **2. `requests`: El estándar de la industria (Síncrono)**

La librería `requests` ha sido durante años la forma "Pythonica" por excelencia de hacer peticiones HTTP. Es increíblemente fácil de usar, pero tiene una característica clave: es **síncrona** (bloqueante). Esto significa que cuando tu código hace un `requests.get()`, la ejecución del programa se detiene por completo hasta que el servidor remoto contesta.

```python
import requests

url_api = "https://api.rocodromolocal.com/v1/vias"

# 1. Hacemos la petición GET (nuestro programa se detiene aquí hasta recibir respuesta)
response = requests.get(url_api)

# 2. Comprobamos el código de estado (200 OK significa que todo fue bien)
if response.status_code == 200:
    # 3. requests tiene un método nativo para transformar el texto JSON en un diccionario de Python
    data = response.json()

    # Aquí podríamos inyectar este diccionario en nuestros DTOs de Pydantic
    print(f"Se han recuperado {len(data)} vías.")
    print(data[0]['nombre']) # Accedemos al primer elemento
else:
    print(f"Error al conectar con la API. Código HTTP: {response.status_code}")
```

### **3. `httpx`: El futuro moderno (Asíncrono y HTTP/2)**

A medida que las aplicaciones web evolucionan (especialmente con frameworks como FastAPI), necesitamos que nuestro código no se quede "congelado" esperando a que otro servidor responda. `httpx` es una librería moderna inspirada en la sintaxis de `requests`, pero construida desde cero para soportar **asincronía** (`async` / `await`) y protocolos modernos como HTTP/2. Nos permite hacer peticiones síncronas igual que `requests`, pero su verdadero poder brilla cuando lo usamos de forma asíncrona:

```python
import httpx
import asyncio

async def obtener_datos_rocodromo():
    url_api = "https://api.rocodromolocal.com/v1/vias"

    # Usamos un AsyncClient como gestor de contexto (cierra la conexión automáticamente)
    async with httpx.AsyncClient() as client:
        # La palabra 'await' permite que el hilo de Python haga otras cosas mientras llega la respuesta
        response = await client.get(url_api)

        # Validamos que la respuesta sea exitosa (lanza excepción si es un error 4xx o 5xx)
        response.raise_for_status()

        # Parseamos el JSON a diccionario
        datos_vias = response.json()
        print("Datos obtenidos de forma asíncrona exitosamente.")
        return datos_vias

# Para ejecutar la función asíncrona en un entorno normal de Python:
# asyncio.run(obtener_datos_rocodromo())
```

## Montemos un cliente de datos

Creamos una carpeta hermana de `/app`, que llamamos `client`, y dentro crearemos un archivo `client.py`. En lugar de depender de herramientas visuales como Postman, vamos a construir un script en Python que actúe como un cliente autónomo. Este script se comunicará con la API RESTful que hemos construido previamente.

Para ello, utilizaremos `httpx` para las peticiones HTTP y `asyncio` para gestionar la asincronía.

Vamos a desgranar los conceptos necesarios paso a paso antes de ver el código final:

**1. Configuración Base y Rutas**
Lo primero es definir dónde "vive" nuestro servidor. Al igual que en frontend extraemos las URLs a variables de entorno o constantes, en nuestro cliente hacemos lo mismo:

```python
URL_BASE = "http://127.0.0.1:3000"
URL_BASE_API = f"{URL_BASE}/api/v1"
```

De esta forma, si el día de mañana pasamos nuestro servidor de local a producción, solo tendremos que cambiar una línea de código.

**2. Asincronía y el Cliente HTTP (`async with`)**
Fíjate que todas nuestras funciones empiezan con `async def`. Esto le dice a Python que son rutinas asíncronas.
Dentro de ellas, usamos `async with httpx.AsyncClient() as client:`. Esto abre una "sesión" o cliente HTTP optimizado. Al usar `with` (un gestor de contexto), nos aseguramos de que los recursos de red se cierren y limpien automáticamente en cuanto la petición termine, evitando fugas de memoria.

**3. Peticiones GET y Deserialización**
Para obtener todas las vías o una en concreto, usamos `await client.get(url)`. La palabra clave `await` es la magia de la asincronía: le dice a Python _"envía la petición por la red y, mientras el servidor de Flask procesa y responde, tú puedes ir a hacer otras tareas"_.
Una vez llega la respuesta, ejecutamos `response.json()`. Esto toma el texto plano en formato JSON que devuelve nuestra API y lo convierte automáticamente en listas y diccionarios nativos de Python.

**4. Peticiones POST: Enviando JSON**
Cuando queremos crear una vía nueva (`create_via`), la API espera recibir datos. En lugar de mandar un formulario HTML tradicional, le pasamos un diccionario de Python al parámetro `json` de `httpx`:
`await client.post(url_via, json=new_via)`
Al usar el parámetro `json=`, `httpx` hace dos cosas por nosotros: convierte nuestro diccionario de Python a una cadena de texto JSON válida y añade automáticamente la cabecera `Content-Type: application/json` para que el backend lo entienda.

**5. Gestión de Errores (Resiliencia)**
El código "feliz" asume que el servidor siempre está encendido y que nuestros datos siempre son correctos. Pero en el mundo real, los sistemas fallan. Si enviamos un JSON inválido, nuestro DTO de Pydantic en el backend lanzará un error 400. Si el servidor está apagado, la red fallará.
Para evitar que nuestro cliente explote ("crashee") abruptamente, usamos la línea `response.raise_for_status()` combinada con bloques `try...except`.
`raise_for_status()` revisa el código HTTP de respuesta; si es un código de éxito (200, 201), no hace nada. Si es un error (404, 400, 500...), lanza una excepción que podemos capturar.

Las dos excepciones más importantes de `httpx` que debemos gestionar son:

- `httpx.HTTPStatusError`: La petición llegó al servidor, pero el servidor nos contestó con un error (ej. 404 Not Found porque pedimos una vía que no existe).
- `httpx.RequestError`: La petición ni siquiera llegó al servidor (ej. no hay internet o el servidor Flask no está arrancado).

**El Código Final (client.py)**
Juntando todos estos conceptos y aplicando la gestión de errores, nuestro cliente profesional quedaría así:

```python
import httpx
import asyncio

URL_BASE = "http://127.0.0.1:3000"
URL_BASE_API = f"{URL_BASE}/api/v1"

async def get_vias():
    url_vias = f"{URL_BASE_API}/vias/"
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url_vias)
            response.raise_for_status() # Comprueba si hay errores HTTP (4xx, 5xx)
            vias = response.json()
            print("Lista de vías recuperada con éxito.")
            return vias
        except httpx.HTTPStatusError as exc:
            print(f"Error del servidor ({exc.response.status_code}) al pedir vías.")
        except httpx.RequestError as exc:
            print(f"Error de red conectando a {exc.request.url}")

async def get_via_by_id(via_id: int = 1):
    url_via = f"{URL_BASE_API}/vias/{via_id}"
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url_via)
            response.raise_for_status()
            via = response.json()
            print(f"Vía {via_id} recuperada con éxito.")
            return via
        except httpx.HTTPStatusError as exc:
            if exc.response.status_code == 404:
                print(f"La vía con ID {via_id} no existe en la base de datos.")
            else:
                print(f"Error HTTP {exc.response.status_code}.")
        except httpx.RequestError as exc:
            print(f"🔌 Error de red: {exc}")

async def create_via(new_via: dict):
    url_via = f"{URL_BASE_API}/vias/"
    async with httpx.AsyncClient() as client:
        try:
            # Enviamos el diccionario como payload JSON
            response = await client.post(url_via, json=new_via)
            response.raise_for_status()
            via = response.json()
            print("Nueva vía creada correctamente.")
            return via
        except httpx.HTTPStatusError as exc:
            print(f"Error al crear vía. El servidor rechazó los datos ({exc.response.status_code}).")
            # Muy útil para ver qué validación falló en Pydantic en el backend
            print(f"Detalle del servidor: {exc.response.json()}")
        except httpx.RequestError as exc:
            print(f"Error de red: {exc}")

# Agrupamos la ejecución asíncrona en una función main
async def main():
    print("--- Obteniendo todas las vías ---")
    await get_vias()

    print("\n--- Obteniendo una vía específica ---")
    await get_via_by_id(1)

    print("\n--- Forzando un error (vía que no existe) ---")
    await get_via_by_id(9999)

    print("\n--- Creando una vía nueva ---")
    await create_via({
        "nombre": "Mi nueva super vía",
        "grado": "7a",
        "user_id": "f260ceed-234a-45e2-8c4f-637a34e5189d" # Reemplazar por un UUID real de tu BD
    })

if __name__ == '__main__':
    # asyncio.run arranca el bucle de eventos (Event Loop) que maneja las funciones async
    asyncio.run(main())
```

Esto se puede aplicar también en nuestro servidor.
