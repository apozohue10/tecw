# Middlwares

Los middlewares son funciones o clases que actúan como intermediarios entre la aplicación y el servidor, permitiendo modificar o inspeccionar las solicitudes (requests) y respuestas (responses) antes de que lleguen a su destino final. Por ejemplo, en una aplicación en Flask como la que estamos desarrollando se usa un middleware para manejar las solicitudes HTTP.

```python
@via_bp.route('/<viaId>/delete', methods=['POST'])
def delete(viaId):
```

En este caso, cuando llega una petición al servidor se va comprobando si la ruta coincide con alguna de las rutas definidas en la aplicación. Debe coincidir el path y el método HTTP (En este caso `/via/<viaId>/delete` y `POST`). Si coincide, se ejecuta la función `delete(viaId)` que esta a continuación. Este middleware se encarga también de extraer los parámetros de la URL y pasárselos a la función como parámetro de tal forma que el desarrollador no los tiene que extraer manualmente.

Al final de la sección vimos que se genero un error 404 cuando se intentaba acceder a una ruta que no existía. Este error se manejaba también a través de un middleware. Es decir, cuando se ha recorrido todas las rutas definidas en la aplicación y **no se ha encontrado ninguna que coincida con la ruta de la petición, se ejecuta este middleware** que genera un error 404. 

```python
@app.errorhandler(404)
def page_not_found(e):
    return render_template('error404.html'), 404
```

En este caso, Flask ofrece ya estos middlwares para realizar estas tareas. Pero en otros casos, se pueden definir middlewares personalizados para realizar tareas específicas como guardar ficheros, autoload de recursos, comprobación de formularios, autenticación, autorización, logging, etc.

## Middlewares en Flask

Flask no tiene un sistema de middlewares como Django, pero se pueden simular utilizando de varias maneras.

### Los decoradores: `before_request` y `after_request`

La función `before_request` se ejecutará antes de que se ejecute la función que maneja la petición y en el caso de `after_request` se ejecutará después de que se haya ejecutado la función que maneja la petición. Esta funciones se ejecutan por cada petición independientemente de la ruta a la que se dirija. 


```python
@via_bp.before_request
def validar_grado():
    if request.endpoint == "via.create" and request.method == "POST":
        grado = request.form.get("grado")
        if grado not in grades:
            abort(400)

@via_bp.after_request
def agregar_total_vias(response):
    response.headers["X-Total-Vias"] = str(len(vias))
    return response

```

La primera función se encarga de validar que el grado de la vía sea válido antes de crearla. Si no es válido, se dispara un error 400. La segunda función se encarga de añadir una cabecera con el total de vías registradas en la aplicación.

### Middlewares WSGI

A parte, los middlewares se pueden definir de otra manera. Podemos implementar uno a nivel de aplicación usando `wsgi_app`. Flask es una aplicación WSGI, lo que significa que se puede usar cualquier middleware WSGI con Flask. Para ello, simplemente hay que envolver la aplicación de Flask con el middleware WSGI. ¿Esto que quiere decir? Pues que a diferencia de, por ejemplo, `before_request`, que se ejecuta antes de cada petición, un middleware WSGI se ejecuta para cada petición pero a un nivel más bajo, es decir, antes incluso de que Flask procese la petición. Es, decir, antes de que se ejecute en nuestro programa cualquier código de Flask, se ejecuta el middleware WSGI. 

```bash
Cliente → Servidor WSGI → Middleware WSGI → Flask → Controlador
```

Y si hubiera también un before_request, el orden de ejecución sería el siguiente: 

```bash 
Cliente → Servidor WSGI → Middleware WSGI → Flask → before_request → Controlador
```


Por ejemplo, en la sección anterior sobre blueprints, vimos que teniamos que gestionar "manualmente" en los controladores el método PUT y DELETE ya que los formularios HTML solo soportan GET y POST. Para evitar esto, podemos usar un middleware WSGI que se encargue de gestionar el método override antes de que nuestro programa basado en Flask procese la petición. Para ello creamos primero un fichero methodOverride.py con el siguiente código:

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

El cual importamos y usamos luego en app.py:

```python
from methodOverride import MethodOverrideMiddleware
app.wsgi_app = MethodOverrideMiddleware(app.wsgi_app)
```

Y ahora podemos editar nuestras rutas, para que gestionen el método PUT y DELETE directamente sin tener que hacer nada en particular dentro del controlador, dejando un código mucho más limpio. Además al introducir este middleware, cada vez que necesitemos gestionar rutas del tipo PUT o DELETE, no tendremos que preocuparnos de gestionar el método override, ya que el middleware se encargará de ello.

Para actualizar una vía, el código quedaría de la siguiente manera:

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
@via_bp.route('/<viaId>', methods=['PUT'])
def update(viaId):
    for via in vias:
        if via['id'] == int(viaId):
            via = via
            break

    via['nombre'] = request.form.get('nombre') # Actualiza el nombre de la vía
    via['grado'] = request.form.get('grado') # Actualiza el grado de la vía
    return redirect('/vias')
