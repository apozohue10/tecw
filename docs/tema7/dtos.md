# RESTful API con Flask

## A por ello. Creando una RESTful API con Flask

Para ello tenemos que familiarizarnos con algunos conceptos y ponerlos en práctica con las tecnologías que ya conocemos.

- **Exponemos nuestros controladores y preparamos para JSON**: Flask (o fast-api) y tenemos nuestra capa de persistencia: SQLAlchemy (o SQLModel). Esto ya lo conocéis perfectamente.
- Las aplicaciones ya no reciben texto o emiten HTML, **todo va en JSON**. Usando jsonify.
- **Todo son DTOs**. Las aplicaciones ya no van a recibir formularios, ni a renderizar templates de HTML. Reciben DTOs, validan DTOs, convierten DTOs a modelos de SQL, convierten modelos de SQL a DTOs, y envían DTOs. Vivimos en el mundo de los DTOs. Usando Pydantic.
- No tenemos manera de testear nuestra aplicación completa con el navegador web, necesitamos un cliente para nuestra API. Usando **Postman**.
- Nuestra API tiene que estar documentada para otras aplicaciones la consuman. Creando una especificación **OpenAPI**.

Es importante saber que aunque estemos trabajando con Python y con Flask, las técnicas que vamos a aprender son transferibles a otros lenguajes con sus frameworks de ingeniería web (NodeJS - express, Go - Gin, Rust - axum, Java - Springboot, ect…)

## Exponemos nuestros controladores y preparamos para JSON

Conocemos ya los controladores en Flask. Son funciones que mediante un decorador hacen que el motor interno de Flask mapee una petición con un verbo HTTP y una ruta con una función de nuestro código. En este caso no hay ningún cambio, simplemente, en lugar de retornar HTML, retornamos JSON. Para ello tenemos que hacer dos cosas:

- Explicar al modelo SQL cómo convertirse en JSON. Es una manera sencilla de hacer un DTO.

```python
from app import db # Importamos el objeto db de la base de datos inicializado en app

class Via(db.Model):
    __tablename__ = 'via'

    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    nombre = db.Column(db.String(255), nullable=False)
    grado = db.Column(db.String(255), nullable=False)
    altura = db.Column(db.Integer, nullable=True)
    desplome = db.Column(db.Boolean, default=True)
    imagen = db.Column(db.String(120), nullable=True, unique=True)
    numero_chapas = db.Column(db.Integer, nullable=True) # Nuevo atributo
    user_id = db.Column(db.String(36), db.ForeignKey('user.id'), nullable=False) # Nuevo atributo

    def to_dict(self):
        return {
            'id': self.id,
            'nombre': self.nombre,
            'grado': self.grado,
            'altura': self.altura,
            'desplome': self.desplome,
            'imagen': self.imagen,
            'numero_chapas': self.numero_chapas,
            'user_id': self.user_id
        }
```

- Servir el JSON. El Blueprint se metería en `app.py` como de costumbre.

```python
bp_via_api = Blueprint('via_api', __name__)

@bp_via_api.route('/', methods=['GET'])
def get_all():
    query = Via.query.all()
    query_json = [via.to_dict() for via in query]
    return jsonify(query_json)
```

De modo que cuando visitamos `http://127.0.0.1:3000/api/v1/vias/`, en lugar de retornarse un HTML, se retorna esto:

```json
[
  {
    "altura": 20,
    "desplome": true,
    "grado": "6a",
    "id": 1,
    "imagen": null,
    "nombre": "Via 1",
    "numero_chapas": 8,
    "user_id": "67503d78-95fe-40cc-865c-861f6274c55b"
  },
  {
    "altura": 25,
    "desplome": false,
    "grado": "6b",
    "id": 2,
    "imagen": null,
    "nombre": "Via 2",
    "numero_chapas": 9,
    "user_id": "67503d78-95fe-40cc-865c-861f6274c55b"
  }
]
```

## DTOs. Data transfer objects

