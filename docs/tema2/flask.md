# Flask

Flask es un **micro-framework** de desarrollo web para Python. Flask es ligero, flexible y fácil de usar, lo que lo hace ideal para desarrolladores que desean crear aplicaciones web de forma rápida y eficiente. Aunque se le denomina "micro", Flask no limita su funcionalidad, sino que se enfoca en 
proporcionar una base simple que se puede ampliar con complementos según las necesidades del proyecto.

<div class="img-center">
    <img src="../img/tema2/flask.png" alt="Vista" />
</div>

---

El uso básico de Flask es para **enrutamiento** es decir, permite crear el esquema de rutas (mediante URLs) y asociarlas con funciones que se ejecutan cuando los usuarios acceden a esas rutas mediante el protocolo HTTP. Estas funciones pueden devolver un recurso al usuario, crear uno nuevo, borrarlo, etc. A parte, Flask, ofrece algunas de sus funcionalidades destacadas como:


<ul>
  <li>
    <b>Jinja2. </b> Permite para renderizar las plantillas HTML, lo que permite generar contenido dinámico de manera eficiente. Jinja2 es un motor de plantillas potente y flexible que permite incluir variables, bucles y otras estructuras de control en las páginas HTML. Basicamente, lo que permite es que puedas configurar un fichero HTML con la información acorde a cada usuario.
  </li>
  <li>
    <b>Gestión de sesiones y cookies. </b> Flask permite gestionar sesiones de usuario y cookies para mantener la persistencia de datos entre las solicitudes, lo que es útil para crear aplicaciones que requieren autenticación y personalización.
  </li>
  <li class="nested_list">
    <b>Extensiones. </b>Flask es muy flexible y su funcionalidad se puede ampliar mediante diversas extensiones. Algunas de las más populares incluyen:
    <ul>
      <li class="nested_list"><b>Flask-SQLAlchemy</b>: Para la integración con bases de datos.</li>
      <li class="nested_list"><b>Flask-WTF</b>: Para la validación de formularios web.</li>
      <li class="nested_list"><b>Flask-Login</b>: Para la gestión de autenticación de usuarios.</li>
      <li class="nested_list"><b>Flask-Mail</b>: Para enviar correos electrónicos.</li>
    </ul>
  </li>
  <li>
    <b>Soporte para testing. </b>Flask viene con herramientas integradas para realizar pruebas unitarias y pruebas de integración, lo que facilita la creación de aplicaciones robustas y confiables.
  </li>
  <li>
    <b>Desarrollo con soporte para entornos virtuales. </b>Flask se puede integrar fácilmente con entornos virtuales de Python, lo que permite gestionar dependencias específicas del proyecto sin interferir con otras aplicaciones Python en el sistema.
  </li>
</ul>


## Instalación

Para poder instalar Flask se debe tener Python, pip y virtual environments instalado. Este último nos permite instalar librerías para el proyecto en cuestión de tal manera que no entre en interferencia con otros proyectos existentes de Python o incluso con propias funcionalidades del sistema que dependen de Python como ocurre en sistemas operativos en base Linux. 

Como ejemplo a lo largo de la asignatura, vamos a desplegar un servidor web que gestionará los recursos de un **rocódromo**. 

---

El primer paso es preparar el entorno de trabajo con un entorno virtual de Python. Para ello, creamos una carpeta donde estará el código de nuestro servidor y, en una terminal dentro de dicha carpeta ejecutamos:

```bash
python -m venv env
```

Esto nos permite crear un entorno virtual llamado **env** donde se almacenarán todas las dependencias. A continuación activamos dicho entorno:

- En Linux/macOS
```bash
. env/bin/activate
```

- En Windows
```bash
.\env\Scripts\activate
```

Esto permite cambiar el contexto de tu terminal para que todas las instalaciones de paquetes y ejecuciones de scripts se realicen dentro de ese entorno virtual, en lugar de en el sistema global. Podrá observar que la linea de ejecución del terminal precede la palabra (env). En caso de que queramos salir del entorno virtual podemos ejecutar en el terminal: 

