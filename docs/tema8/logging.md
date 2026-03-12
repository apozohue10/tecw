# Logging

En el momento en el que empezamos a trabajar con varios servicios, o con métodos y utilidades externas, es conveniente tener un sistema de logging bueno. Logging es el conjunto de técnicas para tener trazas (logs) de lo que ocurre en nuestra aplicación y tienen muchísima importancia por lo siguiente:

1. **Depuración Local (Debugging):**
   Cuando desarrollamos o cuando un usuario nos reporta un error genérico (ej. "No carga la vía"), los logs son nuestros ojos dentro del servidor. En lugar de intentar adivinar qué falló mediante ensayo y error, un buen sistema de logging nos dirá exactamente dónde se rompió el flujo.
   _Ejemplo:_ Si el servicio de meteorología devuelve un error, el log nos mostrará algo como `[ERROR] 2026-03-12 11:45:00 - Fallo en get_weather_data: Timeout al conectar con open-meteo.com`. Inmediatamente sabemos que el problema no es nuestra base de datos ni nuestro código, sino que la API externa está caída.
2. **Auditoría y Seguridad (Compliance):**
   En sistemas donde los datos son sensibles o donde interactúan múltiples usuarios con diferentes permisos, es vital tener un registro inmutable de "quién hizo qué y cuándo". Esto no solo sirve para depurar, sino para responder ante incidentes de seguridad o cumplir con normativas legales.
   _Ejemplo:_ Si una vía de escalada es borrada accidentalmente, los logs de auditoría nos permiten buscar `[INFO] [AUDIT] Usuario 77b4 (admin) ejecutó DELETE /api/v1/vias/101 a las 10:22 AM desde la IP 192.168.1.45`. Esto resuelve cualquier disputa y permite tomar medidas correctivas.
3. **Depuración Distribuida y Observabilidad (OpenTelemetry - OTLP):**
   A medida que pasamos de monolitos a microservicios, una sola acción del usuario (ej. crear una vía) puede desencadenar peticiones a 3 o 4 servidores diferentes (facturación, notificaciones, clima). Si la petición falla en el paso 3, los logs tradicionales aislados en cada máquina no sirven de mucho, porque es imposible saber qué log del servidor A corresponde con qué log del servidor B.
   Aquí entra el estándar OpenTelemetry (OTel). OTel inyecta un identificador único global (`trace_id`) en la petición inicial y lo pasa de servicio en servicio.
   _Ejemplo:_ Al enviar todos los logs a un sistema centralizado como Jaeger o Datadog, podemos buscar el `trace_id: a1b2c3d4` y ver visualmente la línea de tiempo completa: la petición entró por la API principal, fue a la base de datos, luego llamó al microservicio de clima (donde se produjo un `[ERROR]`), y devolvió un 500 al cliente. Esto permite encontrar cuellos de botella y fallos en milisegundos a través de arquitecturas complejas.

## Implementando un Sistema de Logging Profesional y Observabilidad

Para pasar de la teoría a la práctica, vamos a abandonar los clásicos `print()` (que son volátiles y se pierden al cerrar la terminal) y configurar un sistema de **logging profesional** y **observabilidad** en nuestra aplicación Flask.

El objetivo es cuádruple:

1. **Logger propio**: Crear un sistema de trazas para nuestros controladores.
2. **Persistencia**: Guardar el histórico en un archivo de texto (`api.log`).
3. **Unificación**: Sincronizar el logger de Flask y Werkzeug con el nuestro.
4. **Observabilidad**: Conectar todo con **OpenTelemetry (OTel)** y **Jaeger** para monitorización gráfica y depuración distribuida.

### 1. Generando un Logger Propio

En Python, el manejo de trazas se realiza mediante la librería estándar `logging`. Para que un logger funcione, necesita **Handlers** (que definen el destino: consola, archivo, etc.) y **Formatters** (que definen la estructura del mensaje).

Es una buena práctica centralizar esta configuración en un archivo `logger.py`:

