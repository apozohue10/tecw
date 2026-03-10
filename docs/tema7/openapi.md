# **Documentando nuestra API: Guía manual de OpenAPI**

Cuando programábamos aplicaciones monolíticas clásicas, la propia interfaz HTML (los `<form>`, los `<input>`) servía como documentación implícita: el usuario veía una caja que decía "Nombre de la vía" y sabía qué rellenar.

En el mundo de las APIs RESTful, no hay interfaz visual por defecto. Si le pasamos a un compañero de frontend la URL `http://localhost:5000/api/v1/vias`, su primera pregunta será: _"¿Qué verbos HTTP acepta? ¿Qué JSON te tengo que enviar en el POST? ¿Qué me vas a devolver?"_.

Para resolver esto nació **OpenAPI Specification (OAS)** (anteriormente conocido como Swagger). Es un estándar de la industria para describir de forma agnóstica (independiente del lenguaje de programación) cómo funciona una API REST.

Vamos a ver cómo crear un archivo de especificación OpenAPI desde cero, a mano, utilizando el formato **YAML** (que es mucho más legible para los humanos que escribirlo en JSON).

### La Anatomía de un documento OpenAPI

Un documento OpenAPI versión 3 (la más estándar actualmente) se divide en grandes bloques funcionales. Vamos a construir el archivo `openapi.yaml` paso a paso basándonos en nuestro ejemplo del rocódromo.

### 1. Metadatos básicos y Servidores (`openapi`, `info`, `servers`)

Lo primero es definir qué versión del estándar usamos, dar nombre a nuestra API y especificar dónde está alojada.

```yaml
openapi: 3.0.3
info:
  title: API del Rocódromo Local
  description: Servicio para gestionar las vías y bloques de escalada.
  version: 1.0.0
servers:
  - url: http://localhost:5000/api/v1
    description: Servidor local de desarrollo
  - url: https://api.rocodromolocal.com/v1
    description: Servidor de producción
```

### 2. Los Modelos de Datos (`components`)

¿Recuerdas los DTOs que hicimos con Pydantic? Aquí es donde los traducimos para que el resto del mundo los entienda. En OpenAPI, esto se define dentro de `components/schemas`.

Vamos a definir cómo es el objeto `ViaDTO` que devolvemos, y el `NewViaDTO` que pedimos para crear una vía.

```yaml
components:
  schemas:
    # Este equivale a nuestro ViaDTO de salida
    Via:
      type: object
      required: # Campos que siempre van a estar presentes
        - id
        - nombre
        - grado
        - user_id
      properties:
        id:
          type: integer
          example: 101
        nombre:
          type: string
          example: "El Techo del Mono"
        grado:
          type: string
          example: "6b+"
        altura:
          type: number
          format: float # Para especificar que permite decimales
          example: 15.5
        desplome:
          type: boolean
          default: true
        user_id:
          type: string
          format: uuid

    # Este equivale a nuestro NewViaDTO de entrada (sin ID)
    NewVia:
      type: object
      required:
        - nombre
        - grado
        - user_id
      properties:
        nombre:
          type: string
        grado:
          type: string
        altura:
          type: number
        desplome:
          type: boolean
        user_id:
          type: string
```

_Fíjate en el uso de `example`._ Es una práctica fundamental para que quien lea la documentación vea datos reales.

### 3. Los Endpoints o Rutas (`paths`)

Ahora que hemos definido nuestros "tipos de datos", vamos a documentar las rutas reales de nuestro controlador de Flask.

Documentaremos dos operaciones: un `GET` para obtener todas las vías y un `POST` para crear una nueva. Usaremos la palabra clave `$ref` (referencia) para apuntar a los esquemas que definimos en el paso anterior y no repetir código.

```yaml
paths:
  /vias:
    # Definición del método GET
    get:
      summary: Obtiene la lista de todas las vías
      tags:
        - Vías # Sirve para agrupar visualmente en la documentación
      responses:
        "200":
          description: Lista de vías recuperada con éxito.
          content:
            application/json:
              schema:
                type: array # Indicamos que devolvemos una lista
                items:
                  $ref: "#/components/schemas/Via" # Referencia al componente Via

    # Definición del método POST
    post:
      summary: Crea una nueva vía en el sistema
      tags:
        - Vías
      requestBody:
        description: Objeto JSON con los datos de la nueva vía
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/NewVia" # Referencia al componente NewVia
      responses:
        "201":
          description: Vía creada exitosamente. Devuelve la vía con su nuevo ID.
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Via"
        "400":
          description: Error de validación en los datos enviados.
```

### El resultado final y cómo visualizarlo

Si juntas todos estos bloques respetando la indentación de YAML (espacios, no tabuladores), tendrás un archivo `openapi.yaml` completo y profesional.

Pero escribir código YAML a ciegas no es muy gratificante. La verdadera magia de OpenAPI es que existen herramientas visuales que leen este archivo y generan una página web interactiva preciosa, donde los desarrolladores no solo pueden leer la documentación, ¡sino ejecutar peticiones de prueba directamente desde la pantalla!

**¿Cómo probar lo que acabamos de escribir?**

1. Abre tu navegador y ve a [editor.swagger.io](https://editor.swagger.io/).
2. Borra el código de ejemplo que viene por defecto en el panel izquierdo.
3. Pega nuestro código YAML.
4. Magia: En el panel derecho verás renderizada automáticamente tu interfaz visual, agrupada por métodos, con ejemplos interactivos.

### Integrando Swagger UI con Flasgger

Escribir el `openapi.yaml` a mano está genial para entender el estándar, pero copiar y pegar el código en el editor online de Swagger no es práctico para el día a día. Lo ideal es que nuestra propia API sirva su documentación.

Para ello, vamos a utilizar **Flasgger**, una extensión de Flask que empaqueta la interfaz gráfica de Swagger UI y la conecta con nuestras rutas.

### 1. Instalación

Primero, necesitamos instalar la librería en nuestro entorno virtual:

```yaml
pip install flasgger
```

### 2. Configurando Flasgger para leer nuestro YAML maestro

Flasgger permite documentar las rutas de muchas maneras (por ejemplo, escribiendo el YAML directamente en los comentarios o _docstrings_ de cada función).

Sin embargo, como nosotros ya hemos hecho el trabajo duro de crear un archivo `openapi.yaml` centralizado y completo, la forma más limpia es decirle a Flasgger que simplemente lea ese archivo y lo renderice.

En el archivo principal de tu aplicación (normalmente `app.py`), configuramos la extensión así:

```yaml
from flask import Flask, jsonify
from flasgger import Swagger

app = Flask(__name__)

# 1. Configuración básica para indicar que usamos el estándar OpenAPI 3
app.config['SWAGGER'] = {
    'title': 'API del Rocódromo Local',
    'uiversion': 3,
    'openapi': '3.0.3'
}

# 2. Inicializamos Swagger indicándole dónde está nuestro archivo YAML manual
# Asumimos que openapi.yaml está en la misma carpeta que app.py
swagger = Swagger(app, template_file='openapi.yaml')
```
