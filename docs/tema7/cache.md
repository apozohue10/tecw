# Caché

Imagina que **en nuestro servidor web de rocódromos** la gente realiza consultas frecuentes para ver la lista de vías disponibles. Cada una de esas consultas implica una consulta a la base de datos y además puede implicar descargarse imágenes también. Si el servidor está muy cargado, esto puede ralentizar la respuesta del servidor y por tanto la experiencia de usuario se verá afectada. Para evitar esto, se puede implementar un sistema de caché.

Un **sistema de caché** es un mecanismo de almacenamiento temporal que guarda copias de datos previamente calculados o recuperados, con el objetivo de reducir el tiempo de acceso a esos datos en futuras solicitudes. En lugar de calcular o recuperar los datos repetidamente desde una fuente lenta, como una base de datos o una API, el sistema de caché permite acceder a los datos desde un almacenamiento más rápido (generalmente en memoria).

Los sistemas de caché son comunes en aplicaciones web, sistemas operativos, bases de datos y servidores para mejorar la eficiencia y el rendimiento. Son cruciales por varias razones:

1. **Mejorar el rendimiento**: Acceder a la memoria es mucho más rápido que hacer consultas a bases de datos o llamadas a servicios externos. Al almacenar datos en caché, se mejora la velocidad de respuesta y la eficiencia general de la aplicación.
2. **Reducir la carga en recursos**: El uso de caché disminuye la necesidad de realizar operaciones repetitivas, lo que alivia la carga en servidores de bases de datos y otros sistemas de backend.
3. **Escalabilidad**: Cuando se usan cachés adecuados, una aplicación puede manejar un mayor número de solicitudes sin un aumento proporcional en los tiempos de respuesta o en el uso de recursos.


Se pueden implementar cachés a varios niveles en una aplicación web, como:
- **Caché en servidor**: Almacenar datos en la memoria del servidor (ej. caché en memoria, caché en disco).
- **Caché en red**: Almacenar datos en una capa de red (ej. caché de proxy, caché de CDN).
- **Caché en cliente**: Almacenar datos en el navegador del usuario (ej. cookies, localStorage, sessionStorage).

En esta sección veremos como implementar un sistema de caché en el servidor y en el cliente.

## Caché en servidor

En un servidor basado en Flask podemos implementar un sistema de caché utilizando la extensión `Flask-Caching`. Para instalarla, ejecutamos el siguiente comando:

```bash
pip install Flask-Caching
```

Una vez instalada, debemos actualizar el fichero `requirements.txt` con la nueva dependencia instalada:

```bash
pip freeze > requirements.txt
```

A continuación, debemos configurar la caché en la aplicación Flask. Por ejemplo, para configurar una caché en memoria, debemos poner en el archivo `app.py`:

```python
from flask_caching import Cache

...

##### Cache
app.config['CACHE_TYPE'] = 'simple' 
app.config['CACHE_DEFAULT_TIMEOUT'] = 300  # Tiempo en segundos (5 minutos)
cache = Cache(app)
```

Donde cada uno de los parámetros de configuración se refiere a:
- `CACHE_TYPE`: Tipo de caché a utilizar. En este caso, `simple` es una caché en memoria. Pero también se puede usar fichero (`filesystem`) o incluso otras bases de datos (`redis`, `sqlite`, `mongodb`, `couchdb`, entre otras)  e incluso orms (`sqlalchemy`). Merece mención especial `redis` que es **una base de datos en memoria que se utiliza mucho para caché**.
- `CACHE_DEFAULT_TIMEOUT`: Tiempo en segundos que se almacenará un objeto en caché si no se especifica. En este caso, 300 segundos (5 minutos).


Una vez configurada la caché, podemos utilizarla en nuestra aplicación. Principalmente podemos cachear 2 cosas: **rutas** y **recursos**.

### Cacheo de rutas

El cacheo de rutas consiste en almacenar en caché el resultado de una función que se ejecuta al acceder a una ruta. Si queremos cachear por ejemplo, la página principal de nuestro servidor web (dado que es una página muy frecuentada), podemos hacerlo de la siguiente forma en el archivo `app.py`:

```python
@app.route('/')
@cache.cached(timeout=10)  # Caché durante 10 segundos
def index():
    # Simulamos un retraso de procesamiento (por ejemplo, llamada a una API lenta)
    # time.sleep(5)
    return render_template('home.html')
```

En este caso, la función `index` se cacheará durante 10 segundos. Si se vuelve a acceder a la ruta `/` en menos de 10 segundos, se devolverá el resultado de la caché en lugar de ejecutar la función. Para ello hemos añadido el decorador `@cache.cached(timeout=10)` a la función `index`.

Para probar esto puede descomentar la línea `time.sleep(5)` para simular un retraso de procesamiento y ver cómo la caché se activa probando a acceder varias veces.

### Cacheo de recursos

