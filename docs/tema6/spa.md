# SPA

Las Aplicaciones de Una Sola Página (SPA, por sus siglas en inglés) son aplicaciones web diseñadas para funcionar como una única página dinámica. A diferencia de las aplicaciones tradicionales de múltiples páginas, donde cada interacción resulta en una nueva página cargada desde el servidor, las SPAs cargan una sola página HTML y actualizan dinámicamente el contenido a medida que el usuario interactúa con la aplicación. Las SPAs pueden:

- **Mejora de la experiencia del usuario**: Las SPAs ofrecen una experiencia de usuario más fluida y rápida. Dado que solo se actualiza el contenido necesario, las interacciones se sienten más continuas y menos interrumpidas por recargas de página.  
- **Reducción de tiempos de carga**: Después de la carga inicial de la página, las SPAs solo solicitan los datos necesarios para actualizar partes específicas de la página, lo que hace que las interacciones posteriores sean más rápidas y eficientes.
- **Mejor rendimiento**: Al minimizar las recargas completas de la página, las SPAs reducen la carga en el servidor y mejoran el rendimiento de la aplicación, especialmente en aplicaciones complejas que requieren actualizaciones en tiempo real.
- **Compatibilidad con dispositivos móviles**: Las SPAs son especialmente adecuadas para dispositivos móviles, ya que ofrecen una interfaz más rápida y sensible en comparación con las aplicaciones tradicionales de múltiples páginas.

Esencialmente, una SPA es una aplicación web que se carga una sola vez y luego se actualiza dinámicamente a medida que el usuario interactúa con ella. Esto se logra mediante el uso de tecnologías como AJAX (Asynchronous JavaScript and XML), que permite cargar y enviar datos al servidor sin tener que recargar la página completa. Es decir, **mediante peticiones fetch desde el cliente obtenemos la información que necesitamos y creamos el contenido HTML necesario usando javascript**.

En nuestro caso, tenemos un sistema de rutas, que, por lo general, por cada ruta se renderiza un fichero HTML. Tranformar toda la aplicación web para que sea SPA es un proceso complejo y que requiere de un rediseño completo de la aplicación. Sin embargo, podemos hacer que ciertas partes de la aplicación sean SPA para demostrar como se puede realizar.

Por ejemplo, podemos realizar toda la Restful API de vías para que se comporte como una SPA. Para ello tendremos que hacer una serie de modificaciones en el servidor y en el cliente. 

## Servidor

Una de las claves del lado del servidor es que las rutas que devuelven HTML, devuelvan JSON. En nuestro caso, todas devolveran un JSON. Crearemos una nueva ruta que servirá un fichero `index.html` que ser el punto de entrada de la SPA de vías. Como se ha comentado antes una **SPA pura serviría todo el contenido HTML en una sola carga**, pero en nuestro caso, serviremos un fichero HTML que cargará el contenido de las vías mediante peticiones fetch por simplicidad.

Del lado del servidor tendremos que generar una nueva ruta que sirva el fichero `index.html` de la SPA de vías. Esto lo haremos desde el fichero `app.py` de la siguiente manera:

```python
@app.route('/vias-index')
def viaIndex():
    return render_template('via/index.html')
```

De esta manera, dejaremos en el blueprint de `vias` todas las rutas que componen la RESTful API de vías. En este caso, hay que editar también el menu de navegación para que apunte a la nueva ruta de `viaIndex`.

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
<a href="/vias-index">Vias</a>
```
</div>
<div class="original" hidden>
```python
<a href="/vias">Vias</a>
```
</div>
</div>

<br>

Como se puede ver, se devuelve el fichero `index.html` sin pasarle ningún parámetro. Este fichero como se verá después, se compone de todos los ficheros HTML que se servían antes en las rutas de las vías.

Por otro lado, tenemos que modificar ahora todas las rutas para que devuelvan un JSON en vez de un HTML. Para ello usaremos la librería `jsonify` de Flask, que convierte un diccionario en un JSON y devuelve con una respuesta HTTP con la cabecera `Content-Type: application/json` y además permite añadir un código de estado HTTP como 200, 404, 405, etc.

El problema es que la función jsonify no entiende de objetos generados a través del ORM de SQLAlchemy, por lo que tendremos que convertir los objetos a diccionarios previamente. Para ello, podemos incluir el siguiente método en el modelo de la vía.

```python
def to_dict(self):
    return {
        'id': self.id,
        'nombre': self.nombre,
        'grado': self.grado,
        'altura': self.altura,
        'desplome': self.desplome,
        'numero_chapas': self.numero_chapas,
        'filename': self.filename,
        'user_id': self.user_id
    }