```bash
deactivate
```

El siguiente paso es instalar el framework de Flask. Para ello usamos pip dentro del entorno virtual:

```bash
pip install Flask
```

Una vez instalado, podemos comprobar que se ha instalado correctamente con:

```bash
flask --help
```

**IMPORTANTE. El siguiente comando se debe ejecutar siempre que instalemos una dependencia nueva**

```bash
pip3 freeze > requirements.txt 
```

Este comando nos genera o actualiza un fichero requirements.txt que contiene todas las dependencias instaladas en nuestro proyecto y sus versiones. Esto nos permite exportar el proyecto a otros ordenadores y replicarlo. Según se vaya avanzando en el desarrollo del servidor, ese fichero tendrá un aspecto similar a esto:

```plaintext
alembic==1.14.1
blinker==1.9.0
click==8.1.7
Flask==3.1.0
Flask-Migrate==4.1.0
Flask-SQLAlchemy==3.1.1
greenlet==3.1.1
itsdangerous==2.2.0
Jinja2==3.1.4
Mako==1.3.8
MarkupSafe==3.0.2
mysql-connector-python==9.2.0
SQLAlchemy==2.0.37
SQLAlchemy-Utils==0.41.2
typing_extensions==4.12.2
Werkzeug==3.1.3
```

Con todo esto, nuestro sistema de ficheros deberá tener un aspecto al siguiente:

```plaintext
rocodromo/
│
├── requirements.txt        # Lista de dependencias del proyecto
└── env/                    # Entorno virtual
```

## Primer despliegue

Una vez instalado el entorno, podemos desplegar nuestro primer servidor web. Para ello deberemos generar una carpeta donde se almacenara todo nuestro código. Esta carpeta se llamará app. Dentro de esta carpeta createmos el fichero app.py el cual será el fichero de arranque del servicio. El sistema de ficheros de nuestro proyecto quedará de la siguiente manera. 


```plaintext
rocodromo/
│
├── app/                    # Carpeta principal de la aplicación
│   ├── app.py              # Inicializa la aplicación y las configuraciones
├── requirements.txt        # Lista de dependencias del proyecto
└── env/                    # Entorno virtual
```

En el fichero app.py, incluimos el siguiente código. Este código esta lleno de comentarios para el entendimiento de que hace cada línea. No es necesario copiar todos los comentarios.

```python
from flask import Flask  # Importa la clase Flask desde el módulo flask

# Crear una instancia de la aplicación
app = Flask(__name__)  # Crea una instancia de la aplicación Flask.

# Definir una ruta
@app.route('/') 
def home():  # Define una función llamada home que se ejecutará cuando el usuario acceda a la ruta '/'
    return """ 
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
    """

# Ejecutar la aplicación
if __name__ == '__main__':
    app.run(debug=True) 
```

La última línea del fichero app.py permite ejecutar la aplicación en modo de depuración, lo que permite ver errores detallados. Además, permite recargar la aplicación automáticamente al cambiar el código. Esto último agilizará el desarrollo del código.

Podemos ejecutar el servidor con:

```bash
flask --app app/app.py run
```

Para no especificar `--app` podemos crear una variable de entorno que apunte al fichero app.py:

- En Linux/macOS
```bash
export FLASK_APP=app/app.py
```

- En Windows
```bash
set FLASK_APP=app/app.py
```

Y una vez hecho eso, también podemos arrancarlo en modo debug para que **se reinicie con cada cambio**, lo que agiliza el desarrollo:

```bash
flask --debug run      # Rearranca con cada cambio
```

Por defecto, Flask arranca el servidor en el puerto 8000. También podemos arrancarlo en otro puerto distinto al de por defecto:

```bash
flask --debug run -p 5000  # Ejecuta en el puerto 5000
```

Para conectarnos al servidor y ver el resultado, podemos introducir la URL [http://localhost:5000](http://localhost:5000) en cualquier navegador como Chrome o Firefox.

Y aquí es donde entra uno de los principales conceptos a entender: URL (Uniform Resource Locator)