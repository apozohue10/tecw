# Ficheros privados

Flask permite también servir ficheros estáticos de forma privada. **Los ficheros privados son aquellos que, aún siendo accesibles por URL, no deberían ser descargables si no existe una autenticación o autorización previa**. Por ejemplo, con Dropbox o google drive, los usuarios pueden almacenar ficheros estáticos pero estos no son accesibles por URL si no se comparte el enlace. 

En nuestro caso, por ejemplo, los dueños del rocodromo podrían querer subir fotos de las vías o bloques que tienen instalados, pero que solo sean accesibles por los usuarios que son socios del rocodromo y poseen una cuenta de usuario en la plataforma.

Como se ha comentado antes, en el momento que colocas un fichero dentro de la carpeta public (configurada como tal en el código) este podrá ser accesible automaticamente. En temas posteriores, veremos como añadir autenticación a nuestra aplicación para proteger información sensible de los usuarios. Pero por ahora, vamos a ver como se podrían servir y crear ficheros estáticos de otra forma diferente.

## Formularios

El primer paso para ello es habilitar en los formularios de creación y edición de vías un campo que permita a los dueños del rocodromo subir una foto de la vía. Para ello, en el formulario de creación de vías, añadimos un campo de tipo `file` que permita subir una imagen. 

```html
<label for="imagen">Imagen</label>
<input type="file" class="form-control" id="file" name="file" placeholder="Upload files" autocomplete="off">
```

Además, debemos configurar el formulario para que permita subir ficheros. Para ello, en la cabecera de los formularios debemos añadir `enctype="multipart/form-data"`:

```html
<form action="/vias" method="post" enctype="multipart/form-data">
```

El parámetro `enctype="multipart/form-data"` nos permite configurar el formulario como un formulario multimedia que permite subir ficheros y datos al mismo tiempo. **Si no se configura, el formulario obvia los campos de tipo `file` y no se envían los ficheros**.

## Estructura de carpetas

Como se ha comentado previamente, no podemos almacenar estas imágenes en la carpeta public por lo que debemos crear una carpeta privada donde almacenar estas imágenes. **Crearemos dos carpetas: assets y assets/images** dentro de la carpeta app. En en el futuro, y si la aplicación escalase, se podría explorar la posibilidad de crear subcarpetas para organizar las imágenes de forma más eficiente. Pero de momento, basta con una única carpeta para las imágenes.

Para gestionar el almacenamiento y borrado de imágenes por parte de los usuarios, **necesitamos crear un middleware que se encargue de ello**. Dado que se tratan de funciones genéricas que se pueden reutilizar en cualquier parte de la aplicación, es recomendable crear un fichero al margen de los blueprints. Por ejemplo, podemos crear el fichero `handle_files.py` en la carpeta app que contenga las funciones necesarias para gestionar las imágenes. Estas funciones se importarán en los blueprints que necesiten gestionar imágenes.

Con todo esto, la estructura de las carpetas de nuestra aplicación quedaría de la siguiente forma:


```plaintext
rocodromo/
│
├── app/                    # Carpeta principal de la aplicación
│   ├── assets/             # Contiene ficheros estáticos que no pueden accederse directamente a través de una URL
│       ├── images          # Contiene imágenes de carácter privado
│   ├── models/             # Contiene la estructura de los datos. MODELO de MVC
│   ├── blueprints/         # Contiene las rutas de acceso y ciertas lógicas. CONTROLADOR de MVC
│       ├── via.py          # Controlador de las rutas de vias
│       ├── bloque.py       # Controlador de las rutas de bloques
│       ├── __init__.py     # Para importar blueprints
│   ├── public/             # Contiene ficheros que pueden accederse directamente a través de una URL
│       ├── stylesheets     # Contiene los estilos CSS divididos por páginas
│       ├── javascripts     # Contiene los ficheros javascript divididos por páginas
│       ├── images          # Contiene imágenes y logos
│       ├── favicon.ico     # Favicon de la aplicación
│   ├── templates/          # Contiene las vistas de la aplicación, es decir los ficheros HTML. VISTA de MVC
│       ├── via/            # Contiene las vistas para las vias
│       ├── bloque/         # Contiene las vistas para las bloques
│       ├── layout.html     # Contiene el layout base de la aplicación
│   ├── app.py              # Inicializa la aplicación y las configuraciones
│   ├── handle_files.py     # Middlewares para gestionar ficheros
│   ├── methodOverride.py   # Middlewares para gestionar rutas PUT y DELETE
├── requirements.txt        # Lista de dependencias del proyecto
└── env/                    # Entorno virtual
```

## Middleware para guardar imágenes