```
<blockquote>
<h4>Inciso: Marshmallow</h4>
<p>
    Si solo se necesita una conversión simple de tus modelos SQLAlchemy a diccionarios (por ejemplo, para devolverlos en JSON), se puede hacer con el método <code>to_dict</code> como en este caso. Lo cual tiene la ventaja de que es muy sencillo de implementar y no se necesitan librería extra. Sin embargo, si se necesita una conversión más compleja, se puede usar la librería <a href="https://marshmallow.readthedocs.io/en/stable/">Marshmallow</a> que permite una serialización y deserialización de objetos más compleja. Los Marshmallow:
</p>
<ul>
    <li>Permiten excluir o modificar campos fácilmente.</li>
    <li>Soportan validaciones y transformación de datos.</li>
    <li>Funcionan bien con relaciones (<code>Nested()</code>).</li>
    <li>Pueden deserializar datos y convertirlos a objetos SQLAlchemy.</li>
</ul>
<p>    
    Se basan en la definición de esquemas que permiten definir cómo se serializan y deserializan los objetos. Ejemplo en vía:
```python
from marshmallow import Schema, fields

class ViaSchema(Schema):
    id = fields.Int()
    nombre = fields.Str()
    grado = fields.Str()
    altura = fields.Float()
    desplome = fields.Bool()
    numero_chapas = fields.Int()
    imagen = fields.Str()
    user_id = fields.Int()
```
</p>    
Estos se pueden usar luego para serializar y deserializar objetos de la siguiente manera:
```python
from .schemas import ViaSchema

via_schema = ViaSchema()
vias_schema = ViaSchema(many=True)

...

viaSerialized = vias_schema.dump(via)
```

</p>

</blockquote>
<br>

### load

En este caso, solo tenemos que modificar la línea de `abort(404)` por `jsonify({'error': 'Not found'}), 404`. Ya que al ejecutar `abort(404)` se lanza una excepción que se captura y se devuelve un HTML con el error 404. Y en nuestro caso, en vez de devolver un HTML, devolveremos un JSON con el error, que se gestionará desde el cliente.

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
                jsonify({'error': 'Not found'}), 404
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
</div>

<br>


### list

Devolveremos un JSON con todas las vías en vez de un HTML con todas las vías. Se usa el método `to_dict` que hemos definido anteriormente para convertir los objetos de SQLAlchemy a diccionarios.

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

    vias = query.all()
    vias_list = [via.to_dict() for via in vias]

    return jsonify(vias=vias_list), 200
```
</div>
<div class="original" hidden>
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
    vias_list = [via.to_dict() for via in vias]  # Assuming Via model has a to_dict method

    return jsonify(vias=vias_list), 200
```
</div>
</div>

<br>

### new y edit

Estas rutas las tendremos que borrar ya que en nuestro caso crearemos y editaremos vías desde la SPA.

### create

En este caso, devolvemos el objeto creado en vez de redirigir a la lista de vías.

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
        desplome=request.form.get('desplome') == 'true',
        numero_chapas=float(request.form.get('numero_chapas')),
        filename=filename,
        user_id=session['user']['id']
    )
    db.session.add(via)
    db.session.commit()
    via = via.to_dict()
    return jsonify(via=via), 200
```
</div>
<div class="original" hidden>
```python
def create(filename):
    via = Via(
        nombre=request.form.get('nombre'),
        grado=request.form.get('grado'),
        altura=float(request.form.get('altura')),
        desplome=request.form.get('desplome') == 'true',
        numero_chapas=float(request.form.get('numero_chapas')),
        filename=filename,
        user_id=session['user']['id']
    )
    db.session.add(via)
    db.session.commit()

    return redirect('/vias')
```
</div>
</div>

<br>

### show

Para el caso de show, nos limitaremos a devolver un JSON con la vía solicitada que ya esta precargada por el decorador `load_via`.

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
def show(viaId, via):
    via = via.to_dict()
    return jsonify(via=via), 200
```
</div>
<div class="original" hidden>
```python
def show(viaId, via):    
    return render_template('via/show.html', via=via)
