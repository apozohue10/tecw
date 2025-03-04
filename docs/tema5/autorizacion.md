# Autorizacion

Una vez que un usuario se ha autenticado podemos controlar que puede hacer y que no puede hacer. Esto es lo que se conoce como autorización. Para implementar la autorización en una aplicación web se pueden utilizar diferentes técnicas como por ejemplo Control de acceso basado en roles (RBAC) o Control de acceso basado en atributos (ABAC). No obstante, el primer control de acceso que podemos implementar es el basado en la sesión del usuario.

## Control de sesión

Hasta ahora se han definido todas las funciones que permiten manejar el ciclo de la sesión de los usuarios. El siguiente paso es adaptar todas nuestras interfaces para que se puedan acceder a determinadas rutas solo si el usuario ha iniciado sesión.  Por ejemplo, para acceder a la ruta raiz y a la ruta '/' o '/about' no sería necesario que el usuario este autenticado. 

Pero para acceder a los recursos de vías si lo podemos necesitar. Para ello vamos a utilizar un middleware que se encargará de verificar si el usuario tiene permisos para acceder a una determinada ruta. En este caso, vamos a generar un fichero `access_control.py` en el que definiremos el middleware correspondiente.


```python
from flask import Flask, request, session, redirect
from functools import wraps

def check_session(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if session.get('user') is None:
            return redirect('/')
        return f(*args, **kwargs)
    return decorated_function
```

Este middleware se encargará de verificar si el usuario ha iniciado sesión antes de acceder a una determinada ruta. Para ello, se comprueba si la clave `user` está presente en la sesión del usuario. Si no está presente, se redirige al usuario a la página de inicio de sesión. Esta clave se incluyo en la sesion cuando el usuario realizo el login.

Una vez hecho esto, podemos ahora proteger nuestras rutas incluyendo el siguiente código en `via_bp` y `user_bp`:

```python
from access_control import check_session

...

@via_bp.before_request
@check_session
def before_request_via_bp():
    pass
```

Este código se ejecutará para todas y cada una de las rutas definidas dentro de los blueprints `via_bp` y `user_bp`.

## Control de acceso

### RBAC: Control de Acceso Basado en Roles

RBAC es un modelo de autorización en el que los permisos se asignan a roles, y los usuarios heredan los permisos de los roles a los que pertenecen. Es un método ampliamente utilizado debido a su simplicidad y facilidad de gestión.

- Los permisos no se asignan directamente a los usuarios, sino a roles.
- Los usuarios pueden tener uno o más roles.
- Se define una jerarquía de roles según las necesidades del sistema.

### ABAC: Control de Acceso Basado en Atributos

ABAC es un modelo más flexible que permite definir reglas de autorización basadas en atributos de los usuarios, recursos, acciones y contexto.

- Se toman en cuenta múltiples atributos para la autorización (edad, grupo, ubicación, hora del día, etc.).
- Permite definir políticas más granulares.
- Mayor complejidad en la implementación respecto a RBAC.

---

### Implementación de RBAC

En nuestro caso, vamos a implementar un sistema RBAC sencillo en el que los usuarios pueden ser administradores o usuarios normales. Los administradores tendrán permisos para realizar cualquier acción en la aplicación, mientras que los usuarios normales tendrán permisos limitados. En nuestro caso, los administradores podrán acceder a las rutas de usuarios y vías, mientras que los usuarios normales solo podrán acceder a las rutas de vías.

Para implementar el control de acceso RBAC, en el fichero `access_control.py` podemos implementar el siguiente código:


```python
from flask import Flask, request, session, abort
from functools import wraps

# Definir permisos por rol
permissions = {
    "admin": [
        "via.list",
        "via.new",
        "via.create",
        "via.show",
        "via.edit",
        "via.update",
        "via.delete",
        "user.list",
        "user.new",
        "user.create",
        "user.show",
        "user.edit",
        "user.update",
        "user.delete"
    ],
    "user": [
        "via.list",
        "via.new",
        "via.create",
        "via.show",
        "via.edit",
        "via.update",
        "via.delete",
    ]
}

def check_role():
    def decorator(f):
        @wraps(f)
        def wrapper(*args, **kwargs):
            endpoint = request.endpoint
            role = session["user"]["role"]
            if endpoint not in permissions[role]:
                abort(403)
            return f(*args, **kwargs)
        return wrapper
    return decorator
```

