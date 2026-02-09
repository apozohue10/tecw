# URL

Una URL (Uniform Resource Locator) es una dirección de acceso a cualquier recurso o servicio en Internet. 

Las **URL** están definidas en el estándar [RFC 1738](https://www.ietf.org/rfc/rfc1738.txt) y son un caso particular de los **URI** (Uniform Resource Identifier), definidos en el [RFC 3986](https://tools.ietf.org/html/rfc3986). Originalmente, las URL fueron creadas para la Web como direcciones de archivos, pero su uso se ha generalizado para acceder a todo tipo de servicios en Internet.  

El formato completo de una URL es:  

<div style="text-align: center;">
    <span style="font-size: 1.5em; color: #4A90E2;">scheme://user:password@host:port/path?query#fragment</span>
</div>
<br>
Donde:  
<ul>
  <li><b>Scheme</b>: Protocolo de acceso utilizado por el servidor (HTTP, HTTPS, FTP, etc.).</li>
<li><b>user:password</b>: <b>Credenciales de acceso (opcional)</b>: Permite incluir un nombre de usuario y una contraseña para autenticarse en algunos servidores. Se usa en URLs como <code>ftp://user:password@servidor.com</code>, aunque su uso en HTTP está en desuso por razones de seguridad.</li>
  <li><b>Host</b> y <b>port</b>: Dirección del servidor y el puerto de conexión (por defecto, HTTP usa el puerto 80 y HTTPS el 443).</li>
  <li><b>Path</b>: Ruta del recurso (por ejemplo, la ubicación de un archivo en el servidor).</li>
  <li><b>Query</b>: Define parámetros con valores asignados, en el formato <code>?key1=valor1&key2=valor2&...</code></li>
  <li><b>Fragment (o Anchor)</b>: Identifica un punto específico dentro del recurso, normalmente asociado a un atributo <code>id="identificador"</code> Por lo general esto es ejecutado por el navegador y te permite visualizar directamente el elemento</li>
</ul>  

Así por ejemplo, en nuestro contexto, cuando hemos accedido a nuestro servidor web a través del navegador, hemos introducido:

<div style="text-align: center;">
    <span style="font-size: 1.5em; color: #4A90E2;">http://localhost:5000</span>
</div>
<br>
Donde:
<ul>
  <li><b><code>http://</code></b> → <b>Scheme</b>: En este caso, el protocolo utilizado para la comunicación es <code>HTTP</code>.</li>
  <li><b><code>localhost</code></b> → <b>Host</b>: Se refiere a la máquina local (tu propio ordenador). <code>localhost</code> es un nombre de dominio especial que apunta a la dirección <code>127.0.0.1</code>, usada para pruebas y desarrollo local.</li>
  <li><b><code>:5000</code></b> → <b>Port</b>: Es el puerto en el que el servidor está escuchando las peticiones. En este caso, Flask usa el puerto <code>5000</code> por defecto cuando ejecutas <code>app.run()</code>.</li>
</ul>

En este caso, no se han añadido los otros elementos de la URL como path o query, pero veamos algunos ejemplos. Para ello se puede editar el código de app.py para añadir algunos ejemplos:


```python
@app.route('/about')  
def about():
    return """ 
        <!DOCTYPE html> 
        <html>
        <head>
            <title>Mi Primer Rocódromo</title>
        </head>
        <body>
            <h1>¡Conócenos un poco más!</h1>
        </body>
        </html>
    """  

@app.route('/ciudad/<ciudad>')  
def ciudad(ciudad): # Flask automaticamente parsea y nos da elementos del path que sean dinamicos
    return f""" 
        <!DOCTYPE html> 
        <html>
        <head>
            <title>Mi Primer Rocódromo</title>
        </head>
        <body>
            <h1>¡Hola! Bienvenido al rocódromo de {ciudad}.</h1>
        </body>
        </html>
    """

# Ruta con query parameters (Ejemplo: http://localhost:5000/disponibilidad?via=ElGigante)
@app.route('/disponibilidad')  
def disponibilidad():
    via = request.args['via']  # Obtiene el parámetro 'via' de la query de la URL
    return f""" 
        <!DOCTYPE html> 
        <html>
        <head>
            <title>Mi Primer Rocódromo</title>
        </head>
        <body>
            <h2>La vía def escalada '{via}' está disponible para su uso.</h2>
        </body>
        </html>
    """
```

Para este último, estamos necesitamos importar la librería **request**. Esta librería nos permite acceder a los parámetros de la URL, tanto los parámetros de ruta (como el parámetro dinámico <code>ciudad</code> en la ruta <code>/ciudad/&lt;ciudad&gt;</code>) como los parámetros de consulta (query parameters) que se pasan después del signo de interrogación en la URL (como el parámetro <code>via</code> en la ruta <code>/disponibilidad?via=ElGigante</code>).

Cuando instalamos Flask, la librería request se instala automáticamente como parte de Flask, por lo que no es necesario instalarla por separado. Sin embargo, para usarla en nuestro código, debemos importarla explícitamente:

```python
from flask import Flask, request
```

Como podemos observar, este enfoque nos permite definir múltiples rutas accesibles desde el navegador. Sin embargo, concentrar todas las rutas en un único archivo y escribir el código HTML como cadenas dentro del código Python puede resultar poco práctico y difícil de mantener. En la siguiente sección, exploraremos cómo estructurar mejor el código de nuestra aplicación. Pero antes, realizaremos un último ejercicio sobre URLs.

#### Ejercicio de clase

En este ejercicio, aprenderás a utilizar el parámetro fragment de la URL para navegar a un punto específico dentro de una página web.

Instrucciones:

1. Crea una página HTML que contenga varias secciones y que cada secciṕn ocupe un espacio determinado.
2. Asigna un identificador único (id) a cada sección.
3. Pruebe a usar fragment en la url para saltar a otras secciones.