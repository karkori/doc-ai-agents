# Patrones y Flujos de Trabajo Avanzados

[Volver al índice principal](./mcp-agent-python-esqueleto.md)

En esta sección, exploraremos patrones avanzados para la construcción de agentes MCP que puedan abordar tareas complejas y flujos de trabajo sofisticados. Estos patrones representan estructuras arquitectónicas probadas que pueden combinarse y adaptarse para crear sistemas de agentes altamente capaces y especializados.

## Introducción a los Patrones de Agentes

Los patrones de agentes son estructuras reutilizables que definen cómo los agentes interactúan entre sí y con los servidores MCP para resolver problemas complejos. Estos patrones permiten:

- **Modularidad**: Dividir problemas complejos en componentes más manejables
- **Especialización**: Crear agentes expertos en tareas específicas
- **Orquestación**: Coordinar la colaboración entre múltiples agentes
- **Escalabilidad**: Manejar tareas de mayor complejidad de manera estructurada

Implementar estos patrones con MCP-Agent aprovecha la capacidad del framework para gestionar conexiones MCP y componer flujos de trabajo de manera elegante.

## Patrón Parallel

El patrón Parallel permite ejecutar múltiples agentes simultáneamente para procesar diferentes aspectos de un problema y luego combinar sus resultados.

### Estructura del Patrón

```
                  +--------------+
                  |  Controlador |
                  +------+-------+
                         |
         +---------------+---------------+
         |               |               |
+--------v-----+ +-------v------+ +------v-------+
| Agente A     | | Agente B     | | Agente C     |
| (Tarea 1)    | | (Tarea 2)    | | (Tarea 3)    |
+--------------+ +--------------+ +--------------+
         |               |               |
         +---------------+---------------+
                         |
                  +------v-------+
                  | Integrador   |
                  +--------------+
```

### Implementación Básica

```python
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client
import asyncio
from typing import List, Dict, Any

# Configuración básica
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(app.get_mcp_config())

# Definir agentes especializados
agente_datos = Agent(
    name="AnalizadorDatos",
    instruction="Extrae y analiza datos numéricos relevantes de la consulta.",
    mcp_servers=["brave"]
)

agente_contexto = Agent(
    name="AnalizadorContexto",
    instruction="Proporciona contexto histórico y explicativo sobre el tema.",
    mcp_servers=["brave"]
)

agente_tendencias = Agent(
    name="AnalizadorTendencias",
    instruction="Identifica tendencias recientes y predicciones futuras sobre el tema.",
    mcp_servers=["brave"]
)

agente_integrador = Agent(
    name="Integrador",
    instruction="""
    Integra diferentes análisis en un informe cohesivo y estructurado.
    Sintetiza la información proporcionada por distintos agentes especializados.
    Resuelve contradicciones y proporciona una perspectiva unificada.
    """,
    mcp_servers=[]  # No necesita servidores MCP
)

async def ejecutar_agente(agente: Agent, consulta: str, connection_map) -> str:
    """Ejecuta un agente específico con la consulta dada."""
    cliente = gen_client(agente, connection_map, app.get_llm_config())
    return await cliente.generate_str(consulta)

async def parallel_processing(consulta: str) -> Dict[str, Any]:
    """Implementa el patrón Parallel para procesar una consulta con múltiples agentes."""
    print(f"Procesando en paralelo: {consulta}")
    
    async with connection_manager.connect() as connection_map:
        # Ejecutar agentes especializados en paralelo
        tareas = [
            ejecutar_agente(agente_datos, f"Analiza datos sobre: {consulta}", connection_map),
            ejecutar_agente(agente_contexto, f"Proporciona contexto sobre: {consulta}", connection_map),
            ejecutar_agente(agente_tendencias, f"Analiza tendencias sobre: {consulta}", connection_map)
        ]
        
        resultados = await asyncio.gather(*tareas)
        
        # Preparar los resultados para integración
        analisis_datos = resultados[0]
        analisis_contexto = resultados[1]
        analisis_tendencias = resultados[2]
        
        # Integrar resultados
        prompt_integrador = f"""
        Integra los siguientes análisis sobre "{consulta}" en un informe cohesivo:
        
        === ANÁLISIS DE DATOS ===
        {analisis_datos}
        
        === ANÁLISIS DE CONTEXTO ===
        {analisis_contexto}
        
        === ANÁLISIS DE TENDENCIAS ===
        {analisis_tendencias}
        
        Proporciona un informe estructurado que sintetice toda esta información.
        """
        
        informe_integrado = await ejecutar_agente(agente_integrador, prompt_integrador, connection_map)
        
        return {
            "informe_completo": informe_integrado,
            "componentes": {
                "datos": analisis_datos,
                "contexto": analisis_contexto,
                "tendencias": analisis_tendencias
            }
        }
```

### Caso de Uso: Análisis de Mercado Multidimensional

```python
async def analisis_mercado(sector: str) -> Dict[str, Any]:
    """Realiza un análisis de mercado completo de un sector determinado."""
    consulta = f"Análisis completo del mercado del sector {sector}"
    
    # Aplicar el patrón Parallel
    resultado = await parallel_processing(consulta)
    
    # Formatear el resultado para presentación
    print("\n" + "=" * 80)
    print(f"ANÁLISIS DE MERCADO: {sector.upper()}")
    print("=" * 80)
    print(resultado["informe_completo"])
    print("=" * 80)
    
    return resultado

# Uso:
# await analisis_mercado("tecnología blockchain")
```

