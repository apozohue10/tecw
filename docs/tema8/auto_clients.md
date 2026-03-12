# Gestión de un cliente profesional

El cliente que hemos hecho antes es un cliente funcional; para alguna petición aislada nos vale. Pero si tenemos que hacer una integración más potente de nuestro cliente de datos con el servidor, tendríamos que implementar manualmente todas las peticiones a la API y generar los DTOs correspondientes en Python basándonos en las respuestas. Para esto, ya conocemos OpenAPI y podríamos tomar nuestra especificación (`openapi.yaml`) y ponernos a escribir a mano todas las clases Pydantic y las funciones asíncronas de `httpx` que se conecten con cada una de las rutas.

Aunque en nuestro caso de ejemplo la API es pequeña y solo contiene un puñado de rutas, en entornos profesionales una API completa de un proyecto complejo puede tener cientos de rutas y varios centenares de DTOs. Además, dicha API evolucionará con el tiempo (añadiendo campos, cambiando rutas), y nuestro cliente escrito a mano quedaría obsoleto enseguida, requiriendo un mantenimiento tedioso y propenso a errores. Estaría bien poder automatizar esto.

Para ello se usan los **OpenAPI Code Generators** (Generadores de código OpenAPI).

## ¿Qué es un Generador de Código OpenAPI?

Dado que OpenAPI es un estándar estricto que define exactamente qué rutas existen, qué parámetros aceptan y qué JSONs (modelos/DTOs) devuelven, es posible crear programas que lean ese archivo YAML/JSON y escriban el código del cliente por nosotros.

Estos generadores toman tu archivo `openapi.yaml` como entrada y escupen una carpeta entera con:

1. **Modelos (DTOs):** Clases Pydantic o Dataclasses generadas automáticamente que representan las entidades de la API.
2. **Endpoints (Clientes):** Funciones o clases preparadas con `requests` o `httpx` que ya tienen las URLs, métodos y validaciones configuradas.
3. **Tipado estricto:** Todo viene con type hints, por lo que tu editor (VS Code, PyCharm) te autocompletará los campos y te avisará si intentas enviar un dato incorrecto.

Esto no es exclusivo de Python; puedes generar clientes para TypeScript (muy usado en frontend), Java, Go, Swift y decenas de lenguajes más usando la misma especificación.

### Usando un Generador en Python: `openapi-python-client`

Existen varias herramientas, siendo _OpenAPI Generator_ (el proyecto oficial basado en Java) la más famosa. Sin embargo, para el ecosistema moderno de Python (basado en `httpx` y `pydantic`), la herramienta más recomendada actualmente es `openapi-python-client`.

Vamos a ver cómo automatizar la creación de nuestro cliente en tres sencillos pasos.

### 1. Instalación de la herramienta

Primero, instalamos el generador en nuestro entorno virtual. Es recomendable instalarlo de forma global o en tu entorno de desarrollo, ya que es una herramienta de línea de comandos (CLI).

```python
pip install openapi-python-client
```

### 2. Generando el cliente

Asegúrate de tener a mano el archivo `openapi.yaml` que creamos anteriormente (o, si tu API está corriendo, la URL donde expone el JSON de Swagger, por ejemplo `http://localhost:5000/apispec_1.json`).

En tu terminal, sitúate en la carpeta donde quieres que se cree el proyecto cliente y ejecuta:

```python
openapi-python-client generate --path openapi.yaml
openapi-python-client generate --url http://127.0.0.1:3000/apispec_1.json
openapi-python-client generate --path openapi.yaml --meta none --output-path ./client/api_rocodromo
```

**¿Qué acaba de pasar?**
La herramienta habrá creado una nueva carpeta (cuyo nombre se basa en el `title` de tu especificación, por ejemplo `api-del-rocodromo-local-client`). Si entras en ella, verás que ha construido un paquete Python completo:

- Una carpeta `models/` llena de clases creadas automáticamente (tu `ViaDTO`, `NewViaDTO`, etc.).
- Una carpeta `api/` separada por "Tags" (por ejemplo `vias/`, `users/`) con funciones para hacer GET, POST, PUT, etc.
- Un archivo `client.py` que gestiona la autenticación y la configuración base.

### 3. Usando nuestro nuevo cliente autogenerado

Ahora, en lugar de escribir cadenas de texto para las URLs y diccionarios sueltos, importamos y usamos el SDK que acabamos de generar. El código queda infinitamente más limpio y robusto:

```python
import asyncio
from oapi_client.client import Client
from oapi_client.api.users import get_users

URL_BASE = "http://127.0.0.1:3000"
URL_BASE_API = f"{URL_BASE}/api/v1"

async def main():
    # Instanciamos el cliente
    client = Client(base_url=URL_BASE_API)
    # Hacemos la petición
    response = await get_users.asyncio_detailed(client=client)
    # Comprobamos la petición
    if response.status_code == 200:
        data = response.parsed.data
        headers =  response.headers
        status_code = response.status_code
        print(data)
        print(headers)
        print(status_code)

if __name__ == '__main__':
    asyncio.run(main())
```

**Beneficios inmediatos:**

- **Cero código de red (boilerplate):** Ya no tienes que lidiar con `httpx`, `raise_for_status()` ni con el manejo básico de errores. El SDK generado lo hace por ti.
- **Autocompletado:** El mayor beneficio. Al usar los DTOs autogenerados (`NewViaDTO`), tu editor de código sabe exactamente qué propiedades tiene una vía. Se acabaron los errores por escribir mal el nombre de una clave en un diccionario (`"nombree"` en vez de `"nombre"`).
- **Sincronización:** Si mañana el equipo de backend decide cambiar la API (por ejemplo, renombrar `grado` a `dificultad`), simplemente actualizas el archivo OpenAPI, vuelves a correr el comando `generate`, y tu cliente se actualiza al instante, marcando en rojo en tu editor dónde debes cambiar tu código.