A través de este código, definimos un diccionario `permissions` en el que se definen los permisos asociados a cada rol, que basicamente son las rutas que se pueden acceder. A continuación, definimos un decorador `check_role` que se encargará de verificar si el usuario tiene permisos para acceder a una determinada ruta. Para ello, se comprueba si la ruta a la que se quiere acceder está presente en la lista de permisos asociada al rol del usuario. Si no está presente, se devuelve un error 403 (Unauthorized).

Y nuevamente, podemos ejecutarlo en nuestras rutas de la siguiente manera:

```python
from access_control.py import check_session, check_role

...

@via_bp.before_request
@check_session
@check_role()
def before_request_via_bp():
    pass
```

Y además debemos modifiar el código del navigation para que la lista de usuarios solo sea visible para los administradores:

```html
...

{% if session['user'] %}
    <a href="/vias">Vias</a>
    {% if session['user']['role'] == 'admin' %}
        <a href="/users">Users</a>
    {% endif %}
    <span>Bienvenido, {{session['user']['nombre']}}</span>
    <a href="/auth/logout">Logout</a>
{% endif %}
```

### Gestión de recursos

Actualmente, los usuarios pueden ver y editar cualquier recurso de la aplicación. Sin embargo, en muchos casos, es necesario restringir el acceso a los recursos de la aplicación para que los usuarios solo puedan ver y editar sus propios recursos. En este caso, se trata de un desarrollo a nivel de base de datos, ya que lo primero es establecer algún tipo de relación entre los recursos y los usuarios. Por ejemplo, en el nuestro caso del rocodromo, podemos añadir un campo `user_id` que haga referencia al usuario que ha creado la vía lo que **permite establecer una relación 1:N entre usuarios y vías**.

```python
class Via(db.Model):
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    nombre = db.Column(db.String(255), nullable=False)
    grado = db.Column(db.String(255), nullable=False)
    altura = db.Column(db.Integer, nullable=False)
    desplome = db.Column(db.Boolean, default=False)
    imagen = db.Column(db.String(120), nullable=True, unique=True)
    numero_chapas = db.Column(db.Integer, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False) # Nuevo atributo
```

Una vez añadido, creamos y ejecutamos la migración como hicimos en otras secciones. Una vez realizado esto, debemos adaptar todo el código del blueprint de vías para que los usuarios creen, editen, listen y eliminen solo sus propias vías.

#### 2. load_via

