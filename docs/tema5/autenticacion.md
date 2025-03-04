# Gestión de identidades

Si queremos permitir que los usuarios gestionen sus propios recursos, como que los administradores del rocódromo puedan crear y modificar rutas, o que los escaladores guarden sus rutas favoritas, es fundamental **implementar un sistema de gestión de identidades**.

Este sistema garantizará que cada usuario solo pueda acceder y gestionar la información que le corresponde, evitando modificaciones no autorizadas. Dado que se manejarán datos sensibles, como credenciales de acceso y preferencias personales, la autenticación debe ser segura y robusta. Para ello, es necesario utilizar buenas prácticas como el almacenamiento seguro de contraseñas mediante hashing. En esta sección nos centraremos en la autenticación de usuarios y en la gestión de sesiones.

## Modelo de usuarios

Para realizar poder desarrollar un sistema de autenticación, es necesario **definir un modelo de usuarios** que permita almacenar la información necesaria para identificar y gestionar a los usuarios. Para ello generamos un nuevo modelo en el fichero `user.py`:

```python
import uuid
from app import db  # Importamos el objeto db de la base de datos inicializado en app
from sqlalchemy import Enum
from werkzeug.security import generate_password_hash, check_password_hash

class User(db.Model):
    __tablename__ = 'user'

    id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
    nombre = db.Column(db.String(255), nullable=False)
    email = db.Column(db.String(255), nullable=False, unique=True)
    password_hash = db.Column(db.String(255), nullable=False)
    role = db.Column(Enum('admin', 'user', name='user_roles'), default='user')

    def set_password(self, password):
        """Genera el hash de la contraseña y lo almacena."""
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        """Verifica si la contraseña ingresada es correcta."""
        return check_password_hash(self.password_hash, password)
```

En este código hemos definido varios atributos para el modelo `user`:

- `id`: Identificador único del usuario. Es un tipo String de 36 carácteres que se genera automáticamente al crear un nuevo usuario. Este String es en realidad un uuid como los vistos al almacenar ficheros. En este caso se usa uuid en vez de un entero autoincremental para evitar que un atacante pueda adivinar los identificadores de otros usuarios.
- `nombre`: Nombre del usuario.
- `email`: Correo electrónico del usuario. Se utiliza como identificador único para garantizar que no existan usuarios duplicados. El email es el que se usará para el login.
- `password_hash`: Hash de la contraseña del usuario. En lugar de almacenar la contraseña en texto plano, se almacena su hash para garantizar la seguridad de los datos. 
- `role`: Es un campo de tipo Enum que solo puede tener dos valores: 'admin' o 'user'. Por defecto, los usuarios no son administradores. Esto se usará sobre todo en la sección de autorización. 

Además, hemos definido dos métodos para el modelo `Usuario` que se usarán cuando se registre un nuevo usuario o cuando se realice el login:

- `set_password`: Este método recibe la contraseña en texto plano, genera su hash y lo almacena en el atributo `password_hash`.
- `check_password`: Este método recibe una contraseña en texto plano y verifica si coincide con el hash almacenado en `password_hash`. Devuelve `True` si la contraseña es correcta y `False` en caso contrario. Basicamente compara los hash de la contraseña introducida por el usuario al hacer log in con el hash almacenado en la base de datos.


Una vez creado el modelo usuario, **debemos crear la correspondiente migración y ejecutarla** como se ha visto en el tema anterior, así como los **seeders**.

El siguiente paso es crear un blueprint de los usuarios al igual que hicimos para el recurso de vías. Para ello, creamos un nuevo fichero `user.py` en el directorio `blueprints` y añadimos las funciones necesarias para realizar una **RESTFul API de usuarios usando el modelo creado previamente**. Es decir, las funciones de load, list, new, create, show, edit, update. Este blueprint se debe registrar en el fichero `app.py` para que la aplicación pueda acceder a él usando la ruta `/users`. Algunas de esas rutas **podrán ser solo usadas por el administrador**, como la de listar o crear  usuarios, pero esto se verá en la sección de autorización.




## Blueprint de gestión de identidades

Para poder implementar una sistema de gestión de identidades debemos crear primero un par de interfaces: una para el login y otra para el registro de nuevos usuarios. Para ello, creamos un nuevo blueprint en el fichero `auth.py` y una carpera `auth` dentro de `templates` que contendrá las rutas necesarias para realizar estas operaciones. 

```python
from flask import Blueprint, request, redirect, render_template, session, flash, abort # Se importa la clase Blueprint desde el módulo flask
from models import User
from app import db

auth_bp = Blueprint('auth', __name__, template_folder='../templates/auth')

...
```

Este blueprint lo registraremos en app.py para que la aplicación pueda acceder a él usando la ruta `/auth`. Se ha importado la libreria `flash` que nos permitirá mostrar mensajes de error o de éxito en la interfaz de usuario. Y en particular se ha importado la libreria session que nos permitirá gestionar la sesión del usuario. 