Para almacenar imágenes el primer paso es definir en nuestra aplicación en que carpeta se van a almacenar y que formatos se van a permitir. Para ello, en el fichero `app.py` añadimos las siguientes configuraciones:

```python
##### Configure file uploads
app.config['UPLOAD_FOLDER'] = 'app/assets/images'
app.config['ALLOWED_EXTENSIONS'] = {'png', 'jpg', 'jpeg'}
```
Al definir las variables mediante app.config, es como si las definieramos a nivel global de la aplicación y podemos acceder a ellas desde cualquier parte. En nuestro caso, accederemos a ellas desde el middleware que vamos a crear para gestionar las imágenes.

Para crear el middleware nos valdremos nuevamente de decoradores, al igual que hicimos con la función autoload de los blueprints. Pero en este caso, en vez de crear los decoradores en el propio blueprint, los crearemos en un fichero aparte llamado `handle_files.py` como se ha comentado previamente. Además necesitaremos otras librerías como `uuid` que nos permitirá generar un identificador único para cada imagen que se suba.

```python
import os
import uuid
from functools import wraps
from flask import request, redirect
from app import app # Importamos la aplicacion para acceder a las variables de configuración ALLOWED_EXTENSIONS y UPLOAD_FOLDER

def allowed_file(filename): # Comprueba si el fichero tiene una extensión permitida
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in app.config['ALLOWED_EXTENSIONS']

def save_file(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        file = request.files['file'] if 'file' in request.files else None # Comprueba si se ha subido un fichero
        kwargs['filename'] = None
        if file:
            if not allowed_file(file.filename): # Comprueba si la extensión del fichero es permitida
                return redirect(request.url)
            filename = f"{uuid.uuid4()}.{file.filename.rsplit('.', 1)[1].lower()}" if file else None # Genera un nombre único para el fichero
            kwargs['filename'] = filename
            file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename)) # Guarda el fichero en la carpeta de imágenes usando la libreria os
        return f(*args, **kwargs)
    return decorated_function
```

Con este middleware, se comprueba si se ha subido un fichero y si la extensión del fichero es permitida. Si se cumplen ambas condiciones, se genera un nombre único para el fichero y se guarda en la carpeta de imágenes. Además, se añade un parámetro a la función que se ejecutará después del middleware con el nombre del fichero guardado. Si el el usuario no sube un fichero, el parámetro será `None`, pero la vía se seguirá creando igualmente.

<blockquote>
<h4>Inciso: uuid</h4>
<p>
    El módulo <code>uuid</code> es una librería que permite generar identificadores únicos universales (UUID). Un UUID es un número de 128 bits que se genera aleatoriamente. El número de posibles de UUID diferentes sería de 2<sup>128</sup> por tanto la posibilidad de colisión de que se generen dos números con el mismo UUID es extremadamente baja.
<p>
</p>    
    <b>Los UUID se utilizan para identificar de forma única información en un sistema distribuido sin necesidad de coordinación centralizada</b>. Los UUID se suelen expresar con 32 carácteres en formato hexadecimal con guiones, por ejemplo: <code>550e8400-e29b-41d4-a716-446655440000</code>.
</p>
</p>    
    Así por ejemplo, en nuestro caso, generamos un UUID y le añadimos la extensión del fichero que se ha subido para garantizar que el nombre del fichero sea único.
</p>

</blockquote>
<br>

Este middleware lo podremos usar en cualquier ruta que necesite subir una imagen. Por ejemplo, en la ruta de creación de vías, podríamos usarlo de la siguiente forma:

```python
from handle_files import save_file

...

@via_bp.route('/', methods=['POST'])
@check_via
@save_file # Se añade el middleware de guardar el fichero que se ejecutará antes de la función create
def create(filename): # El middleware añade un parámetro con el nombre del fichero
    vias.append({
        'id': len(vias) + 1,
        'nombre': request.form.get('nombre'),
        'grado': request.form.get('grado'),
        'altura': float(request.form.get('altura')),
        'desplome': request.form.get('desplome') == 'true',
        'imagen': filename # Almacena el nombre del fichero
    })
    return redirect('/vias')
```

Debemos importarlos del fichero que hemos creado previamente. Al igual que ocurre con la función autoload, el middleware se ejecutará antes de la función create y añadirá un parámetro con el nombre del fichero guardado, que posteriormente almacenamos en el array.

Es probable, que necesite editar su fichero `methodOverride.py` para que el middleware de guardar ficheros funcione correctamente. Esto se debe a que, al añadir un campo de tipo `file` en el formulario, el formulario se configura como `multipart/form-data` y el método override no funciona correctamente. Para solucionarlo, debemos modificar el método override para que funcione con formularios `multipart/form-data`.


