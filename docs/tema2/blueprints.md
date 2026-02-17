# Blueprints


Los **Blueprints** en Flask son una forma de organizar y estructurar una aplicación dividiéndola en módulos reutilizables. Permiten definir partes de la aplicación de manera separada y luego registrarlas en la aplicación principal. En general, son los que toman la **responsabilidad del controlador en el patrón MVC**.

Los Blueprints permiten:

- Definir rutas y vistas específicas para un módulo.
- Aplicar middlewares específicos a ciertas partes de la aplicación.
- Registrar y agrupar rutas bajo un prefijo común.
- Definir templates, archivos estáticos y manejadores de errores específicos.


## Primeros blueprints

Imaginemos que queremos generar dos entidades nuevas para nuestro rocódromo: vías y bloques. Las vías son las rutas de escalada que se pueden encontrar en el rocódromo, mientras que los bloques son las secciones de escalada que se pueden encontrar en las vías. Cada una de estas entidades puede requerir su propio conjunto de rutas y vistas. Alguna parte del código puede ser reutilizable en ambas entidades, pero también habrá partes específicas para cada una de ellas. Un ejemplo claro son las vistas, ya que las entidades tienen necesidades diferentes de visualización en base a los datos que manejan.

El primer paso es crear en la carpeta `blueprints` dos ficheros: `via.py` y `bloque.py`. En cada uno de estos ficheros, definiremos un Blueprint que contendrá las rutas. También es necesario generar las carpetas de templates para cada uno de los Blueprints:

```plaintext
rocodromo/
│
├── app/                    # Carpeta principal de la aplicación
│   ├── models/             # Contiene la estructura de los datos. MODELO de MVC
│   ├── blueprints/         # Contiene las rutas de acceso y ciertas lógicas. CONTROLADOR de MVC
│       ├── via.py          # Controlador de las rutas de vias
│       ├── bloque.py       # Controlador de las rutas de bloques
│       ├── __init__.py     # Para importar blueprints
│   ├── public/             # Contiene ficheros que pueden accederse directamente a través de una URL
│   ├── templates/          # Contiene las vistas de la aplicación, es decir los ficheros HTML. VISTA de MVC
│       ├── via/            # Contiene las vistas para las vias
│       ├── bloque/         # Contiene las vistas para las bloques
│       ├── layout.html     # Contiene el layout base de la aplicación
│   ├── app.py              # Inicializa la aplicación y las configuraciones
├── requirements.txt        # Lista de dependencias del proyecto
└── env/                    # Entorno virtual
```

A continuación, definimos los Blueprints en los ficheros `via.py` y `bloque.py`:

**via.py**

```python
from flask import Blueprint # Se importa la clase Blueprint desde el módulo flask

via_bp = Blueprint('via', __name__, template_folder='../../templates') # Se define el Blueprint con el nombre 'via' y la carpeta de templates

@via_bp.route('/')
def list():
    return "Listado de vías"
```

**bloque.py**

```python
from flask import Blueprint

bloque_bp = Blueprint('bloque', __name__, template_folder='../templates')

@bloque_bp.route('/')
def list():
    return "Listado de bloques"
```

En estos ficheros, se define un Blueprint con el nombre `via` y `bloque` respectivamente. Se define una ruta `/` que devuelve un mensaje con el texto "Listado de vías" y "Listado de bloques" respectivamente. El Blueprint se crea con el nombre `via` y `bloque`. A parte, habría que crear una carpeta especifica para cada uno de los blueprints, ya que en el futuro, se usaran para almacenar los ficheros HTML de cada uno.

<blockquote>
<h4>Inciso: Decoradores en Python y su uso en Flask</h4>
<p>
Los <b>decoradores</b> en Python son funciones que modifican el comportamiento de otras funciones sin alterar su código fuente. Se usan comúnmente para agregar funcionalidades como logging, autorización y validación.
</p>
<p>
Los decoradores son funciones que toman otra función como argumento y devuelven una nueva función con comportamiento modificado. Se aplican utilizando el símbolo <code>@</code> antes de la función que se quiere modificar.
</p>
En Flask, los decoradores se utilizan principalmente para definir rutas pero, como veremos adelante, se pueden utilizar para más. Por ejemplo, para definir <b>middlewares</b>.
</blockquote>

---

Definimos también el fichero `__init__.py` en la carpeta `blueprints` para importar los Blueprints y registrarlos en la aplicación:

```python
from .bloque import bloque_bp
from .via import via_bp
```

Este fichero permite importar todos los blueprints a la vez en la aplicación principal. Lo cual es útil cuando se tienen muchos blueprints definidos. Sin la existencia de este fichero, sería necesario importar cada Blueprint de forma individual en el fichero `app.py`, lo cual puede ser tedioso y propenso a errores.

Una vez definidos los Blueprints, es necesario registrarlos en la aplicación principal. Para ello, importamos la carpeta blueprints que automaticamente toma el código del fichero `__init__.py` y ya se pueden usar los blueprints generados en los ficheros `via.py` y `bloque.py` dentro de `app.py` y se registran en la aplicación definiendo un **path** para cada uno de ellos através del parámetro `url_prefix`:

```python
# Import and Register the blueprints
import blueprints
app.register_blueprint(blueprints.via_bp, url_prefix='/vias')
app.register_blueprint(blueprints.bloque_bp, url_prefix='/bloques')
```

A través del navegador se puede accerder a las rutas `/vias` y `/bloques` para ver el mensaje "Listado de vías" y "Listado de bloques" respectivamente.

#### Ejercicio de clase

Desarrollar en el fichero layout.html un menú de navegación que permita acceder a las rutas de vias y bloques.


#### Ejercicio de clase

Crear un JSON con la información de vías inventadas en el fichero via.py. Modificar la ruta `/` para que devuelva la información de las vías en formato json. Para ello se deberá editar un fichero HTML en la carpeta `templates/via` que muestre la información de las vías usando Jinja2. Un ejemplo de JSON es:

```json
vias = [
    {
        "nombre": "Via 1",
        "grado": "6a",
        "longitud": 20
    },
    {
        "nombre": "Via 2",
        "grado": "6b",
        "longitud": 25
    }
]
```


---

Hasta aquí hemos podido realizar un primer acercamiento a los Blueprints en Flask. Hemos visto como podemos gestionar las rutas de nuestra aplicación de forma modular y reutilizable, pero hemos visto como gestionar solo rutas de tipo GET. En la siguiente sección veremos cómo se pueden utilizar los Blueprints para definir rutas que creen, actualicen o borren datos, pero para ello, antes se deben tener conocimiento acerca de HTTP y organizar las rutas.