```python
import logging

# 1. Creamos nuestro Custom Logger
logger = logging.getLogger("rocodromo_api")
logger.setLevel(logging.INFO)

# 2. Definimos los destinos (Handlers)
# StreamHandler imprime en la consola (stdout)
console_handler = logging.StreamHandler()
# FileHandler almacena los logs en un fichero físico
file_handler = logging.FileHandler("api.log", encoding="utf-8")

# 3. Definimos el formato visual (Formatter)
formatter = logging.Formatter('%(asctime)s [%(levelname)s] %(name)s: %(message)s')
console_handler.setFormatter(formatter)
file_handler.setFormatter(formatter)

# 4. Vinculamos los handlers al logger
logger.addHandler(console_handler)
logger.addHandler(file_handler)
```

Ahora, en cualquier parte de nuestra aplicación, podemos importar este objeto y usarlo: `logger.info("Iniciando proceso...")`.

### 2. Modificando el Logger de Flask y Werkzeug

Flask tiene su propio logger interno, pero este depende de **Werkzeug** (el servidor WSGI subyacente que imprime las peticiones HTTP). Si no los sincronizamos, las peticiones `GET` o `POST` irán por un lado y nuestros mensajes personalizados por otro.

Para unificarlos, debemos realizar un "secuestro" de handlers en nuestro archivo `app.py`:

```python
from logger import logger
import logging

# 1. Limpiamos y reasignamos el logger de Flask
app.logger.handlers.clear()
for handler in logger.handlers:
    app.logger.addHandler(handler)
app.logger.setLevel(logging.INFO)

# 2. Atrapamos a Werkzeug (el servidor HTTP) para que use nuestra configuración
werkzeug_logger = logging.getLogger('werkzeug')
werkzeug_logger.handlers.clear()
for handler in logger.handlers:
    werkzeug_logger.addHandler(handler)
werkzeug_logger.setLevel(logging.INFO)
```

### 3. Observabilidad con OpenTelemetry y Jaeger

En arquitecturas modernas, especialmente en microservicios, no basta con leer un archivo de texto. Necesitamos saber qué camino siguió una petición a través de distintos servicios. Aquí entra **OpenTelemetry**, que asigna un `trace_id` único a cada operación.

### Instalación de dependencias:

```python
pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp opentelemetry-instrumentation-flask opentelemetry-instrumentation-logging
```

### Refinando el Logger para OTel:

Actualizamos nuestro `formatter` en `logger.py` para incluir los IDs de rastreo que inyectará OpenTelemetry:

```python
formatter = logging.Formatter(
    '%(asctime)s [%(levelname)s] %(name)s '
    '[trace_id=%(otelTraceID)s span_id=%(otelSpanID)s]: %(message)s',
    defaults={"otelTraceID": "0" * 32, "otelSpanID": "0" * 16}
)
```

### Configuración del motor de rastreo (Tracer):

Creamos un archivo de inicialización para el motor de OTel (por ejemplo, `otel_config.py`):

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.logging import LoggingInstrumentor
from opentelemetry.sdk.resources import Resource

# 1. Definimos el nombre del servicio para identificarlo en Jaeger
resource = Resource.create({"service.name": "rocodromo-backend"})
provider = TracerProvider(resource=resource)

# 2. Configuramos el exportador hacia Jaeger (puerto gRPC 4317)
otlp_exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
trace.set_tracer_provider(provider)

# 3. Instrumentamos el logging para que inyecte automáticamente los IDs
LoggingInstrumentor().instrument(set_logging_format=False)
```

Finalmente, en `app.py`, conectamos la aplicación Flask:

```python
from opentelemetry.instrumentation.flask import FlaskInstrumentor

# Instrumentación automática de rutas de Flask
FlaskInstrumentor().instrument_app(app)
```

### 4. Visualización en Jaeger

Para ver estas trazas de forma gráfica, utilizaremos **Jaeger**. La forma más rápida de levantarlo es mediante Docker:

```python
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

### ¿Cómo comprobar que funciona?

1. Arranca tu aplicación Flask (asegúrate de que `debug=False` para evitar problemas con el reloader de OTel).
2. Realiza peticiones a tus endpoints (ej. `/api/v1/vias/`).
3. Abre tu navegador en `http://localhost:16686`.
4. En la pestaña **Service**, selecciona `rocodromo-backend` y pulsa **Find Traces**.
5. Podrás ver una línea de tiempo detallada de cada petición y, dentro de cada "Span", encontrarás los logs generados durante esa ejecución específica.