<blockquote>
<h4>Inciso: Sesiones</h4>
<p>
    Una sesión en el contexto de una aplicación web es un mecanismo que permite mantener el estado de un usuario a lo largo de múltiples solicitudes HTTP. Dado que HTTP es un protocolo sin estado, las sesiones permiten que el servidor identifique a un usuario y almacene información temporalmente mientras interactúa con la aplicación.
</p>
<p>

</p>
<h6>Funcionamiento de sesiones:</h6>
<ol>
<li><b>Inicio de sesión</b>: Cuando un usuario se autentica, el servidor crea una sesión y le asigna un identificador único.</li>
<li><b>Almacenamiento de datos</b>: La sesión puede contener información relevante como el ID del usuario, roles o preferencias.</li>
<li><b>Seguimiento</b>: En cada petición, el cliente envía el identificador de sesión al servidor para mantener la persistencia del estado.</li>
<li><b>Cierre de sesión</b>: Al finalizar la sesión, los datos almacenados se eliminan del servidor.</li>
</ol>
<h6>Métodos para gestionar sesiones:</h6>
<ol>
<li><b>Cookies</b>: Se almacena un identificador de sesión en el navegador del usuario.</li>
<li><b>Token JWT (JSON Web Token)</b>: Se usa un token cifrado para mantener la sesión sin necesidad de almacenar estado en el servidor.</li>
<li><b>Almacenamiento en base de datos</b>: Se guarda información de la sesión en una base de datos para mayor persistencia y seguridad.</li>
<li><b>Redis/Memcached</b>: Se utiliza una base de datos en memoria para sesiones rápidas y escalables.</li>
</ol>

<p>
Se usará el almacenamiento por defecto de Flask, es decir usando una  cookie encriptada (de ahí la necesidad de configurar la secret_key) en el navegador del usuario.
</p>

</blockquote>
<br>

### Registro de usuarios

Basicamente habrá dos formas de crear usuarios en la aplicación: mediante el administrador o mediante el propio usuario. En este caso vamos a implementar la segunda opción. A falta de implementar la autorización, el primer caso ya se ha implementado al crear una RESTFul API de usuarios en el punto anterior.

Para el segundo caso es necesario crear un formulario de registro de usuarios. Para ello, creamos un nuevo fichero `register.html` en la carpeta `auth`. Este fichero será bastante parecido al de creación de usuarios por parte del administrador pero sin incluir algunos campos como el de `role`.

```html
{% extends 'layout.html' %}
{% block content %}
<h2>Registro</h2>
<form action='/auth/register' method="POST">
    <label>Nombre:</label>
    <input type="text" name="nombre" required>
    <label>Email:</label>
    <input type="email" name="email" required>
    <label>Contraseña:</label>
    <input type="password" name="password" required>
    <button type="submit">Registrarse</button>
</form>

<p>¿Ya tienes cuenta? <a href="/auth/login">Inicia sesión</a></p>
{% endblock %}
```

Se ha añadido un enlace para que los usuarios redirigirse a iniciar sesión.

Una vez creado el formulario, debemos crear las rutas necesarias para gestionar el registro de usuarios. Para ello, añadimos las siguientes funciones al blueprint `auth.py`:

```python
@auth_bp.route('/register', methods=['GET'])
def register_view():
    return render_template('register.html')

@auth_bp.route('/register', methods=['POST'])
def register():
    nombre = request.form['nombre']
    email = request.form['email']
    password = request.form['password']
    
    if User.query.filter_by(email=email).first():
        flash('El email ya está registrado.')
        return redirect('/auth/register')
    
    new_user = User(nombre=nombre, email=email)
    new_user.set_password(password)
    db.session.add(new_user)
    db.session.commit()
    flash('Registro exitoso, ahora puedes iniciar sesión.')
    return redirect('/')
```

En la segunda ruta vemos como se crea un usuario a través del modelo y se añade a la base de datos. Se comprueba si el email ya está registrado y si no se añade el nuevo usuario a la base de datos. Para ello, se hace uso de la función `set_password` que hemos definido en el modelo `Usuario` para almacenar la contraseña de forma segura. 

### Login

En este apartado nos vamos a encargar de la autenticación propiamente dicha. **La autenticación no es más que comprobar que un usuario es quien dice ser**. Para comprobar existen varias maneras, pero la más básica es mediante un **par de credenciales (usuario y contraseña) que el usuario introduce en un formulario**.

Para esto se debe crear un formulario de login en un fichero `login.html` en la carpeta `auth` similar al de registro pero con los campos de email y contraseña. 

```html
{% extends 'layout.html' %}
{% block content %}
<h2>Iniciar Sesión</h2>
<form action="/auth/login" method="POST">
    <label>Email:</label>
    <input type="email" name="email" required>
    <label>Contraseña:</label>
    <input type="password" name="password" required>
    <button type="submit">Ingresar</button>
</form>

<p>¿No tienes cuenta? <a href="/auth/register">Regístrate</a></p>
{% endblock %}
```

Una vez creado el formulario, debemos crear las rutas necesarias para gestionar el registro de usuarios. Para ello, añadimos las siguientes funciones al blueprint `auth.py`: 

