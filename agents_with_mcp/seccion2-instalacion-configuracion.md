# Instalación y Configuración

[Volver al índice principal](./mcp-agent-python-esqueleto.md)

En esta sección, aprenderás cómo configurar tu entorno de desarrollo para trabajar con el Protocolo de Contexto de Modelo (MCP) en Python. Cubriremos los requisitos del sistema, la instalación de paquetes necesarios y la configuración inicial.

## Requisitos del Sistema

Para trabajar con MCP en Python, necesitarás:

### Requisitos de Software

1. **Python**: Se requiere Python 3.10 o superior. MCP utiliza características avanzadas de tipado y asincronía que no están disponibles en versiones anteriores.

2. **Sistema Operativo**:
   - **Windows**: Windows 10 o superior
   - **macOS**: macOS 10.15 (Catalina) o superior
   - **Linux**: Cualquier distribución moderna con soporte para Python 3.10+

3. **Gestor de Paquetes**: Se recomienda `uv` (un gestor de paquetes moderno y rápido), aunque también puedes usar `pip`.

### Requisitos de Hardware

No hay requisitos específicos de hardware para desarrollar con MCP, pero para un rendimiento óptimo se recomienda:

- Mínimo 4 GB de RAM (8 GB recomendado)
- Espacio de almacenamiento libre de al menos 1 GB
- Conexión a Internet para descargar paquetes e interactuar con APIs

### Acceso a Modelos de Lenguaje

Para implementar agentes completos con MCP, necesitarás acceso a un modelo de lenguaje. Opciones comunes incluyen:

- **Anthropic Claude**: A través de la API de Anthropic (requiere clave API)
- **OpenAI**: A través de la API de OpenAI (requiere clave API)
- **Otras opciones**: Modelos de código abierto como Llama, Mistral, etc.

## Instalación de Paquetes

### Instalación de uv (Recomendado)

`uv` es un gestor de paquetes rápido y moderno para Python que se recomienda para proyectos MCP:

```bash
# En macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# En Windows (PowerShell)
irm -Headers @{'Accept'='application/vnd.github.v3.raw'} https://astral.sh/uv/install.ps1 | iex
```

Después de la instalación, asegúrate de reiniciar tu terminal para que `uv` esté disponible en el PATH.

### Creación y Configuración del Proyecto

```bash
# Crear un nuevo directorio para tu proyecto
mkdir mi-proyecto-mcp
cd mi-proyecto-mcp

# Inicializar el proyecto con uv
uv init

# Crear y activar un entorno virtual
uv venv
# En macOS/Linux
source .venv/bin/activate
# En Windows
.venv\Scripts\activate

# Instalar las dependencias principales
uv add "mcp[cli]" httpx
```

Si prefieres usar `pip` en lugar de `uv`:

```bash
# Crear un entorno virtual
python -m venv venv
# En macOS/Linux
source venv/bin/activate
# En Windows
venv\Scripts\activate

# Instalar dependencias
pip install "mcp[cli]" httpx
```

### Instalación de Paquetes Adicionales

Dependiendo de tus necesidades, puedes instalar paquetes adicionales:

```bash
# Para crear agentes avanzados
uv add mcp-agent

# Para integración con bases de datos
uv add "mcp-server-sqlite" "mcp-server-postgres"

# Para búsqueda web
uv add "mcp-server-brave-search"

# Para acceso a sistemas de archivos
uv add "mcp-server-filesystem"
```

## Configuración Inicial

### Configuración de Claves API

Si planeas usar modelos de lenguaje a través de APIs, necesitarás configurar tus claves API. Te recomendamos seguir las mejores prácticas de seguridad y no incluir estas claves directamente en tu código.

Crea un archivo `.env` en la raíz de tu proyecto:

```
ANTHROPIC_API_KEY=tu-clave-api-de-anthropic
OPENAI_API_KEY=tu-clave-api-de-openai
```

Instala el paquete python-dotenv para cargar estas variables de entorno:

```bash
uv add python-dotenv
```

Luego, en tu código Python:

```python
import os
from dotenv import load_dotenv

# Cargar variables de entorno
load_dotenv()

# Acceder a las claves API
anthropic_api_key = os.getenv("ANTHROPIC_API_KEY")
openai_api_key = os.getenv("OPENAI_API_KEY")
```