```
</div>
</div>

<br>

### update

Para el caso de update, devolveremos el objeto actualizado en vez de redirigir a la lista de vías. En este caso, tenemos una ventaja y **es que podemos ya especificar que en la ruta se use el método PUT**, ya que en las SPAs se usan los métodos HTTP para especificar la acción que se quiere realizar. Es decir, haciendo un fetch desde JS de cliente tenemos la posible de especificar el método PUT. Cuando en los formularios de HTML solo se puede especificar GET o POST.

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
@via_bp.route('/<viaId>', methods=['PUT'])
@check_via
@load_via
@delete_file
@save_file
def update(viaId, via, filename):
    via.nombre = request.form.get('nombre')
    via.grado = request.form.get('grado')
    via.altura = float(request.form.get('altura'))
    via.numero_chapas = float(request.form.get('numero_chapas'))
    via.desplome = request.form.get('desplome') == 'true'
    via.filename = filename
    db.session.commit()
    via = via.to_dict()
    return jsonify(via=via), 200
```
</div>
<div class="original" hidden>
```python
@via_bp.route('/<viaId>/update', methods=['POST'])
@check_via
@load_via
@delete_file
@save_file
def update(viaId, via, filename):
    if request.form.get('_method') == 'PUT':
        via.nombre = request.form.get('nombre')
        via.grado = request.form.get('grado')
        via.altura = float(request.form.get('altura'))
        via.numero_chapas = float(request.form.get('numero_chapas'))
        via.desplome = request.form.get('desplome') == 'true'
        via.filename = filename
        db.session.commit()
        return redirect('/vias')
    return "Method Not Allowed", 405
```
</div>
</div>

<br>

Otro cambio relevante ha sido suprimir `/update` del path de la ruta, ya que en las SPAs no se usan rutas para acciones específicas, si no que se usan los métodos HTTP para especificar la acción que se quiere realizar. En este caso, el método PUT.

### delete

Para el caso de delete, nos limitaremos a enviar el código 200 indicando que la petición ha tenido éxito. En este caso, también podemos especificar que la ruta use el método DELETE.

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
@via_bp.route('/<viaId>', methods=['DELETE'])
@load_via
@delete_file
def delete(viaId, via):
    db.session.delete(via)
    db.session.commit()
    return jsonify({}), 200
```
</div>
<div class="original" hidden>
```python
@via_bp.route('/<viaId>/delete', methods=['POST'])
@load_via
@delete_file
def delete(viaId, via):
    if request.form.get('_method') == 'DELETE':
        db.session.delete(via)
        db.session.commit()
        return redirect('/vias')
    return "Method Not Allowed", 405
```
</div>
</div>


<br>

Otro cambio relevante ha sido suprimir `/delete` del path de la ruta, ya que en las SPAs no se usan rutas para acciones específicas, si no que se usan los métodos HTTP para especificar la acción que se quiere realizar. En este caso, el método DELETE. Y con esto, ya tendríamos una **RESTful API completa** para las vías.

## Cliente

Desde el nuevo fichero `vias/index.html` precargaremos el resto de ficheros HTML que necesitemos. Para ello, haremos uso de JINJA2 para cargar los ficheros HTML. 

```html
{% extends 'layout.html' %}
{% block content %}
<link rel="stylesheet" href="/stylesheets/via.css"></link>
<script src="/javascripts/manage_vias.js"></script> 

<div class="via">
    <h1>Vias</h1>
    {% include 'list.html' %}
    {% include 'new.html' %}
    {% include 'show.html' %}
</div>

{% endblock %}
```

Basicamente el fichero será un agregador del resto de ficheros HTML que necesitamos para componer la SPA. En este caso no se usará el fichero edit.html ya que se podrá reusar el mismo formulario del HTML de `new.html` para editar una vía. El resto de ficheros nos quedará de la siguiente manera:

### list

En el fichero `list.html` sobretodo tendremos que eliminar todas las referencias a JINJA.


```html
{% block content %}

<a id="newVia">Nueva Via</a>

