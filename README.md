# Instalación y Arranque de la Aplicación

Sigue los pasos a continuación para instalar y arrancar la aplicación:

## Requisitos Previos

- **Python**: Asegúrate de tener Python 3.12.3 instalado en tu sistema.
- **Pip**: Verifica que tienes pip 24.0 o una versión superior.

## Pasos de Instalación

1. **Crear un Entorno Virtual**  
    Ejecuta el siguiente comando para crear un entorno virtual:
    ```bash
    python -m venv venv
    ```

2. **Activar el Entorno Virtual**  
    - En **Linux/MacOS**:
      ```bash
      . venv/bin/activate
      ```
    - En **Windows**:
      ```bash
      .\venv\Scripts\activate
      ```

3. **Instalar Dependencias**  
    Instala las dependencias necesarias ejecutando:
    ```bash
    pip install -r requirements.txt
    ```

## Arrancar la Aplicación

Para iniciar la aplicación, utiliza el siguiente comando:
```bash
mkdocs serve
```

Esto iniciará un servidor local donde podrás acceder a la aplicación desde tu navegador.