```python
@auth_bp.route('/login', methods=['GET'])
def login_view():
    return render_template('login.html')

@auth_bp.route('/login', methods=['POST'])
def login():
    email = request.form['email']
    password = request.form['password']
    user = User.query.filter_by(email=email).first()

    if user and user.check_password(password):
        session['user'] = {
            'nombre': user.nombre,
            'email': user.email,
            'role': user.role,
            'id': user.id
        }
        flash('Inicio de sesión exitoso.')
        return redirect('/')
    else:
        flash('Correo o contraseña incorrectos.')
        return redirect('/auth/login')
```

En la segunda se comprueba si el usuario existe y si la contraseña es correcta mediante el método `check_password` que hemos definido en el modelo `Usuario`. En caso afirmativo, se inicia la sesión del usuario y se redirige a la página principal. En caso contrario, se muestra un mensaje de error y se redirige al formulario de login.

Para inciar sesion hemos usado la variable `session` que nos proporciona Flask. Esta variable nos permite almacenar información en la sesión del usuario. La sesión se quedará encriptada gracias a la variable app.secret_key que ya se había definido en el fichero `app.py` cuando usamos flash por primera vez. De hecho, **flash se basa en el mismo concepto de sesiones que el login**, es decir, en el almacenamiento de información en el navegador del usuario.


### Logout

Por último, debemos crear una ruta para cerrar la sesión de un usuario. Para ello, añadimos la siguiente función al blueprint `auth.py`:


```python
@auth_bp.route('/logout')
def logout():
    session.pop('user', None)
    flash('Sesión cerrada correctamente.')
    return redirect('/')
```

Basicamente hemos eliminado la variable `user` de la sesión del usuario.

### Opciones de autenticación en el menú de navegación

Para finalizar, debemos añadir las opciones de autenticación al menú de navegación. Para ello, debemos modificar el fichero `layout.html` en la carpeta `templates` para que muestre las opciones de login, registro y logout según el estado de la sesión del usuario.

```html
<nav>

    ...

    <a href="/auth/login">Login</a>
    <a href="/auth/register">Registro</a>
    {% if 'user' not in session %}
        <a href="/auth/login">Login</a>
        <a href="/auth/register">Registro</a>
    {% endif %}
    {% if session['user'] %}
        <a href="/vias">Vias</a>
        <a href="/users">Users</a>
        <span>Bienvenido, {{session['user']['nombre']}}</span>
        <a href="/auth/logout">Logout</a>
    {% endif %}
</nav>
```

Se usa la variable `session[user]` para acceder al usuario autenticado. Esta variable es accesible en todas las plantillas y contiene la información del usuario que ha iniciado sesión. En este caso, se ha usado el atributo `nombre` de user para mostrar el nombre del usuario en el menú de navegación. Además, se muestran determinados enlances según si el usuario ha iniciado sesión o no.


---

Hasta aqui hemos realizado la parte más básica de gestión de identidades, en la que hemos realizado el login, el registro y el logout de usuarios. Hemos visto que la autenticación es el proceso que nos permite verificar la identidad de un usuario y que este es quien dice ser. No obstante, existen varias formas de gestionar la identidad de los usuarios e incluso librerías que lo gestionan por nosotros como puede ser Flask-login.


<blockquote>
<h4>Inciso: Gestión de identidades</h4>
<p>
    A lo largo de la historia ha ido evolucionando y se puede considerar que ha habido 4 fases:
</p>
<p>

</p>
<h6>Funcionamiento de sesiones:</h6>
<ol>
  <li><strong>Identidad centralizada</strong>: Cada aplicación maneja sus identidades de forma independiente. Cada vez que un usuario se registra en una aplicación, debe crear un nuevo usuario y contraseña. Esto puede llevar a que los usuarios tengan que recordar muchas credenciales diferentes. Este es el que hemos implementado en este apartado.</li>
  <li><strong>Identidad Federada</strong>: Se permite a los usuarios usar las credenciales de una aplicación para acceder a otra. Por ejemplo, la red eduroam es un buen ejemplo, donde los alumnos pueden usar sus credenciales de la universidad para acceder a la red de otras universidades.</li>
  <li><strong>Identidad centrada en el usuario</strong>: En este caso, el usuario es el propietario de su identidad y decide qué aplicaciones pueden acceder a ella. Un ejemplo de esto es OAuth, que permite a los usuarios autorizar a una aplicación a acceder a su información sin compartir su contraseña. Para este caso, python provee librerías como <code>Flask-OAuth</code> que permiten implementar este tipo de autenticación. <strong>Este es el caso más común actualmente</strong> y es el que se usa en aplicaciones como Google o Facebook.</li>
  <li><strong>Identidad autosoberana</strong>: En este caso, el usuario es el propietario de su identidad y la almacena en un lugar seguro. Un ejemplo de esto es la tecnología blockchain, donde los usuarios pueden almacenar en wallets propias sus credenciales y a través de la blockchain los servicios las pueden verificar sin necesidad de un tercero.</li>
</ol>

</blockquote>
<br>

En la siguiente sección veremos más en detalle como implementar la autorización y como podemos usar la información de la sesion del usuario para autorizar o no ciertas acciones.