<!-- Filtros -->
<form method="GET" action="/vias">
    <label for="min_altura">Altura mínima:</label>
    <input type="number" id="min_altura" name="min_altura" placeholder="Altura mínima" step="0.1">
    
    <label for="max_altura">Altura máxima:</label>
    <input type="number" id="max_altura" name="max_altura" placeholder="Altura máxima" step="0.1">
    
    <label for="grado">Grado de Dificultad:</label>
    <select id="grado" name="grado">
        <option value="all">All</option>
        <option value="4a">4a</option>
        <option value="4b">4b</option>
        <option value="4c">4c</option>
        <option value="5a">5a</option>
        <option value="5b">5b</option>
        <option value="5c">5c</option>
        <option value="6a">6a</option>
        <option value="6b">6b</option>
        <option value="6c">6c</option>
        <option value="7a">7a</option>
        <option value="7b">7b</option>
        <option value="7c">7c</option>
        <option value="8a">8a</option>
        <option value="8b">8b</option>
        <option value="8c">8c</option>
        <option value="9a">9a</option>
        <option value="9b">9b</option>
        <option value="9c">9c</option>
    </select>

    <button type="submit">Filtrar</button>
</form>
<table>
    <thead>
        <tr>
            <th>Imagen</th>
            <th>Nombre</th>
            <th>Grado</th>
        </tr>
    </thead>
    <tbody>
        
    </tbody>
</table>
{% endblock %}
```

En este caso, hemos hecho los siguientes cambios: 

- Hemos borrado las referencias al javascript y a los estilos ya que se encuentran en el fichero `index.html`.
- Hemos eliminado el href de la etiqueta `<a>` ya que el formulario se abrirá dinámicamente desde el fichero `manage_vias.js`.
- Hemos vuelto a poner los grados de dificultad sin usar Jinja2, ya que en este caso, no necesitamos que se carguen dinámicamente. Realmente, **antes tampoco porque se consideran datos estáticos y cuando son datos estáticos no tiene sentido cargarlos dinámicamente**.
- Hemos elimiando todas los filas de la tabla que se precargaban dinamicamente con Jinja2.

Editaremos fichero `public/javascript/manage_vias.js` desde 0 para ir añadiendo las funciones que necesitemos. En este primer caso, lo editaremos para que se pidan las vías al servidor y se muestren en la tabla.

```javascript
document.addEventListener('DOMContentLoaded', async function() {

    let vias = [];

    async function getVias() {
        try {
            const response = await fetch('/vias');
            const data = await response.json();
            vias = data.vias;
            fillVias(vias);
        } catch (error) {
            console.error('Error fetching vias:', error);
        }
    }

    function fillVias(vias) {
        const tbody = document.querySelector('tbody');
        tbody.innerHTML = '';
        for (let via of vias) {
            const tr = document.createElement('tr');
            tr.setAttribute('id', `via-${via.id}`);
            tr.innerHTML = `
                <td>
                    <img src="/thumbnails/${via.imagen}" alt="Imagen de la vía" width="50" height="50">
                </td>
                <td>${via.nombre}</td>
                <td>${via.grado}</td>
                <td>
                    <button class="showVia">Ver</button>
                    <button class="editVia"> Editar</button>
                    <button class="deleteVia">Eliminar</button>
                </td>
            `;
            tbody.appendChild(tr);
        }
    }
    
    getVias();
});
```


Este código hace lo siguiente:

- Crea un array vacio de vias que se usará para almacenar las vías obtenidas del servidor.
- Espera a que se cargue la página entera al tratar el evento 'DOMContentLoaded'.
- Hace una petición fetch a la ruta '/vias' para obtener todas las vías. Esta petición es de tipo asíncrona, por lo que se usa async/await para esperar a que se resuelva la petición.
- Convierte la respuesta a un objeto JSON ya el método fetch devuelve un String.
- El resto es usar código javascript para rellenar la tabla con las vías obtenidas. Como se puede se ha creado una funcion `fillVias` que contiene un bucle for que permite iterar sobre las vías obtenidas y generar el código HTML que luego se inyecta en la tabla con la función `appendChild`. Al principio de la función se limpia el contenido de la tabla con `tbody.innerHTML = '';` para que no se dupliquen las vías cuando se vuelva a llamar a esta función.


### new

Para el caso de new, crearemos un formulario que se abrirá dinámicamente cuando se haga click en el enlace de `Nueva Vía`. Este formulario será un dialogo modal que se abrirá encima de la página actual. Para ello, usaremos la etiqueta `<dialog>` de HTML5. 


```html
{% block content %}
<dialog id="viaEditDialog">
    <div class="formulario">
        <form id="viaForm" action="/vias" method="post" enctype="multipart/form-data">
            <div class="form-group">
                <label for="nombre">Nombre</label>
                <input type="text" class="form-control" id="nombre" name="nombre" placeholder="Nombre" autocomplete="off" required>
                <label for="grado">Grado</label>
                <select name="grado" id="grado" class="form-control">
                    <option value="4a">4a</option>
                    <option value="4b">4b</option>
                    <option value="4c">4c</option>
                    <option value="5a">5a</option>
                    <option value="5b">5b</option>
                    <option value="5c">5c</option>
                    <option value="6a">6a</option>
                    <option value="6b">6b</option>
                    <option value="6c">6c</option>
                    <option value="7a">7a</option>
                    <option value="7b">7b</option>
                    <option value="7c">7c</option>
                    <option value="8a">8a</option>
                    <option value="8b">8b</option>
                    <option value="8c">8c</option>
                    <option value="9a">9a</option>
                    <option value="9b">9b</option>
                    <option value="9c">9c</option>
                </select>
        
                <label for="imagen">Imagen</label>
                <input type="file" class="form-control" id="file" name="file" placeholder="Upload files" autocomplete="off">
        
        
                <label for="altura">Altura</label>
                <input type="number" class="form-control" id="altura" name="altura" placeholder="Altura" step="0.1" required>

                <label for="numero_chapas">Numero de chapas</label>
                <input type="number" class="form-control" id="numero_chapas" name="numero_chapas" placeholder="Numero de chapas" step="1" required>
        
                <label for="desplome">Desplomada</label>
                <select name="desplome" id="desplome" class="form-control">
                    <option value="true">Sí</option>
                    <option value="false">No</option>
                </select>
            </div>
            <button id="submitButton" type="submit">Crear</button>
            <button id="cancelVia">Cancel</button>
        </form>
    </div>
