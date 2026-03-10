# Testando nuestra API y mejorándola

## Postman

Usando postman, es una aplicación que podemos descargar. Es gratuita. No vamos a entrar a explicar mucho en este documento el uso de la aplicación. Sí que es importante decir que es un cliente HTTP que nos permite hacer peticiones HTTP, scripts antes y después de las peticiones, manejo de variables internas y de entorno, ect. A la hora de desarrollar APIs esta herramienta es fundamental.

Dejamos un buen tutorial aquí para su uso.

<iframe width="560" height="315" src="https://www.youtube.com/embed/MFxk5BZulVU?si=OynyHhUaHubKrh1G" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

### Probando la API: Adiós al navegador, hola Postman

Hasta ahora probábamos las rutas en la barra de direcciones del navegador. Esto funcionaba porque el navegador, por defecto, hace peticiones `GET`. Al recibir la cabecera `Content-Type: application/json`, simplemente dibujaba el JSON en pantalla.

Pero para desarrollar APIs de verdad, el navegador es insuficiente:

- No podemos hacer fácilmente peticiones `POST`, `PUT` o `DELETE`.
- No podemos adjuntar un _body_ en formato JSON.
- Es tedioso modificar las cabeceras (_headers_) como la autorización.

**La solución es usar un cliente HTTP.**

- **Postman** (o Insomnia) son herramientas visuales fundamentales en la industria para construir, probar y documentar peticiones HTTP.
- Si prefieres la terminal, puedes usar `curl` (Linux/Mac/Windows) o `Invoke-WebRequest` (PowerShell).

Vamos a probar Postman.

![/img/tema7/postman.png](/img/tema7/postman.png)

## NewDtos y EditDtos

Una vez conocemos Postman, vamos a ver cómo podemos hacer los DTOs para la creación de una nueva vía. Lo primero sería definir el DTO

Definir qué sale de la API es importante, pero definir y validar **qué entra** es crítico para la seguridad.

Para la creación de una nueva vía (método `POST`), no necesitamos recibir un ID, y quizás queramos restringir qué campos nos pueden enviar:

```python
class NewViaDTO(BaseModel):
    nombre: str
    grado: str
    altura: Optional[int] = None
    desplome: bool = True
    imagen: Optional[str] = None
    numero_chapas: Optional[int] = Field(default=None, alias='chapas')
    user_id: str

    model_config = {
        'extra': 'forbid' # Prohíbe que el cliente envíe campos inventados que no estén aquí
    }
```

Así quedaría nuestro controlador seguro:

```python
@vias_api_bp.route('/', methods=['POST'])
def create():
    body = request.get_json()

    try:
        # 1. Pydantic valida el JSON entrante
        dto = NewViaDTO.model_validate(body)

        # 2. Validamos lógica de negocio (ej. que el usuario exista)
        user_query = User.query.get(dto.user_id)
        if user_query is None:
            return jsonify({"error": f"User {dto.user_id} not found"}), 404

        # 3. Persistimos en base de datos trasladando datos del DTO al Modelo
        model = Via(
            nombre=dto.nombre,
            grado=dto.grado,
            altura=dto.altura,
            desplome=dto.desplome,
            imagen=dto.imagen,
            numero_chapas=dto.numero_chapas,
            user_id=dto.user_id
        )
        db.session.add(model)
        db.session.commit()

        # 4. Devolvemos el objeto recién creado usando el DTO de salida
        response_dto = ViaDTO.model_validate(model)
        return jsonify(response_dto.model_dump()), 201 # 201 Created

    except ValidationError as e:
        # Si Pydantic falla, devolvemos un error 400 Bad Request con el detalle exacto
        return jsonify(e.errors()), 400
```

### Actualizaciones parciales (PUT / PATCH)

Para editar una vía, todos los campos deberían ser opcionales, ya que el cliente podría querer actualizar solo la `altura`.

```python
class EditViaDTO(BaseModel):
    nombre: Optional[str] = None
    grado: Optional[str] = None
    altura: Optional[int] = None
    desplome: Optional[bool] = None
    imagen: Optional[str] = None
    numero_chapas: Optional[int] = Field(default=None, alias='chapas')

    model_config = {
        'extra': 'forbid'
    }
```

El controlador usaría una característica genial de Pydantic llamada `exclude_unset=True`, que extrae _solo_ los campos que el cliente ha enviado explícitamente, ignorando los que dejó en blanco:

```python
@vias_api_bp.route('/<int:id>', methods=['PUT'])
def edit(id):
    body = request.get_json()

    via_query = Via.query.get(id)
    if via_query is None:
        return jsonify({"error": f"Via {id} not found"}), 404

    try:
        dto = EditViaDTO.model_validate(body)

        # Magia de Pydantic: extrae solo lo que se envió en el JSON
        campos_actualizados = dto.model_dump(exclude_unset=True)

        # Iteramos y actualizamos el modelo SQL dinámicamente
        for field, value in campos_actualizados.items():
            setattr(via_query, field, value)

        db.session.commit()

        response_dto = ViaDTO.model_validate(via_query)
        return jsonify(response_dto.model_dump()), 200

    except ValidationError as e:
        return jsonify(e.errors()), 400
```