## Patrón Router

El patrón Router dirige una consulta al agente más adecuado según el tipo de pregunta o tarea, permitiendo la especialización y optimización de recursos.

### Estructura del Patrón

```
                  +-------------+
                  |             |
                  |   Router    |
                  |             |
                  +------+------+
                         |
     +------------------+-+------------------+
     |                    |                  |
+----v-----+       +-----v----+      +------v---+
| Agente A |       | Agente B |      | Agente C |
| (Tipo A) |       | (Tipo B) |      | (Tipo C) |
+----------+       +----------+      +----------+
```

### Implementación Básica

```python
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client
import asyncio
from typing import Dict, Any
from enum import Enum

# Configuración básica
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(app.get_mcp_config())

# Definir tipos de consultas
class TipoConsulta(Enum):
    TECNICA = "técnica"
    FINANCIERA = "financiera"
    LEGAL = "legal"
    GENERAL = "general"

# Definir agentes especializados
agente_tecnico = Agent(
    name="ExpertoTecnico",
    instruction="Eres un experto técnico que responde preguntas sobre tecnología, programación y ciencia.",
    mcp_servers=["brave", "github"]
)

agente_financiero = Agent(
    name="ExpertoFinanciero",
    instruction="Eres un experto financiero que responde preguntas sobre inversiones, mercados y economía.",
    mcp_servers=["brave", "financial-data"]
)

agente_legal = Agent(
    name="ExpertoLegal",
    instruction="Eres un experto legal que responde preguntas sobre leyes, regulaciones y cumplimiento.",
    mcp_servers=["brave", "legal-db"]
)

agente_general = Agent(
    name="AsistenteGeneral",
    instruction="Eres un asistente general que responde preguntas diversas de manera clara y concisa.",
    mcp_servers=["brave"]
)

# Agente router para clasificar consultas
agente_router = Agent(
    name="Router",
    instruction="""
    Tu única tarea es determinar a qué categoría pertenece una consulta.
    Clasifica cada consulta en UNA SOLA de estas categorías:
    
    - TECNICA: Preguntas sobre tecnología, programación, ciencia, ingeniería, etc.
    - FINANCIERA: Preguntas sobre finanzas, inversiones, economía, mercados, etc.
    - LEGAL: Preguntas sobre leyes, regulaciones, cumplimiento, contratos, etc.
    - GENERAL: Cualquier otra pregunta que no encaje claramente en las categorías anteriores.
    
    Responde ÚNICAMENTE con el nombre de la categoría en mayúsculas (por ejemplo, "TECNICA").
    """,
    mcp_servers=[]  # No necesita servidores MCP
)

async def clasificar_consulta(consulta: str, connection_map) -> TipoConsulta:
    """Clasifica la consulta utilizando el agente router."""
    cliente = gen_client(agente_router, connection_map, app.get_llm_config())
    resultado = await cliente.generate_str(consulta)
    
    # Limpiar el resultado y convertirlo al enum
    resultado = resultado.strip().upper()
    
    try:
        return TipoConsulta(resultado.lower())
    except ValueError:
        # Si no se puede convertir, usar el tipo general como fallback
        return TipoConsulta.GENERAL

async def router_processing(consulta: str) -> Dict[str, Any]:
    """Implementa el patrón Router para dirigir la consulta al agente apropiado."""
    async with connection_manager.connect() as connection_map:
        # Clasificar la consulta
        tipo = await clasificar_consulta(consulta, connection_map)
        
        # Seleccionar el agente adecuado según el tipo
        if tipo == TipoConsulta.TECNICA:
            agente = agente_tecnico
            nombre_categoria = "técnica"
        elif tipo == TipoConsulta.FINANCIERA:
            agente = agente_financiero
            nombre_categoria = "financiera"
        elif tipo == TipoConsulta.LEGAL:
            agente = agente_legal
            nombre_categoria = "legal"
        else:  # TipoConsulta.GENERAL
            agente = agente_general
            nombre_categoria = "general"
        
        print(f"Consulta clasificada como: {nombre_categoria}")
        
        # Procesar la consulta con el agente seleccionado
        cliente = gen_client(agente, connection_map, app.get_llm_config())
        respuesta = await cliente.generate_str(consulta)
        
        return {
            "tipo": nombre_categoria,
            "consulta": consulta,
            "respuesta": respuesta
        }
```

### Caso de Uso: Sistema de Soporte Inteligente

```python
async def sistema_soporte(consulta: str) -> Dict[str, Any]:
    """Sistema de soporte que dirige preguntas a expertos especializados."""
    print(f"Procesando consulta: {consulta}")
    
    # Aplicar el patrón Router
    resultado = await router_processing(consulta)
    
    # Formatear respuesta
    print("\n" + "-" * 50)
    print(f"Consulta: {consulta}")
    print(f"Categoría: {resultado['tipo']}")
    print("-" * 50)
    print(resultado["respuesta"])
    print("-" * 50)
    
    return resultado

# Uso:
# await sistema_soporte("¿Cómo implementar un servidor MCP en Python?")  # Técnica
# await sistema_soporte("¿Cuáles son los mejores ETFs para invertir en 2025?")  # Financiera
```

[Seguir con la parte 2](./seccion6-patrones-avanzados-parte2.md)