</dialog>
{% endblock %}
```

<blockquote>
<h4>Inciso: Dialogos de HTML</h4>
<p>
    Los dialogos modales son ventanas emergentes que se superponen sobre el contenido de la página y que requieren que el usuario realice una acción antes de poder continuar. Estos formularios son muy útiles para capturar información del usuario de manera rápida y sencilla. Se crean y cierren dinámicamente desde el código javascript del cliente.
</p>
</blockquote>
<br>

Y desde el fichero `manage_vias.js` añadiremos el siguiente código para que se abra el dialogo modal cuando se haga click en el enlace de `Nueva Vía`.

```javascript
document.addEventListener('DOMContentLoaded', async function() {

    ...

    const viaEditDialog = document.getElementById('viaEditDialog');
    const newVia = document.getElementById('newVia');
    const cancelVia = document.getElementById('cancelVia');
    const viaForm = document.getElementById('viaForm');

    newVia.addEventListener('click', function() {
        viaEditDialog.showModal();
    });

    cancelVia.addEventListener('click', function() {
        viaEditDialog.close();
    });

    viaForm.addEventListener('submit', async function(event) {
        event.preventDefault();
        const formData = new FormData(viaForm);
        try {
            const response = await fetch('/vias', {
                method: 'POST',
                body: formData
            });
            const data = await response.json();
            vias.push(data.via);
            fillVias(vias);
            viaEditDialog.close();
        } catch (error) {
            console.error('Error creating via:', error);
        }
    });
});
```

Este código hace lo siguiente:

- Obtiene el dialogo modal y los botones de `Nueva Vía` y `Cancel` del DOM. El botón cancel se encuentra dentro del dialogo modal.
- Añade un evento de click al botón de `Nueva Vía` que abre el dialogo modal.
- Añade un evento de click al botón de `Cancel` que cierra el dialogo modal.
- Añade un evento de submit al formulario que se ejecuta cuando se envía el formulario. Este evento se encarga de prevenir el comportamiento por defecto del formulario `event.preventDefault()`, que es recargar la página. Parsea el formulario con FormData y hace una petición fetch al servidor para crear una nueva vía. Si la petición es exitosa, añade la vía al array de vías, llama a la función fillVias para renderizar de nuevo todas las vías y cierra el dialogo modal.


### show

En el caso del show, podemos usar también los dialogos para mostrar. Para ello, editaremos el fichero `show.html` de la siguiente manera:

```html
{% block content %}
<dialog id="viaShowDialog">
<div class="show">
    <h1 id="showId"></h1>
    <p>
        <span>Nombre: </span>
        <span id="showName"></span>
    </p>
    <p>
        <span>Grado: </span>
        <span id="showGrado"></span>
    </p>
    <p>
        <span>Altura:  </span>
        <span id="showAltura"></span>
    </p>
    <p>
        <span>Desplome:  </span>
        <span id="showDesplome"></span>
    </p>
    <p>
        <span>Numero chapas:  </span>
        <span id="showChapas"></span>
    </p>
    <p>
        <img id="showImage" src="" alt="via image">
    </p>