<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
from urllib.parse import parse_qs
from io import BytesIO
import cgi

class MethodOverrideMiddleware:
    allowed_methods = {'GET', 'POST', 'PUT', 'DELETE', 'PATCH'}

    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        if environ['REQUEST_METHOD'] == 'POST':
            try:
                request_body_size = int(environ.get('CONTENT_LENGTH', 0))
            except (ValueError):
                request_body_size = 0

            # Lee el cuerpo de la solicitud
            request_body = environ['wsgi.input'].read(request_body_size)
            environ['wsgi.input'] = BytesIO(request_body)

            # Verifica si el contenido es multipart/form-data
            content_type = environ.get('CONTENT_TYPE', '')
            if 'multipart/form-data' in content_type:
                # Usa cgi.FieldStorage para analizar multipart/form-data
                form = cgi.FieldStorage(fp=BytesIO(request_body), environ=environ, keep_blank_values=True)
                if '_method' in form:
                    method = form['_method'].value.upper()
                    if method in self.allowed_methods:
                        environ['REQUEST_METHOD'] = method
            else:
                # Analiza el cuerpo como application/x-www-form-urlencoded
                try:
                    form_data = parse_qs(request_body.decode('utf-8'))
                    method = form_data.get('_method', [''])[0].upper()
                    if method in self.allowed_methods:
                        environ['REQUEST_METHOD'] = method
                except UnicodeDecodeError:
                    # Si no se puede decodificar, no sobrescribas el método
                    pass

            # Reinicia el puntero del flujo para que Flask pueda leer el cuerpo
            environ['wsgi.input'].seek(0)

        return self.app(environ, start_response)
```
</div>
<div class="original" hidden>
```python
from urllib.parse import parse_qs
from io import BytesIO

class MethodOverrideMiddleware:
    allowed_methods = {'GET', 'POST', 'PUT', 'DELETE', 'PATCH'}

    def __init__(self, app):
        self.app = app

    def __call__(self, environ, start_response):
        if environ['REQUEST_METHOD'] == 'POST':
            try:
                request_body_size = int(environ.get('CONTENT_LENGTH', 0))
            except (ValueError):
                request_body_size = 0

            
            request_body = environ['wsgi.input'].read(request_body_size)
            environ['wsgi.input'] = BytesIO(request_body)
            form_data = parse_qs(request_body.decode())

            method = form_data.get('_method', [''])[0].upper()
            if method in self.allowed_methods:
                environ['REQUEST_METHOD'] = method

        return self.app(environ, start_response)
```
</div>
</div>
<br>

## Ruta para servir imágenes

Como se ha comentado previamente, las imágenes se guardan en la carpeta assets/images, que es una carpeta privada y no accesible por URL. Por tanto, necesitamos crear una ruta que nos permita servir estas imágenes. Para ello, en el fichero `app.py` añadimos la siguiente ruta:

```python
from flask import Flask, request, render_template, send_from_directory  # Importa la clase Flask desde el módulo flask
import os

...

@app.route('/download/<filename>')
def download(filename):
    return send_from_directory(os.path.join(app.root_path, 'assets/images'), filename)
```

Se usa la librería `send_from_directory` que nos permite enviar un fichero estático desde una carpeta concreta. En este caso, se envía el fichero con el nombre `filename`, obtenido del path de la URL, desde la carpeta `assets/images`. La librería `os` se utiliza para unir el path de la carpeta con el path de la aplicación.

Y para poder renderizarla en la vista, podemos añadir la siguiente línea en el fichero de `via/show.html` de la vía:

```html
{% extends "layout.html" %}