El cacheo de recursos consiste en almacenar en caché datos que se obtienen de una base de datos o de una API. Para ello, en `Flask-Caching` disponemos de dos funciones que nos permiten guardar y obtener datos de la caché: `cache.set` y `cache.get`. Por ejemplo, podemos editar la función que metía en una cabecera de HTTP el número total de vías en la base de datos, ya que es algo que no cambia con frecuencia y por tanto podemos cachearlo.

```python
@via_bp.after_request
def agregar_total_vias(response):
    total_vias = cache.get('total_vias')
    if total_vias is None:
        total_vias = Via.query.count()
        cache.set('total_vias', total_vias, timeout=60*60)  # Cachear por 1 hora
    response.headers["X-Total-Vias"] = total_vias
    return response
```

De esta forma, lo primero que hacemos es intentar obtener el número total de vías de la caché. Si no está en caché, lo obtenemos de la base de datos y lo guardamos en caché durante 1 hora. De esta forma, si se vuelve a acceder a la ruta que llama a esta función en menos de 1 hora, se devolverá el número total de vías de la caché en lugar de hacer una consulta a la base de datos.


---

Esto son algunas formas simples de implementar caché en Flask. Existen muchas más opciones y configuraciones que se pueden hacer con `Flask-Caching`. Para más información, puedes consultar la [documentación oficial de Flask-Caching](https://flask-caching.readthedocs.io/en/latest/). Como se ha comentado antes, `Flask-Caching` permite almacenar no solo en memoria, sino también en bases de datos como Redis (o incluso ficheros pero suele ser menos eficiente). Y aquí surge la siguiente cuestión:


### ¿Cuándo usar una Caché en Memoria y cuándo una Caché con Redis en Flask?

El uso de una caché en memoria o una caché con Redis depende de las necesidades y la escala de tu aplicación Flask.

#### Usar una Caché en Memoria 

Podemos usarla cuando:

- **Aplicaciones pequeñas o de bajo tráfico**: Si tu servidor corre en una sola instancia y el tráfico no es muy alto, la caché en memoria puede ser suficiente.
- **Evitar sobrecarga en consultas repetitivas**: Ideal para almacenar resultados de funciones que se llaman frecuentemente y cuyos datos cambian poco (ej. cálculos costosos, consultas a la base de datos con poca variabilidad).
- **Necesitas baja latencia**: Al estar en la misma memoria del proceso, el acceso a los datos es extremadamente rápido.
- **No necesitas compartir la caché entre múltiples instancias**: Si el servidor esta arrancado en un solo proceso o dentro de un único contenedor, la caché en memoria es válida.

Pero tiene una serie de limitaciones:

- **Se pierde** cuando el servidor se **reinicia**.
- **No es escalable para múltiples servidores**, ya que cada una tendría su propia caché separada.
- Puede **aumentar el consumo de RAM** si almacenas muchos datos en memoria.

#### Usar una Caché con Redis

Redis es un tipo de base de datos en memoria que se utiliza mucho para caché. Algunas de las ventajas de usar Redis son: 

- **Tienes múltiples instancias del servidor web**: Redis permite compartir la caché entre varias instancias, evitando inconsistencias y duplicación de datos en memoria.
- **Necesitas persistencia parcial**: Redis puede configurarse para mantener los datos aunque la aplicación se reinicie.
- **Tienes alto tráfico y necesitas escalar horizontalmente**: Redis es eficiente para manejar muchas peticiones concurrentes.
- **Almacenas datos temporales y sesiones de usuarios**: Redis es excelente para manejar autenticación, sesiones y datos que expiran automáticamente.
- **Quieres más flexibilidad**: Redis permite estructuras de datos avanzadas (listas, conjuntos, hashes), TTLs configurables y políticas de expiración más avanzadas.

Pero tiene una serie de limitaciones:

- **Introduce latencia** en comparación con una caché en memoria pura, aunque sigue siendo rápida.
- Necesitas configurar y **mantener Redis** como un servicio externo.
- Aumenta la **complejidad del despliegue y la infraestructura**.

#### Resumen: ¿Cuál elegir?

| Caso de Uso | Caché en Memoria | Caché con Redis |
|------------|-----------------|----------------|
| Pocas consultas repetitivas | ✅ | ❌ |
| Aplicación monolítica y pequeña | ✅ | ❌ |
| Múltiples instancias de Flask | ❌ | ✅ |
| Escalabilidad horizontal | ❌ | ✅ |
| Persistencia parcial tras reinicio | ❌ | ✅ |
| Manejo de sesiones | ❌ | ✅ |
| Expiración avanzada de caché | ❌ | ✅ |

---

Basicamente depende de la apliacación que estés desarrollando. En general, puedes seguir estas reglas generales:

- Si es una **aplicación pequeña o monolítica** → **Caché en memoria**.
- Si usas **múltiples instancias de Flask o quieres escalabilidad** → **Redis**.

Si tu aplicación empieza pequeña pero podría crecer, podrías comenzar con una caché en memoria y migrar a Redis cuando sea necesario.


## Caché en cliente

En el lado del cliente se puede implementar un sistema de caché utilizando el **almacenamiento local del navegador** basado en Javascript de cliente. Por ejemplo, podemos almacenar en caché los datos de una API en el navegador para evitar hacer consultas repetitivas al servidor. Generalmente este tipo de caché se basa en peticiones AJAX (fetch) y en el uso de `localStorage` o `sessionStorage` de Javascript.

De forma similar a `Flask-Caching` existen dos funciones principales para almacenar y obtener datos de la caché en el navegador: `localStorage.setItem` y `localStorage.getItem`. 

Por ejemplo, podriamos almacenar el nombre del usuario en localStorage para que no se incluya en las cookies que se envían en cada petición y así reducir el tamaño de las peticiones. Para ello debemos modificar el código de la función `login` en el archivo `auth.js`:


<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
@auth_bp.route('/login', methods=['POST'])
def login():
    email = request.form['email']
    password = request.form['password']
    user = User.query.filter_by(email=email).first()

    if user and user.check_password(password):
        session['user'] = {
            'email': user.email,
            'role': user.role,
            'id': user.id
        }
        flash('Inicio de sesión exitoso.')
        return jsonify({'nombre': user.nombre}), 200
    else:
        flash('Correo o contraseña incorrectos.')
        return redirect('/auth/login')
```
</div>
<div class="original" hidden>
```python
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
</div>
</div>

En este código hemos quitado el nombre de `session['user']` y en vez de redirigir cuando la autenticación es correcta, devolvemos un JSON con el nombre del usuario. En el lado del cliente, se implementará una función fetch que llamará a la ruta `/login`, procesará la respuesta json y almacenará el nombre del usuario en localStorage. 

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```html
{% extends 'layout.html' %}
{% block content %}
<script src="/javascripts/login.js"></script> 
<h1>Iniciar Sesión</h1>
<form id="loginForm" method="POST" action="/auth/login">
    <label for="email">Email:</label>
    <input type="email" id="email" name="email" required>
    <br>
    <label for="password">Contraseña:</label>
    <input type="password" id="password" name="password" required>
    <br>
    <button type="submit">Iniciar Sesión</button>
</form>
{% endblock %}
```
</div>
<div class="original" hidden>
```html
{% extends 'layout.html' %}
{% block content %}
<h1>Iniciar Sesión</h1>
<form method="POST" action="/auth/login">
    <label for="email">Email:</label>
    <input type="email" id="email" name="email" required>
    <br>
    <label for="password">Contraseña:</label>
    <input type="password" id="password" name="password" required>
    <br>
    <button type="submit">Iniciar Sesión</button>
</form>
{% endblock %}
```
</div>
</div>



En este caso solo hemos añadido un id al formulario para poder añadir un evento en Javascript y hemos incluido el script `login.js` que contendrá la lógica necesaria.

El archivo `login.js` lo creamos dentro de la carpeta `public/javascripts` con el siguiente contenido:

```javascript
document.addEventListener('DOMContentLoaded', function() {
    document.getElementById('loginForm').addEventListener('submit', function(event) {
        event.preventDefault();
        const form = event.target;
        const formData = new FormData(form);
        fetch(form.action, {
            method: form.method,
            body: formData
        })
        .then(response => response.json())
        .then(data => {
            if (data.nombre) {
                localStorage.setItem('nombre', data.nombre);
                window.location.href = '/';
            } else {
                alert('Correo o contraseña incorrectos.');
            }
        })
        .catch(error => console.error('Error:', error));
    });
});
```

Esta función espera a que este cargada la página y una vez que esta cargada:

1. Añade un evento al formulario de login que se ejecuta cuando se envía el formulario.
2. Evita que se envíe el formulario con `event.preventDefault()`.
3. Obtiene los datos del formulario con `new FormData(form)`.
4. Realiza una petición fetch al servidor con los datos del formulario.
5. Procesa la respuesta en formato JSON.
6. Si la respuesta contiene un nombre, lo almacena en localStorage y redirige a la página principal.
7. Si la respuesta no contiene un nombre, muestra un mensaje de error.

En este momento, hemos almacenado el nombre del usuario en localStorage. Para recuperarlo y mostrarlo en la página principal, podemos modificar el archivo `layout.html` para que muestre el nombre del usuario en el menú de navegación si está almacenado en localStorage:

```html
<script src="/javascripts/obtain_username.js"></script> 

<nav>
...

<span>Bienvenido, <span id="user-name"></span></span>

...
</nav>
```

Y creamos el archivo `obtain_username.js` en la carpeta `public/javascripts` con el siguiente contenido:

```javascript
document.addEventListener('DOMContentLoaded', (event) => {
    const userName = localStorage.getItem('nombre');
    if (userName) {
        document.getElementById('user-name').textContent = userName;
    }
});
```

Donde obtenemos el nombre del usuario de localStorage y lo mostramos en el elemento con id `user-name` si existe.

---

Con esto hemos visto distintas formas de cachear tanto en el servidor como en el cliente. En el servidor, hemos visto cómo cachear rutas y recursos en Flask utilizando `Flask-Caching`. En el cliente, hemos visto cómo almacenar datos en caché en el navegador utilizando `localStorage` y `sessionStorage` de Javascript.








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