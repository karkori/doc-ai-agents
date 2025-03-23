# Depuración y Pruebas

[Volver al índice principal](./mcp-agent-python-esqueleto.md)

Desarrollar agentes con el Protocolo de Contexto de Modelo (MCP) requiere técnicas específicas de depuración y pruebas para garantizar su correcto funcionamiento. En esta sección, exploramos herramientas, metodologías y mejores prácticas para identificar y solucionar problemas, así como para validar el comportamiento de tus agentes MCP.

## Herramientas de Depuración

### MCP Inspector

El Inspector MCP es una herramienta oficial que permite interactuar directamente con servidores MCP y examinar su funcionamiento. Es esencial para el desarrollo y la depuración de servidores MCP.

#### Instalación y Configuración

```bash
# Instalar CLI de MCP con componentes de desarrollo
uv add "mcp[cli,dev]"

# O con pip
pip install "mcp[cli,dev]"
```

#### Uso Básico

```bash
# Iniciar el Inspector con un servidor MCP
mcp dev path/to/your/server.py

# Con dependencias adicionales
mcp dev server.py --with pandas --with httpx
```

El Inspector MCP proporciona una interfaz web con las siguientes capacidades:

1. **Exploración de Capacidades**: Listar todos los recursos, herramientas y prompts disponibles.
2. **Prueba de Recursos**: Probar la lectura de recursos con diferentes parámetros.
3. **Prueba de Herramientas**: Ejecutar herramientas directamente y ver los resultados.
4. **Examen de Errores**: Ver errores detallados y trazas de la ejecución.
5. **Historial de Interacciones**: Revisar interacciones previas para análisis.

### Registro (Logging)

Implementar un sistema de registro detallado es crucial para depurar agentes MCP, especialmente en sistemas complejos con múltiples agentes.

#### Configuración Básica de Logging

```python
import logging
import os
from datetime import datetime

def setup_logging(name="mcp_agent", level=logging.DEBUG):
    """Configura un sistema de logging para los agentes MCP."""
    
    # Crear directorio para logs si no existe
    log_dir = "logs"
    os.makedirs(log_dir, exist_ok=True)
    
    # Configurar archivo de log con timestamp
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    log_file = f"{log_dir}/{name}_{timestamp}.log"
    
    # Configurar logger
    logger = logging.getLogger(name)
    logger.setLevel(level)
    
    # Handler para archivo
    file_handler = logging.FileHandler(log_file)
    file_handler.setLevel(level)
    
    # Handler para consola
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)  # Menos detallado en consola
    
    # Formato de mensaje
    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    file_handler.setFormatter(formatter)
    console_handler.setFormatter(formatter)
    
    # Añadir handlers
    logger.addHandler(file_handler)
    logger.addHandler(console_handler)
    
    return logger

# Uso
logger = setup_logging()
```

#### Logging Avanzado para Agentes MCP

```python
class MCPAgentLogger:
    """Clase para loggin especializado en agentes MCP."""
    
    def __init__(self, agent_name, log_level=logging.DEBUG):
        self.logger = setup_logging(f"agent_{agent_name}", log_level)
        self.agent_name = agent_name
    
    def log_prompt(self, prompt, truncate=500):
        """Registra un prompt enviado al LLM."""
        truncated = prompt if len(prompt) <= truncate else prompt[:truncate] + "..."
        self.logger.debug(f"PROMPT TO {self.agent_name}:\n{truncated}")
    
    def log_completion(self, completion, truncate=500):
        """Registra una respuesta recibida del LLM."""
        truncated = completion if len(completion) <= truncate else completion[:truncate] + "..."
        self.logger.debug(f"COMPLETION FROM {self.agent_name}:\n{truncated}")
    
    def log_tool_call(self, tool_name, params):
        """Registra una llamada a herramienta."""
        self.logger.info(f"TOOL CALL: {tool_name} - Params: {params}")
    
    def log_tool_result(self, tool_name, result, truncate=500):
        """Registra el resultado de una herramienta."""
        result_str = str(result)
        truncated = result_str if len(result_str) <= truncate else result_str[:truncate] + "..."
        self.logger.info(f"TOOL RESULT: {tool_name} - Result: {truncated}")
    
    def log_resource_access(self, resource_uri):
        """Registra acceso a un recurso."""
        self.logger.info(f"RESOURCE ACCESS: {resource_uri}")
    
    def log_error(self, message, error=None):
        """Registra un error."""
        if error:
            self.logger.error(f"ERROR: {message} - {str(error)}", exc_info=True)
        else:
            self.logger.error(f"ERROR: {message}")
    
    def log_warn(self, message):
        """Registra una advertencia."""
        self.logger.warning(f"WARNING: {message}")
    
    def log_info(self, message):
        """Registra información general."""
        self.logger.info(message)
```