</div>
<footer>
    <button id="closeShow">Close</button>
</footer>
</dialog>
{% endblock %}
```

En este caso, podemos ver que se han añadido una serie de ids por cada uno de los atributos que queremos mostrar a través del dialogo modal. Y desde el fichero `manage_vias.js` añadiremos el siguiente código:

```javascript
document.addEventListener('DOMContentLoaded', async function() {

    ...

    const viaShowDialog = document.getElementById('viaShowDialog');
    const closeShow = document.getElementById('closeShow');

    function addViaHandlers() {
        const buttonsShow = document.getElementsByClassName('showVia');
        for (button of buttonsShow) {
            button.addEventListener('click', handleShowVia);
        }
    }

    function handleShowVia() {
        const viaId = this.parentNode.parentNode.getAttribute('id').split('-')[1];
        const via = vias.find(via => via.id == viaId);

        let showId = document.getElementById('showId')
        let showName = document.getElementById('showName')
        let showGrado = document.getElementById('showGrado')
        let showAltura = document.getElementById('showAltura')
        let showDesplome = document.getElementById('showDesplome')
        let showChapas = document.getElementById('showChapas')
        let showImage = document.getElementById('showImage')
        showId.textContent = '';
        showName.textContent = '';
        showGrado.textContent = '';
        showAltura.textContent = '';
        showDesplome.textContent = '';
        showChapas.textContent = '';
        showImage.src = '';

        showId.textContent = via.id;
        showName.textContent = via.nombre;
        showGrado.textContent = via.grado;
        showAltura.textContent = via.altura;
        showDesplome.textContent = via.desplome;
        showChapas.textContent = via.numero_chapas;
        showImage.src = `/download/${via.imagen}`;
        viaShowDialog.showModal();
    }

    closeShow.addEventListener('click', function() {
        viaShowDialog.close();
    });

    function fillVias(vias) {
        ...


        addViaHandlers();
    }


});
```

Este código hace lo siguiente:

- Obtiene el dialogo modal y `Close` del show dialog del DOM. El botón close se encuentra dentro del dialogo modal.
- Crea una función llamada `addViaHandlers` que añade un evento de click a cada botón de `Ver` de cada vía. A ese evento se le asocia la función `handleShowVia`.La función `addViaHandlers` se llama al final de la función `fillVias` para que se añada el evento de click a cada botón de `Ver` de cada vía cada vez que se cambien los elementos de la tabla. 
- La función `handleShowVia` se encarga de abrir el dialogo modal y mostrar la información de la vía seleccionada. Primero limpia el contenido del dialogo modal y luego obtiene el id de la vía seleccionada. El id lo obtiene del atributo id de la fila de la tabla. Luego busca la vía en el array de vías y muestra la información en el dialogo modal. Busca la vía en el array de vías y muestra la información en el dialogo modal.
- Añade un evento de click al botón de `Close` que cierra el dialogo modal.


En algunas ocasiones, puede tener sentido ahorrarnos hacer una petición para obtener la información de un recurso en concreto ya que la información ya estará en la lista de recursos que hemos obtenido previamente. En este caso, se podría pasar la información de la vía a mostrar al dialogo modal desde el evento de click del botón de `Ver`. De esta manera, se evitaría hacer una petición extra al servidor. Sin embargo, en otras ocasiones puede tener sentido mantener esta funcionalidad así. Por ejemplo, cuando el recurso tiene muchos atributos definidos o cuando hay recursos asociados que también se quieren mostrar. 

### edit

Para el caso del edit lo que haremos es reusar el formulario de `new.html` y a través del botón de `Editar` en cada fila de la tabla de vías abririmos el formulario pero con la información ya precargada. Para ello, editaremos el fichero `manage_vias.js` de la siguiente manera:

```javascript
document.addEventListener('DOMContentLoaded', async function() {

    ...

    /// EDITADO DE APARTADOS ANTERIORES
    function addViaHandlers() {
        const buttonsShow = document.getElementsByClassName('showVia');
        const buttonsEdit = document.getElementsByClassName('editVia');
        for (button of buttonsShow) {
            button.addEventListener('click', handleShowVia);
        }
        for (button of buttonsEdit) {
            button.addEventListener('click', handleEditVia);
        }
    }

    /// EDITADO DE APARTADOS ANTERIORES
    newVia.addEventListener('click', function() {
        viaForm.action = `/vias`;

        document.getElementById('submitButton').textContent = 'Crear';
        let inputNombre = document.getElementById('nombre');
        let selectGrado = document.getElementById('grado');
        let inputAltura = document.getElementById('altura');
        let inputNumeroChapas = document.getElementById('numero_chapas');
        let inputDesplome = document.getElementById('desplome');
        inputNombre.value = '';
        selectGrado.value = undefined;
        inputAltura.value = undefined;
        inputNumeroChapas.value = undefined;
        inputDesplome.value = undefined;

        viaEditDialog.showModal();
    });

    function handleEditVia() {
        const viaId = this.parentNode.parentNode.getAttribute('id').split('-')[1];
        const via = vias.find(via => via.id == viaId);

        viaForm.action = `/vias/${via.id}`;

        document.getElementById('submitButton').textContent = 'Editar';
        let inputNombre = document.getElementById('nombre');
        let selectGrado = document.getElementById('grado');
        let inputAltura = document.getElementById('altura');
        let inputNumeroChapas = document.getElementById('numero_chapas');
        let inputDesplome = document.getElementById('desplome');
        inputNombre.value = via.nombre;
        selectGrado.value = via.grado;
        inputAltura.value = via.altura;
        inputNumeroChapas.value = via.numero_chapas;
        inputDesplome.value = via.desplome;

        viaEditDialog.showModal();
    }

    /// EDITADO DE APARTADOS ANTERIORES
    viaForm.addEventListener('submit', async function(event) {
        event.preventDefault();
        const formData = new FormData(viaForm);
        const action = viaForm.action;
        const method = (action.includes('/vias/')) ? 'PUT' : 'POST';
        try {
            const response = await fetch(action, {
                method: method,
                body: formData
            });
            const data = await response.json();
            if (method === 'POST') {
                vias.push(data.via);
            } else {
                const index = vias.findIndex(via => via.id == data.via.id);
                vias[index] = data.via;
            }
            fillVias(vias);
            viaEditDialog.close();
        } catch (error) {
            console.error('Error editing via:', error);
        }
    });
});
```

El código hace lo siguiente:

- Editamos la función `addViaHandlers` para que añada un evento de click a cada botón de `Ver` y `Editar` de cada vía. A cada evento se le asocia la función `handleShowVia` y `handleEditVia` respectivamente.
- Edita la captura del evento de cuando se crea una nueva vía para que se abra el dialogo modal con el formulario vacío. Es decir, vacia de valor cada uno de los campos del formulario.
- Añade un evento de click a cada botón de `Editar` que abre el dialogo modal y muestra la información de la vía seleccionada. Obtiene el id de la vía seleccionada del DOM. El id lo obtiene del atributo id de la fila de la tabla. Luego busca la vía en el array de vías y muestra la información en el dialogo modal. Busca la vía en el array de vías y rellena los valores de cada uno de los campos del formulario con la información de la vía seleccionada.
- Edita el evento de submit del formulario para que se envíe una petición PUT si se está editando una vía y una petición POST si se está creando una nueva vía. Para ello, se comprueba si la acción del formulario contiene la cadena '/vias/' para saber si se está editando una vía o creando una nueva. Si se está editando, se envía una petición PUT, si se está creando, se envía una petición POST. Luego se usa también el método para comprobar que hacer con la respuesta del servidor. Si se está creando una nueva vía, se añade la vía al array de vías, si se está editando una vía, se actualiza la vía en el array de vías. Luego se renderiza de nuevo todas las vías y se cierra el dialogo modal.

### delete

En cuanto a la función de delete, lo que haremos será añadir un evento de click a cada botón de `Eliminar` de cada vía. Y desde el fichero `manage_vias.js` añadiremos el siguiente código:

```javascript
document.addEventListener('DOMContentLoaded', async function() {

    ...

    /// EDITADO DE APARTADOS ANTERIORES
    function addViaHandlers() {
        const buttonsShow = document.getElementsByClassName('showVia');
        const buttonsEdit = document.getElementsByClassName('editVia');
        const buttonsDelete = document.getElementsByClassName('deleteVia');
        for (button of buttonsShow) {
            button.addEventListener('click', handleShowVia);
        }
        for (button of buttonsEdit) {
            button.addEventListener('click', handleEditVia);
        }
        for (button of buttonsDelete) {
            button.addEventListener('click', handleDeleteVia);
        }
    }

    async function handleDeleteVia() {
        const viaId = this.parentNode.parentNode.getAttribute('id').split('-')[1];
        try {
            const response = await fetch(`/vias/${viaId}`, {
                method: 'DELETE'
            });
            if (response.ok) {
                vias = vias.filter(via => via.id != viaId);
                fillVias(vias);
            }
        } catch (error) {
            console.error('Error deleting via:', error);
        }
    }

});
```

El código hace lo siguiente:

- Edita la función `addViaHandlers` para que añada un evento de click a cada botón de `Ver`, `Editar` y `Eliminar` de cada vía. A cada evento se le asocia la función `handleShowVia`, `handleEditVia` y `handleDeleteVia` respectivamente.
- La función `handleDeleteVia` se encarga de obtener el id de la vía seleccionada y hacer una petición fetch al servidor para eliminar la vía. Si la petición es exitosa, elimina la vía del array de vías y renderiza de nuevo todas las vías.

### filtro de búsqueda

Para implementar el filtro de búsqueda tendremos que añadir un evento de submit al formulario de filtros y hacer una petición fetch al servidor con los parámetros de búsqueda. Para ello, editaremos el ficheros `list.html` para añadir un id al formulario de filtros:

```html

