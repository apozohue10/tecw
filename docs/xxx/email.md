# Email

El envío de correos electrónicos es una funcionalidad esencial en muchos servidores web. Las aplicaciones web suelen necesitar enviar correos para diversas finalidades, como:

- Verificación de cuentas: Envío de enlaces de activación tras el registro de un usuario.
- Restablecimiento de contraseñas: Generación de enlaces temporales para recuperar credenciales.
- Notificaciones automáticas: Alertas sobre eventos importantes dentro de la aplicación.
- Facturación y confirmaciones: Envío de recibos y confirmaciones de pedidos.

Para gestionar estos envíos, las aplicaciones no suelen depender de clientes de correo tradicionales, sino que integran pasarelas de emails. Estas pasarelas permiten que el servidor web envíe correos electrónicos mediante protocolos estándar como SMTP o APIs especializadas ofrecidas por servicios externos como SendGrid, Mailgun o Amazon SES.

En un servidor web desarrollado con Flask, se puede integrar el envío de correos electrónicos utilizando Flask-Mail para SMTP. Para ello, lo primero es instalar la dependencia:

```bash
pip install Flask-Mail
```

Una vez instalado, se debe actualizar el fichero `requirements.txt` con la nueva dependencia instalada:

```bash
pip freeze > requirements.txt
```

A continuación debemos configurar la pasarela de correo en la aplicación Flask. Por ejemplo, para configurar el envío de correos a través de Gmail debemos poner en el archivo `app.py`:


```python
from flask_mail import Mail

...

# Configuración SMTP con Gmail
app.config['MAIL_SERVER'] = 'smtp.gmail.com'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = '<email>'
app.config['MAIL_PASSWORD'] = '<password>'  # Mejor usa variables de entorno
app.config['MAIL_DEFAULT_SENDER'] = '<email>'

mail = Mail(app)
```

Donde cada uno de los parámetros de configuración se refiere a:

- `MAIL_SERVER`: Servidor SMTP de Gmail. Es `smtp.gmail.com` para Gmail. A través de este servidor se enviarán los correos.
- `MAIL_PORT`: Puerto de conexión al servidor SMTP. Es `587` para Gmail.
- `MAIL_USE_TLS`: Habilita el uso de TLS para la conexión segura. Es decir, para cifrar los correos.
- `MAIL_USERNAME`: Dirección de correo electrónico desde la que se enviarán los correos. En este caso se debe configurar la dirección de Gmail del alumno.
- `MAIL_PASSWORD`: Contraseña creada en la cuenta de Gmail. Para ello, podemos ir al siguiente [enlace](https://myaccount.google.com/apppasswords) donde podremos crear una contraseña de aplicación para nuestra cuenta de Gmail. Esa contraseña será la que debemos poner en este campo. En general, es recomendable no escribir la contraseña directamente en el código, sino almacenarla en una variable de entorno, pero en este caso se ha puesto directamente para simplificar el ejemplo.
- `MAIL_DEFAULT_SENDER`: Dirección de correo electrónico desde la que se enviarán los correos. En este caso se debe configurar la dirección de Gmail del alumno.

Una vez configurado, podemos generar un nuevo fichero llamado `handle_email.py` donde se definirá una función para enviar correos electrónicos. Por ejemplo, para enviar un correo con un asunto, un destinatario y un cuerpo, se puede definir la siguiente función:

```python
from flask_mail import Message
from app import mail

def send_email(asunto, dest, body):
    msg = Message(asunto, recipients=dest, body=body)
    mail.send(msg)
    return "Correo enviado"
```

Esta función la podemos importar desde otras funciones para enviar correos electrónicos. Por ejemplo, si queremos enviar un correo para confirmar el registro de un usuario, podemos hacerlo modificando el código de la función `register` del blueprint `auth.py`:

```python
from handle_email import send_email

...

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
    send_email('Registro exitoso', [email], 'Registro exitoso, ahora puedes iniciar sesión.')
    flash('Comprueba tu bandeja de correo.')
    return redirect('/')
```

Si se fija, el correo se está enviando una vez el usuario se ha almacenado con éxito en la base de datos. Comentar también, que en nuestro caso estamos enviando texto plano pero que también se pueden enviar correos con formato HTML. Para ello, se puede modificar el cuerpo del correo de la siguiente forma:

```python
body="<h1>Registro exitoso</h1><p>Registro exitoso, ahora puedes iniciar sesión.</p>"
msg = Message(asunto, recipients=dest, html=body)
```


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
│   ├── email.py                # Contiene funciones para el envío de correos
│   └── handle_files.py         # Middlewares para gestionar ficheros
├── requirements.txt            # Lista de dependencias del proyecto
└── env/                        # Entorno virtual
```

---

En cuanto a lo que se refiere al autorregistro de usuarios, es recomendable que se utilice un sistema de verificación de correo electrónico. Por lo general, cuando un usuario se registra en una aplicación se le envía un correo con un enlace que contiene un código de confirmación que nuestro propio servidor web usará para confirmar que al usuario le pertenece ese correo. O por ejemplo, cuando a un usuario se le olvida la contraseña se le envía un correo con un enlace (que incluye un código) para restablecerla usando un nuevo formulario. Por otro lado, también suele ser recomendable que el correo se envíe de forma asíncrona para no bloquear la ejecución de la aplicación.

En este caso hemos usado nuestra cuenta de Gmail para enviar el correo. Pero cuando se despliega una apliación en producción es recomendable usar un servicio de envío de correos como SendGrid, Mailgun o Amazon SES. Estos servicios ofrecen APIs especializadas para el envío de correos y suelen ofrecer la creación de cuentas de correo profesionales desde las que se enviarán los correos. Además, suelen tener una mejor tasa de entrega de correos que los servidores SMTP tradicionales. 