![image.png](attachment:6e87c6f0-6c00-4319-83d4-89a7fbd0a76d:image.png)

Cuando se ha dado el temario de MVC, -modelo, vista, controlador-. Se ha explicado la separación de responsabilidades. Se ha hablado de que tenemos el Modelo, que se encarga de gestionar el almacenamiento de datos. En nuestro caso el modelo lo gestionamos con el ORM SqlAlchemy. Nosotros hemos definido un modelo con una class de python:

```python
from app import db

class Via(db.Model):
    __tablename__ = 'via'

    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    nombre = db.Column(db.String(255), nullable=False)
    grado = db.Column(db.String(255), nullable=False)
    altura = db.Column(db.Integer, nullable=True)
    desplome = db.Column(db.Boolean, default=True)
    imagen = db.Column(db.String(120), nullable=True, unique=True)
    numero_chapas = db.Column(db.Integer, nullable=True)
    user_id = db.Column(db.String(36), db.ForeignKey('user.id'), nullable=False) # Nuevo atributo

  # le habíamos metido el to_dict()
```

Tenemos el controlador, que va a ser la función (o funciones) que se encarguen de recibir las peticiones, validar cosas, hablar con el modelo.

```python
@bp_via_api.route('/', methods=['GET'])
def get_all():
    query = Via.query.all()
    query_json = [via.to_dict() for via in query]
    return jsonify(query_json)
```

Y antes teníamos una UI, es decir una interfaz gráfica que renderizaba la información en HTML (getters), o nos daba acciones con formularios para modificar el estado del sistema (setters). Ahora no tenemos esto, en su lugar tenemos que definir DTOs.

Los DTOs son Data transfer objects. Son clases que nosotros vamos a definir, para explicar en el programa qué tipo de datos vamos a mostrar (cómo van a ser nuestros getters), qué tipo de datos vamos a querer recibir para crear instancias nuevas o para editar instancias existentes (setters), y otros tipos de datos auxiliares, como sistemas de filtrado, exposición de errores, métodos de autenticación, ect.

Un DTO es un objeto cuyo único propósito es transportar datos. Define exactamente qué forma debe tener el JSON de entrada (cuando un cliente nos envía datos para crear algo) y el JSON de salida (cuando nosotros devolvemos datos).

Antes lo hemos hecho con `to_dict()`, pero vamos a comprobar que se nos queda algo corto. Veamos un par de ejemplos de uso:

### Ejemplo 1. Nosotros no queremos exponer todo nuestro modelo

Si vamos al modelo de User, nos damos cuenta de que tenemos estos campos:

```python
class User(db.Model):
    __tablename__ = 'user'

    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    nombre = db.Column(db.String(255), nullable=False)
    email = db.Column(db.String(255), nullable=False, unique=True)
    password_hash = db.Column(db.String(255), nullable=False)
    role = db.Column(Enum('admin', 'user', name='user_roles'), default='user')
```

Si convierto esto a JSON con to_dict tendré que tener mucho cuidado con qué expongo, ya que no querría que mi API mostrara la contraseña del usuario (aunque esté hasheada). Necesitamos un DTO específico de salida (Ej: `UserResponseDTO`) que garantice que ciertos campos jamás viajen por la red.

```python
 class User(db.Model):
    __tablename__ = 'user'

    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    nombre = db.Column(db.String(255), nullable=False)
    email = db.Column(db.String(255), nullable=False, unique=True)
    password_hash = db.Column(db.String(255), nullable=False)
    role = db.Column(Enum('admin', 'user', name='user_roles'), default='user')

    def to_dict(self):
        return {
            'id': self.id,
            'nombre': self.nombre,
            'email': self.email,
            'role': self.role,
        }
```

### Ejemplo 2. Nostotros podríamos enriquecer la información

En algunas vistas, incluso en JSON, a lo mejor no queremos dar sólo la información granular del usuario, si no que nos gustaría mostrar algunas relaciones. Un caso sería la vía. Tenemos un ejemplo de una vía según sale del modelo:

```json
{
  "altura": 20,
  "desplome": true,
  "grado": "6a",
  "id": 1,
  "imagen": null,
  "nombre": "Via 1",
  "numero_chapas": 8,
  "user_id": "67503d78-95fe-40cc-865c-861f6274c55b"
}
```

O una representación de la vía algo más elaborada. Un DTO bien diseñado (`ViaWithUserDTO`) podría anidar la información del usuario para facilitarle la vida al cliente frontend:

```json
{
  "altura": 20,
  "desplome": true,
  "grado": "6a",
  "id": 1,
  "imagen": null,
  "nombre": "Via 1",
  "numeroChapas": 8,
  "user": {
    "id": 2,
    "nombre": "tecw",
    "email": "admin@tecw.es",
    "role": "admin"
  }
}
```

Aquí vemos la diferencia entre un objeto de un modelo, tal cual está definido en SQL, y un DTO. Hemos completado información, hemos cambiado nombres de cosas, ect…

### Ejemplo 3. Queremos definir qué entra y qué sale, y poder validarlo

Cuando vamos a crear una nueva vía, teníamos el siguiente controlador. Es un controlador que lo que hacer es tomar la información del formulario HTML y del middleware, con esta información se crea una entidad nueva, se persiste, y se hace una redirección.

```python
@via_bp.route('/', methods=['POST']) # Ruta para crear una nueva vía. Se usa el método POST
@save_file
def create(filename):
    via = Via(
        nombre=request.form.get('nombre'),
        grado=request.form.get('grado'),
        imagen=filename,
        user_id=session['user']['id']
    )
    db.session.add(via)
    db.session.commit()

    return redirect('/vias')
```

En nuestro caso sería diferente, recibiríamos un JSON (un DTO), haríamos una serie de validaciones del dicho JSON, haríamos una persistencia y retornaríamos un JSON (un DTO). Es decir, entraría un DTO así:

```json
{
  "altura": 20,
  "desplome": true,
  "grado": "6a",
  "nombre": "Via 1",
  "user_id": 2
}
```

Y retornaríamos un DTO así:

```json
{
  "altura": 20,
  "desplome": true,
  "grado": "6a",
  "id": 1,
  "imagen": null,
  "nombre": "Via 1",
  "numeroChapas": 8,
  "user": {
    "id": 2,
    "nombre": "tecw",
    "email": "admin@tecw.es",
    "role": "admin"
  }
}
```

¿Qué pasa si el cliente nos envía `"altura": "quince"` (un string en vez de un entero)? ¿O si omite el `"grado"` que es obligatorio en base de datos? Si intentamos guardar eso directamente en SQLAlchemy, la aplicación fallará con un error genérico (un 500 Internal Server Error).

Necesitamos que el DTO valide automáticamente que los tipos de datos son correctos y que los campos obligatorios están presentes. Si algo falla, el DTO debe abortar la operación y devolver al cliente un error 400 (Bad Request) estructurado:

```json
{
  "error": "ValidationError",
  "details": "El campo 'altura' debe ser un número entero."
}
```

### **El problema:**

Como vemos, son todo JSONes que entran y salen. Tenemos que tener alguna manera fácil de crear estos JSONes, validarlos, convertirlos a modelos SQL, o mandar JSON de error estructurado si es necesario.

Hacer todas estas validaciones, transformaciones y control de errores "a mano" con diccionarios de Python y muchos `if/else` es tedioso, propenso a errores y genera código espagueti.

Necesitamos una herramienta profesional que convierta JSONs en objetos de Python de forma estricta, que valide los tipos de datos automáticamente y que facilite la creación de DTOs. Es hora de que Pydantic entre al rescate.

## Pydantic al rescate

