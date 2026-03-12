# Conexiones externas y Caching

Vamos a meter una petición secundaria en nuestro servidor. Por ejemplo, cuando hacemos una petición a una via, podría conectarse con un sistema de meteorología y servirnos una información enriquecida de los datos que tenemos de la vía, y de los datos enriquecidos. Para ello tenemos dos opciones:

### Modo Proactivo (Background Fetching)

En este modo, nuestro servidor tiene un proceso en segundo plano (un _worker_ como Celery, o un simple _cron job_) que se despierta cada cierto tiempo (ej. cada hora), recorre todas las vías de nuestra base de datos, pregunta al servicio de meteorología por cada una de ellas, y guarda esa previsión en nuestra propia base de datos (SQL) junto a la vía.

- **Pros:**
  - **Latencia ultrabaja:** Cuando el cliente de móvil pide la vía, nosotros solo hacemos una consulta a nuestra base de datos local y devolvemos el JSON al instante. La respuesta es inmediata.
  - **Alta disponibilidad (Resiliencia):** Si la API del tiempo se cae o está en mantenimiento, a nuestros usuarios no les afecta. Seguiremos sirviendo el último dato que guardamos hace una hora.
- **Contras:**
  - **Desperdicio de recursos:** Si tenemos 5.000 vías registradas pero hoy es martes y nadie está usando la app, estamos haciendo 5.000 peticiones por hora a la API del tiempo para nada. Podríamos agotar nuestra cuota gratuita rápidamente.
  - **Datos menos frescos:** La información siempre lleva un poco de retraso respecto a la realidad.
  - **Complejidad de infraestructura:** Requiere montar y mantener un sistema de colas o tareas programadas en el servidor.

**¿Cuándo usarlo?** Cuando la API externa cobra por petición, cuando los datos cambian muy poco a lo largo del día, o cuando la velocidad de respuesta para el usuario final es absolutamente crítica.

### Modo Reactivo (Fetch On-Demand)

Aquí no hay tareas en segundo plano. Cuando el usuario de la app móvil hace un `GET /api/v1/vias/1`, nuestro controlador de Flask detiene su ejecución, hace un `httpx.get()` a la API del tiempo, espera la respuesta, une los datos de la vía con los del clima en un DTO enriquecido (ej. `ViaWeatherDTO`) y se lo envía al usuario.

- **Pros:**
  - **Datos en tiempo real:** El usuario siempre ve la predicción exacta de ese mismo segundo.
  - **Eficiencia de peticiones:** Solo consumimos datos de la API externa para las vías que realmente le interesan a la gente en ese momento.
  - **Simplicidad:** No hay bases de datos extra ni procesos en segundo plano. Todo ocurre en el propio controlador.
- **Contras:**
  - **Latencia alta (El gran problema):** El tiempo de respuesta de nuestra API ahora es igual a _nuestro tiempo de proceso + el tiempo que tarde la API del tiempo en contestar_. Si la API externa es lenta, nuestra API será lenta.
  - **Cuellos de botella y Rate Limiting:** Si un sábado por la mañana 500 personas miran la misma vía a la vez, nuestro servidor hará 500 peticiones idénticas a la API del tiempo en el mismo segundo. Lo más probable es que la API externa nos bloquee por abuso (Error 429 Too Many Requests).

Vamos a implementar el segundo caso, sobre los controladores de vias, y vamos a hacer una petición a una API externa gratuita:

```json
// https://api.open-meteo.com/v1/forecast?latitude=40.4165&longitude=-3.7026&current_weather=true
{
  "latitude": 40.4375,
  "longitude": -3.6875,
  "generationtime_ms": 0.0413656234741211,
  "utc_offset_seconds": 0,
  "timezone": "GMT",
  "timezone_abbreviation": "GMT",
  "elevation": 651,
  "current_weather_units": {
    "time": "iso8601",
    "interval": "seconds",
    "temperature": "°C",
    "windspeed": "km/h",
    "winddirection": "°",
    "is_day": "",
    "weathercode": "wmo code"
  },
  "current_weather": {
    "time": "2026-03-12T10:00",
    "interval": 900,
    "temperature": 11.6,
    "windspeed": 3.8,
    "winddirection": 49,
    "is_day": 1,
    "weathercode": 1
  }
}
```

Definimos los DTOs necesarios, el que va a parsear el json del servicio, y lo metemos en ViaDTO

```python
from pydantic import BaseModel

class CurrentWeatherUnitsDTO(BaseModel):
    time: str
    interval: str
    temperature: str
    windspeed: str
    winddirection: str
    is_day: str
    weathercode: str

class CurrentWeatherDTO(BaseModel):
    time: str
    interval: int
    temperature: float
    windspeed: float
    winddirection: int
    is_day: int
    weathercode: int

class WeatherResponseDTO(BaseModel):
    latitude: float
    longitude: float
    generationtime_ms: float
    utc_offset_seconds: int
    timezone: str
    timezone_abbreviation: str
    elevation: float
    current_weather_units: CurrentWeatherUnitsDTO
    current_weather: CurrentWeatherDTO

# ---
class ViaDTO(BaseModel):
    id: int
    nombre: str
    grado: str
    altura: Optional[int] = None
    desplome: bool = True
    imagen: Optional[str] = None
    chapas: Optional[int] = Field(default=None, validation_alias='numero_chapas')
    user: Optional[UserDto] = None
    weather: Optional[WeatherResponseDTO] = None

    model_config = {
        'from_attributes': True,
    }
```