### Instrumentación con MCP-Agent

El framework MCP-Agent proporciona capacidades de instrumentación que puedes aprovechar para depuración:

```python
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client
import asyncio

# Configuración con logging activado
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(app.get_mcp_config())

# Definir agente con instrumentación
agente = Agent(
    name="AgenteDePrueba",
    instruction="Instrucción para el agente...",
    mcp_servers=["brave"],
    # Habilitar instrumentación detallada
    instrumentation={
        "log_prompts": True,           # Registrar prompts enviados al LLM
        "log_completions": True,       # Registrar respuestas recibidas
        "log_tool_calls": True,        # Registrar llamadas a herramientas
        "log_resource_access": True,   # Registrar acceso a recursos
        "log_to_file": "logs/agent_instrumentation.log"  # Archivo de log
    }
)

async def ejecutar_agente(consulta):
    async with connection_manager.connect() as connection_map:
        cliente = gen_client(agente, connection_map, app.get_llm_config())
        
        # Esta llamada generará logs detallados
        respuesta = await cliente.generate_str(consulta)
        return respuesta
```

### Claude Desktop CLI

Si utilizas Claude Desktop como host MCP, puedes utilizar su CLI para depuración:

```bash
# Ver logs de MCP
tail -f ~/Library/Logs/Claude/mcp*.log

# Verificar errores específicos de un servidor
cat ~/Library/Logs/Claude/mcp-server-miServidor.log
```

En Windows, los logs se encuentran en:
```
%APPDATA%\Claude\Logs\
```

## Gestión de Errores

Un manejo efectivo de errores es crucial para construir agentes MCP robustos y confiables.

### Tipos Comunes de Errores en MCP

1. **Errores de Conexión**: Fallos al conectar con servidores MCP
2. **Errores de Protocolo**: Mensajes incorrectos o malformados
3. **Errores de Ejecución**: Fallos durante la ejecución de herramientas
4. **Errores del LLM**: Problemas con el modelo de lenguaje
5. **Errores de Recursos**: Fallos al acceder a recursos

### Manejo de Errores con Try/Except

```python
async def llamar_herramienta_segura(cliente, herramienta, params):
    """Llama a una herramienta con manejo de errores robusto."""
    try:
        resultado = await cliente.call_tool(herramienta, {"params": params})
        return {
            "exito": True,
            "resultado": resultado
        }
    except ConnectionError as e:
        logger.error(f"Error de conexión al llamar a {herramienta}: {str(e)}")
        return {
            "exito": False,
            "error": "Error de conexión",
            "detalle": str(e)
        }
    except TimeoutError as e:
        logger.error(f"Timeout al llamar a {herramienta}: {str(e)}")
        return {
            "exito": False,
            "error": "Timeout",
            "detalle": str(e)
        }
    except Exception as e:
        logger.error(f"Error inesperado al llamar a {herramienta}: {str(e)}")
        return {
            "exito": False,
            "error": "Error inesperado",
            "detalle": str(e)
        }
```

### Retries con Backoff Exponencial

```python
import random
import asyncio

async def llamada_con_retry(funcion, *args, max_intentos=3, backoff_inicial=1, **kwargs):
    """Ejecuta una función con retry y backoff exponencial."""
    intento = 0
    ultima_excepcion = None
    
    while intento < max_intentos:
        try:
            return await funcion(*args, **kwargs)
        except Exception as e:
            intento += 1
            ultima_excepcion = e
            
            if intento >= max_intentos:
                logger.error(f"Agotados todos los intentos ({max_intentos}): {str(e)}")
                break
            
            # Calcular tiempo de espera con jitter
            tiempo_espera = backoff_inicial * (2 ** (intento - 1)) * (0.5 + random.random())
            logger.warning(f"Intento {intento} fallido. Reintentando en {tiempo_espera:.2f}s: {str(e)}")
            
            await asyncio.sleep(tiempo_espera)
    
    # Si llegamos aquí, todos los intentos fallaron
    raise ultima_excepcion
```
