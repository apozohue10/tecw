# Templates

Antes hemos visto que los ficheros HTML que devolviamos a los usuarios los integrabamos dentro del código en Python. Pero podemos traspasar todo ese código a ficheros HTML en la carpeta templates. Para ello, Flask utiliza un motor de plantillas llamado **Jinja2**. Este motor de plantillas nos permite integrar código Python en nuestros ficheros HTML, además de ofrecer otras funcionalidades. Pero primero pasemos el código ficheros HTML y veamos como podemos integrarlos en nuestra aplicación.


En la carpeta templates generamos los siguientes ficheros HTML:

<h4>home.html</h4>
```html
<!DOCTYPE html> 
<html>
    <head>
        <title>Mi Primer Rocódromo</title>
    </head>
    <body>
        <h1>¡Bienvenido a mi primer rocódromo!</h1>
        <p>Este es un sitio web donde podrás encontrar información sobre escalada y rutas disponibles.</p>
    </body>
</html>
```

<h4>about.html</h4>
```html
<!DOCTYPE html>
<html>
<head>
    <title>Mi Primer Rocódromo</title>
</head>
<body>
    <h1>¡Conócenos un poco más!</h1>
</body>
</html>
```

<h4>city.html</h4>
```html
<!DOCTYPE html>
<html>
<head>
    <title>Mi Primer Rocódromo</title>
</head>
<body>
    <h1>¡Hola! Bienvenido al rocódromo de Madrid.</h1>
</body>
</html>
```

<h4>availability.html</h4>
```html
<!DOCTYPE html>
<html>
<head>
    <title>Mi Primer Rocódromo</title>
</head>
<body>
    <h2>La vía de escalada elGigange está disponible para su uso.</h2>
</body>
</html>
```

Ahora, en el fichero app.py, modificamos las rutas para que devuelvan los ficheros HTML que acabamos de crear. Para ello usaremos la función render_template de Flask, aunque esta función en realidad es de Jinja2 y es la que nos permite generas las plantillas HTML que se enviaran a los usuarios posteriormente.

```python
from flask import Flask, request, render_template # Importamos la función render_template

app = Flask(__name__, template_folder='templates') # Indicamos la carpeta donde se encuentran los templates

@app.route('/') 
def home():
    return render_template('home.html')

@app.route('/about')  
def about():
    return render_template('about.html')

@app.route('/ciudad/<ciudad>')  
def ciudad(ciudad):
    return render_template('city.html')

@app.route('/disponibilidad')  
def disponibilidad():
    return render_template('availability.html')
```

Ahora, si ejecutamos la aplicación y accedemos a las rutas, veremos que se renderizan los ficheros HTML que acabamos de crear.

No obstante, tenemos varios problemas: 
<ul>
  <li><b>Primero</b>, si queremos pasar una variable como en el caso de la función ciudad o disponibilidad, no podemos hacerlo como se hacía antes con los strings.</li>
  <li><b>Segundo</b>, repetimos gran cantidad de código en los ficheros HTML.</li>
</ul>

Para resolver estos problemas podemos usar Jinja2 y sus funcionalidades.

## ¿Qué es Jinja2?

**Jinja2** es un motor de plantillas para Python que permite generar contenido dinámico en aplicaciones web. Se utiliza ampliamente en frameworks como **Flask** y **Django** para la renderización de HTML a partir de datos proporcionados por el servidor. Se encarga principalmente de la vista según el patrón MVC. 

<div class="img-center">
    <img src="../img/tema2/jinja2.png" alt="Vista" />
</div>


Sus principales usos incluyen:

- **Generación de contenido dinámico** en páginas web.
- **Uso de estructuras de control** como bucles e instrucciones condicionales en plantillas HTML.
- **Herencia de plantillas**, lo que facilita la reutilización de código.
- **Filtrado y manipulación de datos** dentro de las plantillas.

**Jinja2 se instala cuando instalamos Flask** como se puede ver en la lista de dependencias requirements.txt.

---

Jinja2 ofrece múltiples características que facilitan la generación de contenido dinámico:

- **Variables**: Permite insertar variables en una plantilla utilizando `{{ variable }}`.
- **Estructuras de control**: Incluye sentencias `if`, `for`, `elif`, `else`, entre otras.
- **Herencia de plantillas**: Facilita la reutilización de código mediante bloques reutilizables.
- **Filtros y macros**: Permiten manipular datos dentro de la plantilla.

A continuación veremos algunas de ellas

### Uso de variables

Las variables en Jinja2 se insertan con doble llave dentro de nuestro fichero HTML ```<p>Hola, {{ nombre }}!</p>```. Y se deben pasar como argumento a la función render_template ```return render_template('hola.html', nombre='Pedro')```. 

Por ejemplo, si queremos pasar una variable nombre a nuestro fichero HTML, lo podemos hacer de la siguiente manera:

```python
@app.route('/ciudad/<ciudad>')  
def ciudad(ciudad):
    return render_template('city.html', ciudad=ciudad)
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>Mi Primer Rocódromo</title>
</head>
<body>
    <h1>¡Hola! Bienvenido al rocódromo de {{ciudad }}.</h1>
</body>
</html>
```

Jinja2 también proporciona filtros para transformar los datos. Por ejemplo:

```html
<p>{{ mensaje | upper }}</p>  {# Convierte el texto a mayúsculas #}
<p>{{ lista | length }}</p>   {# Muestra la cantidad de elementos de la lista #}
<p>{{ numero | round }}</p>   {# Redondea el número #}
```

#### Ejercicio de clase

Modifica la función disponibilidad para que devuelva el nombre de la vía de escalada en Mayusculas.

### Estrucutras de control

Jinja2 permite el uso de estructuras de control como condicionales en plantillas HTML. Para realizar esto, se utilizan las palabras clave `if`, `for`, `elif`, `else`, entre otras.

```html
{% if ciudad == 'madrid' %}
    <p>En Madrid se encuetran los mejores rocodromos del centro de la peninsula.</p>
{% elif ciudad == 'barcelona' %}
    <p>Los rocodromos de Barcelona son los más visitados de España.</p>
{% else %}
    <p>En {{ ciudad }} también hay rocodromos.</p>
{% endif %}
```

También permite incluir bucles `for` para iterar sobre listas, diccionarios y otros objetos iterables.

```html
<p>Las ciudades con rocodromos son:</p>
<ul>
    {% for nombre in ['madrid', 'barcelona', 'segovia', 'valencia'] %}
        <li>{{ nombre }}</li>
    {% endfor %}
</ul>
```

### Plantillas

Jinja2 permite definir plantillas base HTML para reutilizar código en diferentes páginas. Para ello, se utiliza la directiva `extends` en la plantilla secundaria y se definen bloques reutilizables con la directiva `block`.

Por ejemplo, en nuestro caso, se puede generar un fichero llamado layout.html en la carpeta templates que contenga la estructura base de nuestra página web y luego extenderlo en los ficheros home.html, about.html, city.html y availability.html.

<h4>layout.html</h4>
```html
<!DOCTYPE html> 
<html>
    <head>
        <title>Mi Primer Rocódromo</title>
    </head>
    <body>
        {% block content %}{% endblock %}
    </body>
</html>
```

<h4>home.html</h4>
```html
{% extends 'layout.html' %}
{% block content %}
<h1>¡Bienvenido a mi primer rocódromo!</h1>
<p>Este es un sitio web donde podrás encontrar información sobre escalada y rutas disponibles.</p>
{% endblock %}
```

#### Ejercicio de clase

Integrar el layout en el resto de ficheros HTML.

---

Estas son algunas de las cosas básicas que se puede hacer con Jinja2. En su [página web oficial](https://jinja.palletsprojects.com/en/stable/), puedes encontrar más información sobre cómo utilizar este motor de plantillas.

Hemos visto como se pueden generar varias rutas y devolver ficheros HTML con Flask. Se trata de algo que suele escalar muy rapido y que puede llegar a ser muy complejo. Por ello, en la siguiente lección veremos como podemos estructurar nuestra aplicación para que sea más escalable y mantenible.