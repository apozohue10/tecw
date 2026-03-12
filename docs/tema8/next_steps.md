# Resumen de lo que hemos visto

1. **Del HTML al JSON (Cambio de Paradigma):** Hemos pasado de construir aplicaciones monolíticas que devuelven interfaces visuales (HTML) para humanos, a diseñar APIs RESTful que sirven datos crudos (JSON) legibles por máquina, abriendo la puerta a múltiples clientes y arquitecturas de microservicios.
2. **REST y la Ausencia de Estado (_Stateless_):** Hemos mapeado las operaciones de base de datos (CRUD) a los verbos HTTP estándar (GET, POST, PUT, DELETE) y entendido que en una API el servidor no guarda "sesiones"; cada petición debe contener toda la información necesaria para ser procesada.
3. **El reinado de los DTOs:** Hemos aprendido que no debemos exponer nuestros modelos de base de datos directamente. Los _Data Transfer Objects_ (DTOs) nos permiten definir estrictamente qué datos entran y salen, ocultar información sensible y estructurar nuestras respuestas.
4. **Pydantic como escudo validador:** Hemos sustituido el tedioso manejo de diccionarios manuales por Pydantic, utilizando los _type hints_ de Python para validar datos automáticamente, forzar la coerción de tipos y generar respuestas de error estructuradas antes de tocar la base de datos.
5. **Documentación interactiva con OpenAPI:** Hemos descubierto que una API sin documentar es inútil. Escribiendo la especificación en YAML e integrando Swagger UI mediante Flasgger, hemos logrado que nuestra propia aplicación ofrezca un portal interactivo para que cualquier desarrollador sepa cómo consumirla.

## Next Steps para API profesional

Ahora que tenemos una API funcional, validada y documentada, el ecosistema web nos plantea nuevos retos que abordaremos más adelante. ¿Qué nos falta por aprender?

- **Autenticación _Stateless_ (JWT):** Si ya no tenemos el objeto `session` de Flask ni usamos _cookies_ tradicionales... ¿Cómo sabemos quién es el usuario que hace la petición? Aprenderemos a usar _JSON Web Tokens_ (JWT) y a pasarlos por las cabeceras HTTP (`Authorization: Bearer <token>`).
- **Paginación y Filtrado:** En ingeniería de datos no podemos devolver 100.000 registros en un solo JSON porque tumbaríamos el servidor y el cliente. Aprenderemos a recibir parámetros por la URL (`?page=2&limit=50&dificultad=6a`) para devolver datos fragmentados.
- **Arquitectura de Carpetas y Clean Code:** A medida que la API crece, tener todo en un `app.py` o en un solo controlador es insostenible. Veremos cómo estructurar un proyecto profesional (Controladores, Capa de Servicios, Capa de Acceso a Datos/Repositorios).
- **Testing Automatizado:** Postman está genial para probar a mano, pero los profesionales escriben código que prueba su código. Aprenderemos a usar `pytest` para lanzar miles de peticiones automatizadas a nuestra API y garantizar que no hemos roto nada al añadir una nueva funcionalidad.
- **Despliegue (Dockerización):** Cómo empaquetar nuestra API y nuestra base de datos en contenedores Docker para subirla a la nube y que esté disponible para el mundo real.
