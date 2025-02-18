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

Flask no tiene un sistema de middlewares como Django, pero se pueden simular utilizando los decoradores: `before_request` y `after_request`. La función `before_request` se ejecutará antes de que se ejecute la función que maneja la petición y en el caso de `after_request` se ejecutará después de que se haya ejecutado la función que maneja la petición. Esta funciones se ejecutan por cada petición independientemente de la ruta a la que se dirija. 


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


Flask provee de una serie de extensiones que permiten añadir middlewares a la aplicación. Por ejemplo, `flask-cors` para añadir CORS a la aplicación, `flask-jwt-extended` para añadir autenticación JWT, `flask-socketio` para añadir WebSockets, etc. Un ejemplo muy interesante de esto es **SQLAlchemy, que añade un middleware para manejar las conexiones a la base de datos** como se verá más adelante. En general estos middlewares se importan y se inicializan con la aplicación de Flask creada, por ejemplo:


```python
from flask import Flask
from flask_cors import CORS

app = Flask(__name__, static_folder='public', template_folder='templates')
CORS(app)
```

---

A parte de todo esto, también podemos crear nuestros propios middlewares usando decoradores como veremos a continuación.

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