### Configuración para Claude Desktop

Si planeas usar Claude Desktop como host MCP, necesitarás configurarlo para reconocer tus servidores MCP:

1. Asegúrate de tener Claude Desktop instalado y actualizado a la última versión.

2. Encuentra y edita el archivo de configuración:
   - En macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - En Windows: `%APPDATA%\Claude\claude_desktop_config.json`

3. Añade tus servidores MCP a la configuración:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "uv",
      "args": [
        "--directory",
        "/ruta/absoluta/a/tu/proyecto",
        "run",
        "tu_servidor.py"
      ]
    }
  }
}
```

Asegúrate de utilizar rutas absolutas en la configuración.

### Estructura de Proyecto Recomendada

Para proyectos MCP, se recomienda la siguiente estructura de directorios:

```
mi-proyecto-mcp/
├── .env                # Variables de entorno (no incluir en control de versiones)
├── .gitignore          # Configuración de archivos a ignorar
├── mcp_agent.yaml      # Configuración del agente
├── mcp_agent.secrets.yaml  # Secretos del agente (no incluir en control de versiones)
├── servers/            # Directorio para servidores MCP
│   ├── task_server.py  # Ejemplo de servidor de tareas
│   └── data_server.py  # Ejemplo de servidor de datos
├── clients/            # Directorio para clientes MCP
│   └── task_client.py  # Ejemplo de cliente
├── agents/             # Directorio para definiciones de agentes
│   └── finder_agent.py # Ejemplo de agente de búsqueda
├── workflows/          # Directorio para flujos de trabajo
│   ├── parallel_workflow.py # Ejemplo de flujo de trabajo paralelo
│   └── router_workflow.py   # Ejemplo de flujo de trabajo de enrutamiento
└── README.md           # Documentación del proyecto
```

## Verificación de la Instalación

Para verificar que todo está configurado correctamente, puedes crear un servidor MCP de prueba y ejecutarlo:

```python
# test_server.py
from mcp.server.fastmcp import FastMCP

# Crear una instancia de servidor MCP
mcp = FastMCP(name="TestServer")

# Definir una herramienta simple
@mcp.tool()
def echo(message: str) -> str:
    """Devuelve el mensaje recibido."""
    return f"Echo: {message}"

# Ejecutar el servidor
if __name__ == "__main__":
    mcp.run()
```

Ejecuta el servidor:

```bash
python test_server.py
```

Si no hay errores, tu instalación de MCP está funcionando correctamente.

También puedes usar el inspector MCP para probar el servidor:

```bash
mcp dev test_server.py
```

Esto abrirá una interfaz web que te permitirá interactuar con tu servidor MCP para fines de depuración.

## Solución de Problemas Comunes

### Error: "No module named 'mcp'"

Si recibes este error, asegúrate de:
- Haber activado el entorno virtual correctamente
- Haber instalado el paquete MCP con `uv add "mcp[cli]"` o `pip install "mcp[cli]"`

### Error en Claude Desktop: "No se puede encontrar el servidor MCP"

Si Claude Desktop no encuentra tu servidor MCP:
- Verifica que la ruta en `claude_desktop_config.json` sea absoluta y correcta
- Confirma que la ruta al ejecutable `uv` o `python` es correcta
- Reinicia Claude Desktop completamente

### Error: "Python 3.10 or newer is required"

Si recibes este error, necesitas actualizar tu versión de Python:
- Instala Python 3.10 o superior desde [python.org](https://python.org)
- Asegúrate de que la versión correcta esté siendo utilizada en tu entorno

### Problemas con Permisos en Linux/macOS

Si tienes problemas de permisos al ejecutar scripts:
- Asegúrate de que los archivos tienen permisos de ejecución: `chmod +x script.py`
- Verifica que tienes permisos en las carpetas relevantes

### Error: "Cannot connect to MCP server"

Si el cliente no puede conectarse al servidor:
- Verifica que el servidor esté ejecutándose
- Asegúrate de que estás utilizando el protocolo de transporte correcto (`stdio` o `sse`)
- Comprueba si hay cortafuegos o problemas de red que puedan estar bloqueando la conexión