Pydantic es, de lejos, la librería de serialización y validación de datos más utilizada en Python. Ha tenido un éxito rotundo en el mundo de las APIs (es el motor detrás de FastAPI, por ejemplo), y ahora con el auge de la IA, las _function-calls_ y los agentes, su uso se ha estandarizado aún más.

Nos permite hacer cosas como:

- **Validación automática:** Usa los _type hints_ (pistas de tipado) nativos de Python. Definimos una clase que represente nuestro DTO y obtenemos gratis validadores, serializadores y deserializadores.
- **Coerción de tipos:** Si alguien envía un string (ej. `"15"`) en un campo que espera un número, Pydantic intenta corregirlo y convertirlo por debajo.
- **Errores estructurados:** Nos proporciona mensajes de error descriptivos y fáciles de procesar cuando una validación falla.
- **Integración nativa con ORMs:** Permite transformar objetos de SQLAlchemy a JSON sin apenas configuración.

```json
pip install pydantic pydantic[email] sqlalchemy
```

Vamos a crear un DTO para nuestra entidad `Via`, veremos cómo construirlo desde un JSON, cómo validarlo, generar errores estructurados y finalmente integrarlo con nuestros modelos de SQLAlchemy y nuestros controladores.

### Definiendo un DTO sencillo

Si recordamos, nuestro modelo de SQLAlchemy para `Via` tenía esta estructura:

```python
class Via(db.Model):
    __tablename__ = 'via'

    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    nombre = db.Column(db.String(255), nullable=False)
    grado = db.Column(db.String(255), nullable=False)
    altura = db.Column(db.Integer, nullable=True)
    desplome = db.Column(db.Boolean, default=True)
    imagen = db.Column(db.String(120), nullable=True, unique=True)
    numero_chapas = db.Column(db.Integer, nullable=True) # Nuevo atributo
    user_id = db.Column(db.String(36), db.ForeignKey('user.id'), nullable=False) # Nuevo atributo

```

Nuestro DTO (el objeto encargado de estructurar el JSON que entra o sale de la API) se definiría así:

```python
from typing import Optional
from pydantic import BaseModel

class ViaDTO(BaseModel):
    id: int
    nombre: str
    grado: str
    altura: Optional[float] = None
    desplome: bool = True
    imagen: Optional[str] = None
    numero_chapas: Optional[int] = None
    user_id: str
```

**Conceptos clave:**

1. **Herencia de `BaseModel`:** Igual que un modelo de SQLAlchemy hereda de `db.Model`, un DTO de Pydantic debe heredar de `BaseModel` para adquirir sus "superpoderes".
2. **Type Hints:** Pydantic aprovecha el tipado opcional de Python para saber qué tipo de dato debe exigir en cada campo (`int`, `str`, `bool`).
3. **Campos Opcionales:** Si un campo no es obligatorio, usamos `Optional[tipo] = None`.
4. **Valores por defecto:** Podemos asignar valores por defecto (ej. `desplome: bool = True`). Si el cliente no envía ese dato, Pydantic lo rellenará automáticamente.

Si probamos a instanciar este DTO en Python y lo exportamos a JSON:

```python
via_example = ViaDTO(
    id=1,
    nombre="Via de ejemplo",
    grado="6a",
    altura="20.5", # Ojo, pasamos un string pero espera un float
    user_id="123e4567-e89b-12d3-a456-426614174000"
)

print(via_example.model_dump_json(indent=4))
```

Veremos que genera el JSON perfecto. Pydantic ha inyectado los valores por defecto (`desplome: true`, `imagen: null`) y ha hecho **coerción de tipos** (convirtiendo el string `"20.5"` en el número `20.5`):

```python
{
    "id": 1,
    "nombre": "Via de ejemplo",
    "grado": "6a",
    "altura": 20.5,
    "desplome": true,
    "imagen": null,
    "numero_chapas": null,
    "user_id": "123e4567-e89b-12d3-a456-426614174000"
}
```

### Métodos vitales de un objeto Pydantic

Podemos hacer varias operaciones fundamentales con estas clases:

**1. Validar diccionarios o datos crudos:**
Si introducimos datos incorrectos, Pydantic levanta una excepción `ValidationError`.

```python
from pydantic import ValidationError

try:
    via_bad = ViaDTO(
        id=2,
        nombre="Via rota",
        grado="V2",
        numero_chapas="hola",  # Esto provocará un error, no es convertible a entero
        user_id="123e"
    )
except ValidationError as err:
    print(err.errors()) # Imprime una lista estructurada detallando qué falló
```

**2. Serialización (De Python a JSON/Diccionario):**

```python
print(via_example.model_dump()) # Devuelve un diccionario de Python
print(via_example.model_dump_json(indent=4)) # Devuelve un string en formato JSON
```

**3. Deserialización (De JSON a Python):**
Transforma un string JSON que nos envíe un cliente directamente en un objeto validado de Python.

```python
via_as_json = """
{
    "id": 3,
    "nombre": "Via desde JSON",
    "grado": "V2",
    "user_id": "123e4567"
}
"""
via_as_dto = ViaDTO.model_validate_json(via_as_json)
```

---

### Relación directa con SQLAlchemy

Hasta ahora hemos validado datos sueltos, pero nuestro objetivo es sacar modelos de la base de datos (SQLAlchemy) y mapearlos a nuestros DTOs (lo que antes hacíamos "a mano" con `to_dict()`).

Para lograrlo, solo necesitamos añadir una configuración a nuestro DTO:

```python
class ViaDTO(BaseModel):
    nombre: str
    grado: str
    altura: Optional[float] = None
    desplome: bool = True
    imagen: Optional[str] = None
    numero_chapas: Optional[int] = None
    user_id: str

    # Configuración mágica para ORMs
    model_config = {
        'from_attributes': True,
    }
```

**¿Qué hace `from_attributes=True`?** Por defecto, Pydantic intenta leer los datos como si fueran un diccionario (`datos['nombre']`). Al activar esto, le decimos a Pydantic: _"Si te paso un objeto complejo (como un modelo de SQLAlchemy), intenta leer sus atributos usando el punto (`objeto.nombre`)"_.

Veamos cómo queda nuestro controlador `GET`:

```python
@bp_via_api.route('/', methods=['GET'])
def get_all():
    vias = Via.query.all() # Obtenemos lista de modelos SQLAlchemy
    # model_validate() lee los objetos SQL y los convierte en DTOs validados
    vias_dto: list[ViaDTO] = [ViaDTO.model_validate(via) for via in vias]

    # model_dump() convierte los DTOs en diccionarios para que Flask los haga JSON
    response = [via.model_dump() for via in vias_dto]
    return jsonify(response)
```

### Enriqueciendo el DTO (Relaciones y Alias)

Podemos hacer DTOs mucho más potentes. Imagina que:

1. Queremos que el campo `numero_chapas` de la base de datos se exponga en el JSON simplemente como `chapas`.
2. Queremos embeber los datos del creador de la vía en lugar de mostrar solo su `user_id`.

```python
from pydantic import Field

# 1. Creamos un DTO para el Usuario
class UserDto(BaseModel):
    id: str
    nombre: str
    email: str
    role: str
    model_config = {'from_attributes': True}

# 2. Asumiendo que en SQLAlchemy definimos: user = db.relationship('User')

# 3. Creamos el DTO avanzado para la Vía
class ViaDTO(BaseModel):
    nombre: str
    grado: str
    altura: Optional[float] = None
    desplome: bool = True
    imagen: Optional[str] = None

    # Cambiamos el nombre en el JSON a 'chapas', pero lee de 'numero_chapas'
    chapas: Optional[int] = Field(None, validation_alias='numero_chapas')

    # Anidamos el objeto UserDto. Pydantic leerá la relación de SQLAlchemy automáticamente
    user: UserDto

    model_config = {'from_attributes': True}
```

Con esto ya tenemos un DTO de nivel profesional.