```
</div>
<div class="original" hidden>
```python
@via_bp.route('/<viaId>/update', methods=['POST'])
def update(viaId):
    if request.form.get('_method') == 'PUT': # Comprueba si el formulario se envió con el método PUT
      for via in vias:
        if via['id'] == int(viaId):
          via = via
          break

      via['nombre'] = request.form.get('nombre') # Actualiza el nombre de la vía
      via['grado'] = request.form.get('grado') # Actualiza el grado de la vía
      return redirect('/vias')
    return "Method Not Allowed", 405 # Devuelve un error 405 si el método no es PUT
```
</div>
</div>
<br>

Para borrar una vía, el código quedaría de la siguiente manera:

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
@via_bp.route('/<viaId>', methods=['DELETE'])
def delete(viaId):
    for via in vias:
        if via['id'] == int(viaId):
            vias.remove(via)
            break
    return redirect('/vias')
```
</div>
<div class="original" hidden>
```python
@via_bp.route('/<viaId>/delete', methods=['POST'])
def delete(viaId):
    if request.form.get('_method') == 'DELETE': # Comprueba si el formulario se envió con el método DELETE
      for via in vias:
          if via['id'] == int(viaId):
              vias.remove(via)
              break
      return redirect('/vias')
    return "Method Not Allowed", 405 # Devuelve un error 405 si el método no es DELETE
```
</div>
</div>

<br>

Puede observar, que se ha suprimido el código que gestionaba el override dentro de los controladores. También se han editado los métodos de las rutas para que gestionen directamente el método **PUT y DELETE en vez de POST**.  También que se han suprimido de los paths delas rutas el sufijo `/update` y `/delete` ya que ahora se gestionan directamente con el método HTTP y por tanto ya no es necesario hacer una distinción en el path como ocurría antes. Esto implica también que hay que editar los path de los formularios en el html para que apunten a la nueva ruta.

En edit.html:

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```html
<form action="/vias/{{via.id}}" method="post"> 
    <input type="hidden" name="_method" value="PUT">
    ...
</form>
```
</div>
<div class="original" hidden>
```html
<form action="/vias/{{via.id}}/update" method="post"> 
    <input type="hidden" name="_method" value="PUT">
    ...
</form>
```
</div>
</div>

<br>

En list.html:

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```html
<form action="/vias/{{via.id}}" method="post">
    <input type="hidden" name="_method" value="DELETE" />
    ...
</form>
```
</div>
<div class="original" hidden>
```html
<form action="/vias/{{via.id}}/delete" method="post">
    <input type="hidden" name="_method" value="DELETE" />
    ...
</form>
```
</div>
</div>

<br>



De esta forma, hemos conseguido que nuestra API sea más RESTful, ya que ahora se gestionan los métodos HTTP de forma correcta. Además, el código es mucho más limpio y mantenible, ya que no tenemos que preocuparnos de gestionar el método override en cada controlador, sino que el middleware se encarga de ello.

### Otras librerías

Flask provee de una serie de extensiones (que solo tenemos que instalar) que permiten añadir middlewares a la aplicación. Estos middlewares ya cumplen una función determinada. Por ejemplo, `flask-cors` para añadir CORS a la aplicación, `flask-jwt-extended` para añadir autenticación JWT, `flask-socketio` para añadir WebSockets, etc. Un ejemplo muy interesante de esto es **SQLAlchemy, que añade un middleware para manejar las conexiones a la base de datos** como se verá más adelante. En general estos middlewares se importan y se inicializan con la aplicación de Flask creada, por ejemplo:


```python
from flask import Flask
from flask_cors import CORS

app = Flask(__name__, static_folder='public', template_folder='templates')
CORS(app)
```

---

¿Cuando habría que usar que tipo de middleware? Por lo general, lo normal es buscar librerías siempre se adapten a la necesidad que tengamos. Pero si no encontramos ninguna librería que se adapte a nuestras necesidades, entonces podemos crear nuestros propios middlewares. La siguiente pregunta podría ser ¿Es mejor usar `before_request`/`after_request` o un middleware WSGI? Pues depende de la tarea que queramos realizar. Si la tarea que queremos realizar es algo que se puede hacer a nivel de aplicación, como por ejemplo, validar un formulario, entonces es mejor usar `before_request`. Pero si la tarea que queremos realizar es algo que se tiene que hacer antes de que Flask procese la petición, como por ejemplo, gestionar el método override, entonces es mejor usar un middleware WSGI. 

En otras ocasiones, querremos gestionar tareas específicas para ciertas rutas y no para todas las del proyecto. En este caso, lo mejor es crear un middleware personalizado usando decoradores. Como veremos en la próxima sección.

## Autoload