En lugar de buscar una vía por su id, ahora se busca por su id y el id del usuario que la creó. Esto se hace con una consulta a la base de datos con `filter()` en lugar de `get()`. Esto permite, que si un usuario intenta acceder a una vía que no le pertenece, se devuelva un error 404. Además, se ha añadido que solo se aplique este filtro si el usuario no es un administrador.

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
def load_via(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        viaId = request.view_args.get('viaId')
        user_id = session['user']['id']
        role = session['user']['role']
        if viaId is not None:
            query = Via.query.filter_by(id=viaId)    
            if role != 'admin':
                query = query.filter_by(user_id=user_id)
            via_requested = query.first()
            if via_requested is None:
                abort(404)
            kwargs['via'] = via_requested
        return f(*args, **kwargs)
    return decorated_function
```
</div>
<div class="original" hidden>
```python
def load_via(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        viaId = request.view_args.get('viaId')
        if viaId is not None:
            via_requested = Via.query.get(viaId)
            if via_requested is None:
                abort(404)
            kwargs['via'] = via_requested
        return f(*args, **kwargs)
    return decorated_function
```
</div>
</div>


---

#### 3. list

En este caso hemos añadido un filtro para que solo se muestren las vías que pertenecen al usuario que ha iniciado sesión. Para ello, se obtiene el `user_id` de la sesión y se filtra la consulta por ese `user_id`. En el caso de que el usuario sea un admin, este filtro no se aplica y así el administrador porá ver todas las vías.

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
def list():
    min_altura = request.args.get('min_altura', type=int)
    max_altura = request.args.get('max_altura', type=int)
    grado = request.args.get('grado')

    query = Via.query

    # Obtener el user_id de la sesión
    user_id = session['user']['id']
    role = session['user']['role']

    # Filtrar por user_id
    if role != 'admin':
        query = query.filter(Via.user_id == user_id)

    if min_altura is not None:
        query = query.filter(Via.altura >= min_altura)

    if max_altura is not None:
        query = query.filter(Via.altura <= max_altura)

    if grado and grado != 'all':
        query = query.filter(Via.grado == grado)

    vias = query.all()
    
    return render_template('list.html', vias=vias, grades=grades)
```
</div>
<div class="original" hidden>
```python
def list():
    min_altura = request.args.get('min_altura', type=int)
    max_altura = request.args.get('max_altura', type=int)
    grado = request.args.get('grado')

    query = Via.query

    if min_altura is not None:
        query = query.filter(Via.altura >= min_altura)

    if max_altura is not None:
        query = query.filter(Via.altura <= max_altura)

    if grado and grado != 'all':
        query = query.filter(Via.grado == grado)

    vias = query.all()
    
    return render_template('list.html', vias=vias, grades=grades)
```
</div>
</div>


---

#### 4. Create

En este caso se ha añadido el `user_id` a la vía antes de persistirla en la base de datos. De esta forma, se asocia la vía al usuario que la ha creado. El dato te `user_id` se obtiene de la sesión del usuario y **asi garantizamos que el usuario que ha hecho login es el que ha creado la vía**.

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
def create(filename):
    via = Via(
        nombre=request.form.get('nombre'),
        grado=request.form.get('grado'),
        altura=float(request.form.get('altura')),
        numero_chapas=float(request.form.get('numero_chapas')),
        desplome=request.form.get('desplome') == 'true',
        filename=filename,
        user_id=session['user']['id']
    )
    db.session.add(via)
    db.session.commit()
    
    return redirect('/vias')
```
</div>
<div class="original" hidden>
```python
def create(filename):
    via = Via(
        nombre=request.form.get('nombre'),
        grado=request.form.get('grado'),
        altura=float(request.form.get('altura')),
        numero_chapas=float(request.form.get('numero_chapas')),
        desplome=request.form.get('desplome') == 'true',
        filename=filename
    )
    db.session.add(via)
    db.session.commit()
    
    return redirect('/vias')
```
</div>
</div>

---

#### 5. Update

En este caso no hay que editar nada, ya que el usuario solo puede editar sus propias vías. Por lo tanto, no es necesario añadir ningún filtro adicional.

---

#### 6. Delete

En este caso no hay que editar nada, ya que el usuario solo puede eliminar sus propias vías. Por lo tanto, no es necesario añadir ningún filtro adicional.


---

Con todo esto, la estructura de ficheros nos queda de la siguiente manera:

```plaintext
rocodromo/
│
├── app/                        # Carpeta principal de la aplicación
│   ├── assets/                 # Contiene ficheros estáticos que no pueden accederse directamente a través de una URL
│       └── images              # Contiene imágenes de carácter privado
│   ├── certs/                  # Contiene los certificados SSL/TLS
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
│   ├── access_control.py       # Middlewares para controlar el acceso a las rutas
│   ├── database.py             # Contiene el objeto de conexión a la base de datos
│   └── handle_files.py         # Middlewares para gestionar ficheros
├── requirements.txt            # Lista de dependencias del proyecto
└── env/                        # Entorno virtual
```

---

En nuestro caso, hemos implementado un RBAC que ha separado RESTFul APIs enteras. Por ejemplo el admin podía acceder a todas las rutas de vías y usuarios, mientras que el usuario normal solo podía acceder a las rutas de vías. Pero también se puede implementar un RBAC más granular en el que se restrinja el acceso a determinadas rutas o recursos. Los usuarios, por ejemplo, suelen necesitar ver, editar e incluso eliminar su propio perfil pero no el de otros usuarios. Esto implica desarrollar un sistema RBAC más complejo en el que se definen políticas de autorización más granulares.

Como comentario final cuando **una aplicación escala y se vuelve más compleja es necesario implementar un sistema de autorización más avanzado**. En estos casos, es recomendable utilizar un sistema de control de acceso basado en roles (RBAC) más avanzado o control de acceso basado en atributos (ABAC). Para ello, existen ya herramientas y librerías que facilitan la implementación de estos sistemas. 

Estas librería suelen implementar estándares ya existentes como XACML que define tanto la arquitectura software a desplegar como el lenguaje de definición de politicas. Por ejemplo, existe la herramienta open-source llamada [Authzforce](https://authzforce-ce-fiware.readthedocs.io/en/latest/) que implementa dicho estándar y proporciona una API (basado en HTTP) a través de la cual definir las políticas de nuestra aplicación en formato XML. Inluso existen algunos modelos que permiten no solo controlar el acceso a los datos si no también el uso de los mismos. Pero esto, esta fuera del alcance de este curso.


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
