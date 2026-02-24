# Seeders

Los **seeders** son ficheros que se utilizan para rellenar la base de datos con datos de prueba. Se utilizan principalmente en entornos de desarrollo y pruebas para garantizar que la aplicación tenga datos con los que trabajar sin necesidad de insertarlos manualmente.

Para crear seeders en nuestra aplicación, vamos a crear un nuevo directorio llamado `seeders` dentro de la carpeta `models`. En esta carpeta generaremos un fichero por cada tabla que deseemos rellenar con datos de prueba. En nuestro caso, vamos a crear un seeder para la tabla de vias llamado `via.py`.

```python
from app import db
from models import Via  # Import your models

def seedVia():
    # Example seeding for Via model
    via1 = Via(nombre='Via 1', grado='6a', altura=20, numero_chapas=8, desplome=True, imagen=None)
    via2 = Via(nombre='Via 2', grado='6b', altura=25, numero_chapas=9, desplome=False, imagen=None)

    db.session.add(via1)
    db.session.add(via2)
    db.session.commit()

    print("Via data inserted successfully.")
```
Como puede observarse, se ha importado el modelo Via y se han usado una serie de sentencias de SQL Alchemy para añadir nuevas entradas en la tabla. Si el modelo se extiende en el futuro con nuevos atributos, estos deberán ser añadidos en el seeder.

Al igual que hicimos con los blueprints y los modelos es conveniente añadir un fichero `__init__.py` en la carpeta `seeders` para poder importar los seeders de forma más sencilla.

```python
from .via import seedVia
```

Para poder ejecutar el seeder vamos a crear un comando de ejecución de flask. Para ello, en la carpeta app.py podemos incluir el siguiente código:
    
```python
import click
from flask.cli import with_appcontext

...

# Custom command for seeding the database
@click.command(name='seed')
@with_appcontext
def seed():
    from models.seeders import seedVia
    seedVia()

app.cli.add_command(seed)
```

Se ha utilizado el paquete Click de Python que permite crear interfaces de línea de comandos. En este caso, especificamos que el comando se llamará **`seed`**. 

El Middleware **`with_appcontext`** proporcionamos el contexto de la aplicación Flask al comando, de forma que la función que ejecuta pueda acceder a la base de datos asociada a la aplicación de flask. 

Por último, **`app.cli.add_command(seed)`** añade un nuevo comando a la interfaz de línea de comandos (CLI) de la aplicación Flask. Y nos permite ejecutar lo siguiente:

```bash
flask --app app/app.py seed
```

---

Con todo esto, la estrucutra de ficheros nos queda de la siguiente manera:

```plaintext
rocodromo/
│
├── app/                        # Carpeta principal de la aplicación
│   ├── assets/                 # Contiene ficheros estáticos que no pueden accederse directamente a través de una URL
│       └── images              # Contiene imágenes de carácter privado
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

En el siguiente usaremos el ORM de SQL Alchemy para realizar un CRUD sobre nuestra Restful API.