En el tema anterior definimos una API Restful para gestionar las vías de nuestro rocódromo. Es decir, para listar, visualizar, crear, actualizar y borrar las vías. En esta API, hemos visto que repetiamos código para buscar una via en concreto cuando se trataba de visualizar, actualizar o borrar. Lo cual no es muy eficiente o mantenible. Pues bien, en este caso, **podríamos crear un middleware que se encargue de buscar la vía y que la pase a la función que maneja la petición**. De esta forma, nos ahorramos repetir código y, además, si en un futuro cambia la forma de buscar las vías, solo lo tenemos que cambiar en único sitio. A parte, para el caso de borrar, nos permite comprobar que efectivamente la via existe previamente antes de borrarla.

Para realizar esto, usaremos la librería `functools` que nos permite crear funciones de orden superior. Es decir, funciones que devuelven funciones. Y en particular usaremos el decorador `wraps` que nos permite copiar los metadatos de una función a otra.

En via.py añadiremos el siguiente código:

```python
from flask import Blueprint, request, redirect, render_template, abort # Se importa la clase abort para disparar un error 404
from functools import wraps # Se importa la función wraps para copiar los metadatos de una función a otra

def load_via(f):
    @wraps(f) # Se copian los metadatos de la función f a la función decorated_function
    def decorated_function(*args, **kwargs):
        viaId = request.view_args.get('viaId') # Se obtiene el parámetro viaId de la URL
        if viaId is not None:
            via_requested = next((via for via in vias if via['id'] == int(viaId)), None)
            if via_requested is None:
                abort(404)
            kwargs['via'] = via_requested # Se añade la vía a los argumentos de la función
        return f(*args, **kwargs) # Se llama a la función f con los argumentos y la vía
    return decorated_function
```

Este middleware se encarga de buscar la vía en la lista de vías y añadirla a los argumentos de la función que maneja la petición. Si no se encuentra la vía, se dispara un error 404 que posteriormente manejará el decorador que definimos en app.py. Para usar este middleware, simplemente se añade el decorador `@load_via` a las rutas que necesiten buscar una vía. Por ejemplo:

```python
@via_bp.route('/<viaId>', methods = ['GET'])
@load_via
def show(viaId, via): # Se añade el parámetro via que el middleware load_via ha añadido previamente
    return render_template('show.html', via=via)
```

Y esto nos permite borrar le código anterior donde buscabamos la vía en cada función (para actualizar y borrar se puede aplicar también). De esta forma, se evita repetir código y se mantiene la aplicación más limpia y mantenible.

<blockquote>
<h4>Inciso: *args y **kwargs</h4>
<p>
En Python, <code>*args</code> y <code>**kwargs</code> son utilizados para permitir que una función acepte un número variable de argumentos.
</p>
<ul>
    <li><code>*args</code> permite pasar una lista variable de argumentos posicionales a una función.</li>
    <li><code>**kwargs</code> permite pasar un número variable de argumentos de palabra clave (keyword arguments) a una función.</li>
</ul>

```python
def ejemplo(*args, **kwargs):
    print("args:", args)
    print("kwargs:", kwargs)

# Llamada a la función con argumentos posicionales y de palabra clave
ejemplo(1, 2, 3, a=4, b=5)
```

<p>Da como resultado:</p>

```python
args: (1, 2, 3)
kwargs: {'a': 4, 'b': 5}
```
</blockquote>

<br>

#### Ejercicio de clase

Añadir el middleware load_via al resto de funciones de la API Restful de vías.

#### Ejercicio de clase

Implementar un autoload para el blueprint de usuarios creado previamente y añadirlo a las funciones de la API Restful de usuarios donde sea necesario.


## Concatenar Middlewares

---

Los decoradores, como hemos visto, nos permiten añadir middlewares a nuestras funciones para evitar repetir código. Pero que también se pueden usar para otras tareas como el manejo de ficheros estáticos. Que veremos en la próxima sección.

En el ejemplo del principio, usamos la función before_request para validar el grado de la vía antes de crearla. Pero realmente **no tiene tanto sentido** usar before_request para esta tarea ya que **se ejecuta esa función para todas las peticiones que vayan a /vias**. Para ello, es mejor crear un middleware que se ejecute solo para las rutas que queramos, como pueden ser las de crear o actualizar una vía. Y además podemos añadir otras comprobaciones como que el campo del nombre no este vacio.

```python
def check_via(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        grado = request.form.get("grado")
        if grado not in grades:
            abort(400)
        nombre = request.form.get("nombre")
        if nombre is None or nombre == "":
            abort(400)
        return f(*args, **kwargs) # Se llama a la función f con los argumentos y la vía
    return decorated_function
```

Esta función dispara un error 400 Bad request si no se cumple alguna de las condiciones. Y al igual que hicimos con el error 404 podemos crear una función que maneje este error para devolver un html en particular.

Y luego podemos emplearlo en las funciones que queramos:

```python
@via_bp.route('/', methods=['POST'])
@check_via
def create():
    ...

@via_bp.route('/<viaId>/update', methods=['POST'])
@check_via
@load_via
def update(viaId, via):
    ...
```

En el caso de Update, hemos concatenado 2 middlwares, primero se ejecuta check_via y luego load_via.

---

En el siguiente tema veremos como podemos usar los middlewares también para manejar los ficheros estáticos de nuestra aplicación.



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