...

<form id="filter" method="GET" action="/vias">

...
```

Y editaremos el fichero `manage_vias.js` de la siguiente manera:

```javascript
document.addEventListener('DOMContentLoaded', async function() {

    ...

    const filterForm = document.getElementById('filter');
    filterForm.addEventListener('submit', async function(event) {
        event.preventDefault();
        const minAltura = document.getElementById('min_altura').value;
        const maxAltura = document.getElementById('max_altura').value;
        const grado = document.getElementById('grado').value;
        try {
            const response = await fetch(`/vias?min_altura=${minAltura}&max_altura=${maxAltura}&grado=${grado}`);
            const data = await response.json();
            vias = data.vias;
            fillVias(vias);
        } catch (error) {
            console.error('Error fetching vias:', error);
        }
    });

});
```

Este código hace lo siguiente:

- Obtiene el formulario de filtros del DOM.
- Añade un evento de submit al formulario que se ejecuta cuando se envía el formulario. Este evento se encarga de prevenir el comportamiento por defecto del formulario `event.preventDefault()`, que es recargar la página. Obtiene los valores de los campos de los filtros y hace una petición fetch al servidor con los parámetros de búsqueda. Si la petición es exitosa, actualiza el array de vías con las vías obtenidas y renderiza de nuevo todas las vías.

En estos formularios es común añadir un botón de reset del formulario para que se limpien los campos de los filtros. Basicamente sería capturar un click de dicho bóton para que se envíe el formulario con los campos vacíos.

--- 

Y con esto, habriamos cambiado parte de nuestra aplicación para que sea del estilo de SPA. Como se ha comentado varias veces, **lo ideal es que sea toda la aplicación la que sea una SPA**. No obstante, como habrá visto, el código de la SPA es bastante más complejo que el código de la aplicación original y se requiere saber programar en Javascripy. No obstante, **el resultado a través de una SPA es mucho más atractivo y rápido para el usuario**.

**Actualmente las SPAs no se desarrollan con código JS directamente si no que se usan frameworks de JavaScript como React, Angular o Vue.js**, que permiten cargar contenido de manera asíncrona sin tener que recargar toda la página. Estos frameworks proporcionan herramientas y patrones de diseño que facilitan la creación de SPAs, como la gestión de rutas, la actualización del estado de la aplicación y la comunicación con el servidor.


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
