# Logging

Un aspecto importante para los servidores es la gestión de los logs es decir de **los mensajes que se generan durante la ejecución de la aplicación**. Se tratan de unos mensajes que no aportan ninguna funcionalidad a los ususarios pero que sin embargo para los desarrolladores son de gran utilidad para depurar errores o para monitorizar el funcionamiento de la aplicación.

Por lo general, estos mensajes se catalogan en 5 niveles de severidad ordenados de menor a mayor:

- `DEBUG`: Mensajes de depuración.
- `INFO`: Mensajes informativos.
- `WARNING`: Mensajes de advertencia.
- `ERROR`: Mensajes de error.
- `CRITICAL`: Mensajes críticos.

Estos mensajes los ira incorporando el desarrollador dentro del código según considere oportuno. Por ejemplo, si se produce un error en una función, se puede añadir un mensaje ERROR para que el desarrollador pueda identificar el problema. O para un mensaje para ver por donde va la ejecución de un programa se puede añadir un mensaje INFO.

A parte, por lo general el desarrollador luego puede configurar el nivel de severidad de los mensajes que quiere que se muestren. Por ejemplo, si se configura el nivel de severidad a INFO, solo se mostrarán los mensajes de severidad INFO, WARNING, ERROR y CRITICAL. Si se configura el nivel de severidad a ERROR, solo se mostrarán los mensajes de severidad ERROR y CRITICAL.

En Python, el módulo `logging` permite gestionar estos mensajes de forma sencilla. Este módulo ya viene incorporado en Python por lo que no es necesario instalar ninguna dependencia adicional. Sin embargo es recomendable instalar la librería `colorlog` para darle color a los mensajes de log, de tal forma que sea más fácil identificarlos. Para ello instalamos la librería con pip:

```bash
pip install colorlog
```

Y guardamos la dependencia en el fichero `requirements.txt`:

```bash
pip freeze > requirements.txt
```

Una vez que lo hemos añadido, podemos configurar los logs en el ficher `app.py` de la siguiente forma:

```python
import logging
import colorlog

...

##### Logs
# Remove the default Flask logger handlers
for handler in app.logger.handlers:
    app.logger.removeHandler(handler)

# Create a handler
handler = colorlog.StreamHandler()

# Create a formatter
formatter = colorlog.ColoredFormatter(
    "%(asctime)s - %(log_color)s%(levelname)s: %(message)s",
    datefmt='%Y-%m-%d %H:%M:%S',
    log_colors={
        'DEBUG': 'cyan',
        'INFO': 'green',
        'WARNING': 'yellow',
        'ERROR': 'red',
        'CRITICAL': 'bold_red',
    }
)
# Set the formatter for the handler
handler.setFormatter(formatter)

# Add the handler to the app's logger
app.logger.addHandler(handler)
app.logger.setLevel(logging.DEBUG)
```

El código hace lo siguiente:

- En primer lugar se importan los módulos `logging` y `colorlog`.
- Se eliminan los manejadores de logs por defecto de Flask. Eso se implementa con el bucle for que recorre todos los manejadores de logs y los elimina.
- Se crea un manejador de logs con `colorlog.StreamHandler()` que implementará la función `formatter` que dará color a los mensajes de log y además mostrará la fecha y hora en el que han ocurrido.
- Añadimos el manejador de logs al logger de la aplicación con `app.logger.addHandler(handler)`.
- Configuramos el nivel de severidad de los mensajes que queremos que se muestren con `app.logger.setLevel(logging.DEBUG)`. Es decir, mostará todos los mensajes de severidad DEBUG, INFO, WARNING, ERROR y CRITICAL.

Una vez configurado, podemos añadir mensajes de log en cualquier parte de la aplicación con `app.logger.debug()`, `app.logger.info()`, `app.logger.warning()`, `app.logger.error()` y `app.logger.critical()`. 

Por ejemplo en el fichero `app.py` podemos añadir un mensaje de info de arranque:

```python
app.logger.info("Servidor arrancado correctamente!")
```

O en el fichero `auth.py` podemos añadir un mensaje de warning si el usuario no se ha podido registrar porque existe otro email en la base de datos:

```python
if User.query.filter_by(email=email).first():
    flash('El email ya está registrado.')
    app.logger.warning("El usuario " + email + " no se ha podido registrar porque ya existe en la base de datos.")
    return redirect('/auth/register')
```

---

**Implementar un sistema de logs es de vital importancia tanto en el desarrollo como cuando desplegamos una aplicación en producción**. Estos mensajes son la primera pista que tendremos para investigar un error o un comportamiento inesperado de la aplicación cuando este en producción. Por ello, es recomendable añadir mensajes de log en todas las partes de la aplicación que consideremos necesarios para poder identificar problemas en el futuro. Cuanto mayor cantidad de logs incluyamos con su severidad correspondiente, más fácil será identificar un problema en el futuro.

Sin embargo, **si por alguna razón nuestro servidor se cae, no podremos acceder a los logs que se han generado en ese momento**. Para resolver este problema existen librerías que permiten almacenar los logs en un fichero o en una base de datos de forma constante, lo cual además permite hacer búsquedas más rápidas. Incluso se podría llegar a integrar con un sistema de monitorización para que nos avise si se produce un error en la aplicación. Sin embargo, esto ya es un tema más avanzado que no se tratará en este curso.