{% block content %}
    <h1>{{via.id}}</h1>
    <p> Nombre: {{via.nombre}} </p>
    <p> Grado: {{via.grado}} </p>
    <p> Altura: {{via.altura}} </p>
    <p> Desplome: {{via.desplome}} </p>
    {% if via.imagen %}
        <img src="/download/{{via.imagen}}" alt="via image">
    {% endif %}
{% endblock %}
```

Donde en `src` especificamos la URL que acabamos de crear para servir imágenes y le pasamos el nombre de la imagen que queremos mostrar.

## Middleware para borrar imágenes

Además de guardar imágenes, también necesitamos un middleware que nos permita borrarlas, ya que, cuando un usuario borre el recurso en cuestión, también deberíamos borrar la imagen asociada para que no ocupe espacio. O si el usuario sube una nueva imagen, deberíamos borrar la anterior para no acumular imágenes innecesarias. En este último caso, habrá que **añadir un campo de imagen al formulario de edición de vías** al igual que se hizo para el formulario de creación.

Para ello, creamos el siguiente middleware dentro del fichero `handle_files.py`:

```python
def delete_file(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        filename = kwargs['via'].get('imagen') # Obtiene el nombre del fichero de la vía
        if filename:
            file_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
            if os.path.exists(file_path): # Comprueba si el fichero existe
                os.remove(file_path) # Borra el fichero
        return f(*args, **kwargs)
    return decorated_function
```
Como podrá observar, el nombre del fichero se obtiene de **la vía que se ha precargado previamente gracias al autoload**. Por lo que, cuando añadamos este middleware a una ruta, deberemos asegurarnos de que **el autoloader se ejecute antes que `delete_file`**:

```python
@via_bp.route('/<viaId>/update', methods = ['POST'])
@load_via
@delete_file
@save_file
def update(viaId, via, filename):
    ...

@via_bp.route('/<viaId>/delete', methods = ['POST'])
@load_via
@delete_file
def delete(viaId, via):
    ...
```

Además, al ejecutar primero el autoloader, nos aseguramos de que el recurso existe y es accesible. Por tanto, si el recurso no existe, no se ejecutará el middleware de borrar ficheros y saltará un error 404. Para el caso de la ruta de actualización se ejecutarán los tres middlewares en orden: autoload, borrar fichero y guardar fichero. En este caso, luego **se debe actualizar el nombre del fichero para la via ya que este habra cambiado**. Para el caso de la ruta de borrado, se ejecutarán los dos middlewares en orden: autoload y borrar fichero.


### Pregunta de clase

Investigar que problema existe cuando en la interacción de actualizar los datos de un usuario.

### Pregunta de clase

¿Porqué es importante que los middlewares de gestión de ficheros los desarrollamos en un fichero distinto en vez de en el propio blueprint?

---

## Manejo de errores

Como hemos visto, servir ficheros estáticos o recursos programables mediante una API RESTful pueden producir errores al crear o editar recursos a través de un formulario. La forma de actuar hasta ahora ha sido simplemente redirigir al usuario a otra ruta. Sin embargo, en la práctica, es más conveniente mostrar un mensaje de error para que el usuario comprenda qué ha ocurrido y pueda actuar en consecuencia.

Para ello, podemos hacer uso de la función `flash` de Flask. La función `flash` permite enviar mensajes a través de las sesiones de los usuarios. Estos mensajes se almacenan en la sesión del usuario y se borran cuando el usuario cierra el navegador. 

Para añadir mensajes flash, el primer paso es añadir una secret key a la aplicación para que las sesiones sean seguras. Para ello, en el fichero `app.py` añadimos la siguiente línea:

```python
##### Configure flash key
app.secret_key = '1234'
```
Una vez configurada la secret key, podemos usar la función `flash` en cualquier ruta de la aplicación. Por ejemplo, en el middleware de guardar ficheros, podemos añadir un mensaje de error si el fichero no es permitido:

```python
from flask import request, redirect, flash
...
if not allowed_file(file.filename):
    flash('Archivo no permitido')
    return redirect(request.url)
...
```

Y finalmente, en `layout.html` podemos añadir el siguiente código que se encargará de renderizar el error en la vista:

```html
{% with messages = get_flashed_messages() %}
{% if messages %}
    {% for message in messages %}
        <div class="error">{{ message }}</div>
    {% endfor %}
{% endif %}
{% endwith %}
```

---

Con esto terminamos el tema de gestión de ficheros estáticos. Los ficheros estáticos se almacenan y si el servidor se apaga o se reinicia, no se perderán. No ocurre lo mismo con la Restful API que hemos implementado, donde los datos se almacenan en un array en memoria y se perderán si el servidor se apaga. En el siguiente tema, veremos cómo podemos almacenar los datos en una base de datos para que sean persistentes.


<script type="text/javascript">
    window.addEventListener("load", function (event) {
        let toogleButton = document.getElementsByClassName('toogleButton');
        for (let i = 0; i < toogleButton.length; i++) {
            console.log(toogleButton[i]);
            toogleButton[i].addEventListener('click', toggleCode);
        }
        function toggleCode() {
            let container = this.parentNode;
            let modificado = container.getElementsByClassName('modificado');
            let original = container.getElementsByClassName('original');
            let buttonModificado = container.getElementsByClassName('buttonModificado');
            let buttonOriginal = container.getElementsByClassName('buttonOriginal');
            if (modificado[0].hidden) {
                buttonModificado[0].classList.add('active');
                buttonOriginal[0].classList.remove('active');
                modificado[0].hidden = false;
                original[0].hidden = true;
            } else {
                buttonModificado[0].classList.remove('active');
                buttonOriginal[0].classList.add('active');
                modificado[0].hidden = true;
                original[0].hidden = false;
            }
        }
    });
</script>