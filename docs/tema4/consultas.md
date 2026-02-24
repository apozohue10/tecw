# Queries

Una vez que hemos creado los modelos, migrado la base de datos y rellenado con algunos valores por defecto, es el momento de modificar el código de nuestros blueprints para integrar la base de datos y dotar de persistencia a nuestra aplicación. Para ello, vamos a modificar todas aquellas funciones que impliquen una consulta, creación, edición o borrado de datos en la base de datos. Siguiendo el ejemplo del blueprint de via debemos realizar las siguientes modificaciones:


## 1. Eliminación del array en memoria

Se ha eliminado el array `vias` y en su lugar se realizan consultas a la base de datos a través del modelo `Via` de SQLAlchemy. Para ello, hay que importar el modelo de vias en el blueprint para poder crear objetos de via que se traducirán posteriormente en registros en la base de datos. Se debe importar también el objeto db para poder realizar la conexión.

```python
from models import Via
from app import db

...
```

## 2. after_request y before_request

Antes, `after_request` obtenía el número de vías con el length del array. Ahora realiza una consulta a la base de datos con `Via.query.count()`.

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
@via_bp.before_request
def validar_grado():
    if request.endpoint == "via.create" and request.method == "POST":
        grado = request.form.get("grado")
        if grado not in grados:
            abort(400)

@via_bp.after_request
def agregar_total_vias(response):
    response.headers["X-Total-Vias"] = Via.query.count()
    return response
```
</div>
<div class="original" hidden>
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
</div>
</div>

---

## 2. load_via

Antes, `load_via` buscaba la vía en una lista en memoria utilizando `next()`. Ahora, se usa `Via.query.get(viaId)`.

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
        if viaId is not None:
            via_requested = Via.query.get(viaId)
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
            via_requested = next((via for via in vias if via['id'] == int(viaId)), None)
            if via_requested is None:
                abort(404)
            kwargs['via'] = via_requested
        return f(*args, **kwargs)
    return decorated_function
```
</div>
</div>



---

## 3. list

En lugar de filtrar sobre un array en memoria, ahora se usa SQLAlchemy para construir una consulta dinámica con `filter()`, lo que permite aplicar los filtros directamente en la base de datos y mejorar el rendimiento.

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

    if min_altura is not None:
        query = query.filter(Via.altura >= min_altura)

    if max_altura is not None:
        query = query.filter(Via.altura <= max_altura)

    if grado and grado != 'all':
        query = query.filter(Via.grado == grado)

    vias = query.all()
    
    return render_template('list.html', vias=vias, grados=grados)
```
</div>
<div class="original" hidden>
```python
def list():
    min_altura = request.args.get('min_altura', type=int)
    max_altura = request.args.get('max_altura', type=int)
    grado = request.args.get('grado')

    filtered_vias = vias

    if min_altura is not None:
        filtered_vias = [via for via in filtered_vias if via.get('altura', 0) >= min_altura]

    if max_altura is not None:
        filtered_vias = [via for via in filtered_vias if via.get('altura', 0) <= max_altura]

    if grado and grado != 'all':
        filtered_vias = [via for via in filtered_vias if via.get('grado') == grado]

    return render_template('list.html', vias=filtered_vias, grados=grados)
```
</div>
</div>


---

## 4. Create

En lugar de agregar un diccionario a una lista en memoria, se crea una instancia de `Via` y se guarda en la base de datos con `db.session.add()` y `db.session.commit()`.

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
        imagen=filename
    )
    db.session.add(via)
    db.session.commit()
    
    return redirect('/vias')
```
</div>
<div class="original" hidden>
```python
def create(filename):
    vias.append({
        'id': len(vias) + 1,
        'nombre': request.form.get('nombre'),
        'grado': request.form.get('grado'),
        'imagen': filename
    })
    return redirect('/vias')
```
</div>
</div>

Además, también deberemos añadir un nuevo input al formulario de new.html para incluir el nuevo atributo introducido en el modelo, es decir el numero de chapas.

---

## 5. Update

Antes, los datos se actualizaban en un diccionario dentro de una lista en memoria. Ahora, se modifican directamente en la instancia de `Via`, y se persisten en la base de datos con `db.session.commit()`.

<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
def update(viaId, via, filename):
    via.nombre = request.form.get('nombre')
    via.grado = request.form.get('grado')
    via.imagen = filename
    db.session.commit()
    return redirect('/vias')
```
</div>
<div class="original" hidden>
```python
def update(viaId, via, filename):
    via['nombre'] = request.form.get('nombre')
    via['grado'] = request.form.get('grado')
    via['altura'] = float(request.form.get('altura'))
    via['imagen'] = filename
    return redirect('/vias')
```
</div>
</div>

Además, también deberemos añadir un nuevo input al formulario de edit.html para incluir el nuevo atributo introducido en el modelo, es decir el numero de chapas.

---

## 6. Delete

Antes, se eliminaba un diccionario de una lista en memoria. Ahora, se elimina el objeto `Via` de la base de datos con `db.session.delete()` y se confirma con `db.session.commit()`.


<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
def delete(viaId, via):
    db.session.delete(via)
    db.session.commit()
    return redirect('/vias')
```
</div>
<div class="original" hidden>
```python
def delete(viaId, via):
    vias.remove(via)
    return redirect('/vias')
```
</div>
</div>

## 7. delete_file

Se debe modificar ahora el middleware de borrar imágenes ya que ahora se accede al nombre del fichero a través del atributo `imagen` del objeto `Via` en lugar de acceder a un diccionario.


<div class="toggleCodeContainer">
<div class="toogleButton">
<button class="buttonModificado btn btn-primary btn-sm active">Código nuevo</button>
<button class="buttonOriginal btn btn-primary btn-sm">Código antiguo</button>
</div>
<div class="modificado">
```python
def delete_file(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        via = kwargs['via'] # Obtiene el nombre del fichero de la vía
        filename = via.imagen
        if filename:
            file_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
            if os.path.exists(file_path): # Comprueba si el fichero existe
                os.remove(file_path) # Borra el fichero
        return f(*args, **kwargs)
    return decorated_function
```
</div>
<div class="original" hidden>
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
</div>
</div>

---

Con esto, hemos adaptado nuestro servidor web a funcionar con una base de datos en lugar de un array en memoria. Ahora, cada vez que se realice una consulta, creación, edición o borrado de datos, se realizará directamente en la base de datos. Hasta aqui tendriamos el esquema básico de un servidor web desarrollado con Flask y SQLAlchemy. Esto constituye la base sobre la que construyen otras funcionalidades más avanzadas. En los siguientes apartados se verán algunas mejoras y funcionalidades adicionales que se pueden añadir a nuestro servidor web.



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