En nuestro controlador, esto tendría esta pinta:

```python
@vias_api_bp.route('/<int:id>', methods=['GET'])
def get_by_id(id):
    query = Via.query.get(id)

    # ask weather api
    response = requests.get("https://api.open-meteo.com/v1/forecast?latitude=40.4165&longitude=-3.7026&current_weather=true")
    if response.status_code != 200:
        error_dto = NotFoundErrorDto(error=f'Not able to connect to weather service')
        return jsonify(error_dto.model_dump()), 404

    # validation
    if query is None:
        error_dto = NotFoundErrorDto(error=f'Via with id {id} not found')
        return jsonify(error_dto.model_dump()), 404

    # add weather data to DTO
    query.weather = response.json()

    dto = ViaDTO.model_validate(query)
    return jsonify(dto.model_dump())
```

Problema que nos vamos a encontrar aquí, si recibimos muchas peticiones como hemos comentado, obligamos al programa a hacer una conexión al servicio externo cada una de las veces, y esto puede ser un cuello de botella.

## Caching

El clima no cambia cada segundo. Si sabemos que a las 10:00 AM hacen **11.6°C** en Madrid, es estadísticamente seguro asumir que a las 10:05 AM la predicción será exactamente la misma. No hay ninguna necesidad de volver a consumir la API de meteorología, gastar ancho de banda y hacer esperar al usuario.

Aquí es donde entra la **Caché**: una capa de almacenamiento de datos de alta velocidad (normalmente en memoria RAM) que almacena un subconjunto de datos, normalmente transitorios, para que las futuras solicitudes se sirvan mucho más rápido que si se accediera a la fuente de datos principal.

El patrón de Caché funciona mediante dos conceptos clave:

1. **Cache Miss (Fallo de caché):** Llega el Usuario A. Miramos en nuestra memoria a ver si tenemos el clima guardado. Como no está, hacemos la petición a Open-Meteo (reactivo), guardamos la respuesta en la caché con un tiempo de vida (TTL - _Time To Live_, por ejemplo, 15 minutos) y le devolvemos el dato al usuario.
2. **Cache Hit (Acierto de caché):** Un minuto después, llega el Usuario B. Miramos en la memoria y, ¡bingo! El dato está ahí y aún no ha caducado. Se lo devolvemos instantáneamente sin hacer ninguna petición HTTP externa.

De esta forma, si tenemos 500 peticiones en un rango de 15 minutos, **solo 1** golpeará al servicio externo. Las otras 499 se resolverán a la velocidad de la luz desde nuestra memoria local.

Para integrar el Caching hay diferentes tecnologías, tales como bases de datos en memoria como Redis o Memcached, que se usan muchísimo, y son opciones en producción. También podemos usar una caché en la RAM asociada a nuestro proceso, en lugar de usar un proceso externo como el caso de una base de datos.

Para ello vamos a usar `aiocache`. Es una libería de Python que se encarga de tener una caché, persisitr en memoria gestionar TTLs, misses, hits, ect, sin que lo tengamos que hacer nosotros. Y añadimos a Flask la capacidad de trabajar con asincronía:

```python
pip install "Flask[async]" aiocache
```

```python
import httpx
from aiocache import cached, Cache
from aiocache.serializers import JsonSerializer

@cached(ttl=900, cache=Cache.MEMORY, serializer=JsonSerializer())
async def get_weather_data():
    url = "https://api.open-meteo.com/v1/forecast?latitude=40.4165&longitude=-3.7026&current_weather=true"

    # Usamos httpx.AsyncClient en lugar de requests
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        if response.status_code == 200:
            return response.json()

    return None
```

Esta función con el decorador @cached, ya se hace el cacheo sin problema y lo almacena en memoria, y se encarga de los misses y los hits, así como de la invalidación de caché y lo integramos en nuestro controlador

```python
@vias_api_bp.route('/<int:id>', methods=['GET'])
async def get_by_id(id):
    query = Via.query.get(id)

    #
    if query is None:
        error_dto = NotFoundErrorDto(error=f'Via with id {id} not found')
        return jsonify(error_dto.model_dump()), 404

    # add
    query.weather = await get_weather_data()

    dto = ViaDTO.model_validate(query)
    return jsonify(dto.model_dump())
```

Para las peticiones usamos `await` y `async`, para que la función `get_weather_data` se ejecute correctamente y aiocache y httpx funcione bien.
