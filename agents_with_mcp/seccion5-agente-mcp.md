# Creando un Agente de IA con MCP-Agent

[Volver al índice principal](./mcp-agent-python-esqueleto.md)

En esta sección, aprenderemos a crear agentes de IA utilizando el framework `mcp-agent`, que proporciona una manera estructurada y componible de construir agentes que aprovechan las capacidades del Protocolo de Contexto de Modelo (MCP).

## Introducción a MCP-Agent

MCP-Agent es un framework desarrollado para simplificar la creación de agentes de IA que utilizan el Protocolo de Contexto de Modelo. A diferencia de las implementaciones básicas de clientes MCP que vimos en la sección anterior, MCP-Agent proporciona abstracciones de alto nivel para definir agentes, gestionar conexiones MCP y componer flujos de trabajo complejos.

### Conceptos Clave

- **MCPApp**: Representa la aplicación global y mantiene la configuración.
- **Agent**: Define un agente con un nombre, instrucciones y acceso a servidores MCP.
- **AugmentedLLM**: Un modelo de lenguaje mejorado con herramientas proporcionadas por servidores MCP.
- **MCPConnectionManager**: Gestiona conexiones a múltiples servidores MCP.

### ¿Por qué usar MCP-Agent?

- **Simplicidad**: Abstrae la complejidad de gestionar conexiones MCP y ciclos de vida.
- **Componibilidad**: Permite crear flujos de trabajo complejos combinando patrones simples.
- **Reutilización**: Proporciona implementaciones de patrones comunes para agentes de IA.
- **Escalabilidad**: Facilita la creación de sistemas multi-agente que colaboran entre sí.

## Configuración de MCP-Agent

### Instalación

Primero, instala el paquete MCP-Agent:

```bash
# Con uv
uv add mcp-agent

# Con pip
pip install mcp-agent
```

### Configuración Básica

MCP-Agent utiliza archivos YAML para la configuración. Crea un archivo `mcp_agent.yaml` en la raíz de tu proyecto:

```yaml
name: mi-agente-mcp
description: Un agente de IA que utiliza servidores MCP

llm:
  provider: anthropic  # Opciones: anthropic, openai, etc.
  model: claude-3-opus-20240229  # Modelo específico a utilizar

mcp_servers:
  brave:
    command: uvx
    args: [mcp-server-brave-search]
  filesystem:
    command: uvx
    args: [mcp-server-filesystem]
```

Para las claves API y otros secretos, crea un archivo separado `mcp_agent.secrets.yaml`:

```yaml
anthropic:
  api_key: "tu-clave-api-de-anthropic"

openai:
  api_key: "tu-clave-api-de-openai"
```

> **Nota**: Asegúrate de añadir `mcp_agent.secrets.yaml` a tu `.gitignore` para evitar exponer tus claves.

### Inicialización de la Aplicación

```python
from mcp_agent import MCPApp, MCPConnectionManager

# Cargar configuración desde archivos YAML
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")

# Crear gestor de conexiones MCP
connection_manager = MCPConnectionManager(app.get_mcp_config())
```

## Definiendo Agentes

Los agentes en MCP-Agent son entidades con un propósito específico que tienen acceso a un conjunto de servidores MCP.

### Estructura Básica de un Agente

```python
from mcp_agent import Agent

# Definir un agente básico
asistente_agente = Agent(
    name="AsistentePersonal",
    instruction="""
    Eres un asistente personal inteligente. Tu objetivo es ayudar al usuario
    con sus preguntas y tareas diarias. Puedes buscar información en línea,
    gestionar archivos y proporcionar respuestas útiles y concisas.
    """,
    mcp_servers=["brave", "filesystem"]  # Nombres de los servidores a utilizar
)
```

### Agente con Instrucciones Detalladas

```python
investigador_agente = Agent(
    name="InvestigadorAcadémico",
    instruction="""
    Eres un investigador académico especializado en buscar y sintetizar
    información científica. Tu objetivo es proporcionar respuestas basadas
    en evidencia, citando fuentes relevantes y actualizadas.

    Cuando el usuario haga una pregunta:
    1. Utiliza la herramienta de búsqueda para encontrar información relevante.
    2. Evalúa críticamente las fuentes y selecciona las más confiables.
    3. Sintetiza la información en una respuesta cohesiva.
    4. Proporciona citas adecuadas para todas las afirmaciones importantes.
    5. Identifica áreas de incertidumbre o controversia cuando existan.

    Mantén un tono académico pero accesible, evitando jerga innecesaria.
    """,
    mcp_servers=["brave"]
)
```

### Agente para Análisis de Datos

```python
analista_datos_agente = Agent(
    name="AnalistaDatos",
    instruction="""
    Eres un analista de datos experto. Tu objetivo es ayudar al usuario
    a comprender, manipular y visualizar datos.
    
    Puedes:
    - Leer y analizar archivos CSV, Excel y otros formatos de datos
    - Realizar análisis estadísticos básicos y avanzados
    - Generar visualizaciones informativas
    - Interpretar tendencias y patrones en los datos
    
    Utiliza un enfoque metodológico para el análisis, explicando
    claramente tus pasos y conclusiones.
    """,
    mcp_servers=["filesystem", "sqlite", "python-repl"]
)
```

## Construyendo un Cliente para el Agente

Una vez definido el agente, necesitamos crear un cliente que pueda utilizarlo para procesar consultas del usuario.

### Cliente Básico

```python
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client
import asyncio

# Configuración básica
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(app.get_mcp_config())

# Definir agente
asistente_agente = Agent(
    name="AsistentePersonal",
    instruction="Eres un asistente personal inteligente que ayuda con tareas cotidianas.",
    mcp_servers=["brave", "filesystem"]
)

async def procesar_consulta(consulta: str):
    """Procesa una consulta del usuario utilizando el agente."""
    # Establecer conexiones MCP
    async with connection_manager.connect() as connection_map:
        # Generar cliente para el agente
        cliente = gen_client(asistente_agente, connection_map, app.get_llm_config())
        
        # Procesar la consulta y obtener la respuesta
        respuesta = await cliente.generate_str(consulta)
        return respuesta

# Función principal
async def main():
    # Ejemplo de consulta
    consulta = "¿Puedes buscar información sobre el cambio climático y resumir los principales hallazgos?"
    
    # Procesar la consulta
    respuesta = await procesar_consulta(consulta)
    
    # Mostrar resultado
    print("\n=== RESPUESTA DEL AGENTE ===\n")
    print(respuesta)

# Ejecutar la aplicación
if __name__ == "__main__":
    asyncio.run(main())
```

### Cliente Interactivo

Para una experiencia más interactiva, podemos crear un cliente que permita múltiples interacciones:

```python
async def chat_interactivo():
    """Proporciona una interfaz de chat interactiva con el agente."""
    print("=== CHAT CON ASISTENTE IA ===")
    print("Escribe 'salir' para terminar la conversación.")
    
    # Historial de mensajes para mantener contexto
    historial = []
    
    async with connection_manager.connect() as connection_map:
        # Generar cliente para el agente
        cliente = gen_client(asistente_agente, connection_map, app.get_llm_config())
        
        while True:
            # Obtener entrada del usuario
            consulta = input("\nTú: ")
            
            if consulta.lower() in ["salir", "exit", "quit"]:
                print("¡Hasta luego!")
                break
            
            # Añadir mensaje al historial
            historial.append({"role": "user", "content": consulta})
            
            # Procesar la consulta con el historial completo
            respuesta = await cliente.generate_str(historial)
            
            # Mostrar respuesta
            print(f"\nAsistente: {respuesta}")
            
            # Añadir respuesta al historial
            historial.append({"role": "assistant", "content": respuesta})
```

## Procesamiento Estructurado

MCP-Agent también permite obtener respuestas estructuradas utilizando modelos Pydantic:

```python
from pydantic import BaseModel, Field
from typing import List

# Definir un modelo para respuestas estructuradas
class Artículo(BaseModel):
    título: str = Field(description="Título del artículo")
    autor: str = Field(description="Autor del artículo")
    año: int = Field(description="Año de publicación")
    resumen: str = Field(description="Breve resumen del contenido")

class RespuestaBúsqueda(BaseModel):
    query: str = Field(description="Consulta de búsqueda original")
    artículos: List[Artículo] = Field(description="Lista de artículos encontrados")
    análisis: str = Field(description="Análisis global de los resultados")

# Función para obtener respuesta estructurada
async def búsqueda_estructurada(consulta: str):
    async with connection_manager.connect() as connection_map:
        cliente = gen_client(investigador_agente, connection_map, app.get_llm_config())
        
        # Generar respuesta estructurada
        resultado = await cliente.generate_structured(
            f"Busca artículos científicos sobre: {consulta}",
            RespuestaBúsqueda
        )
        
        return resultado
```

Este enfoque es particularmente útil cuando necesitas procesar la respuesta del agente de manera programática.

## Ejemplo Básico de Agente

Pongamos todo junto en un ejemplo completo de un agente investigador:

```python
# investigador_agente.py
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client
import asyncio
import argparse

# Configuración básica
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(app.get_mcp_config())

# Definir agente
investigador_agente = Agent(
    name="InvestigadorAcadémico",
    instruction="""
    Eres un investigador académico especializado en buscar y sintetizar
    información científica. Tu objetivo es proporcionar respuestas basadas
    en evidencia, citando fuentes relevantes y actualizadas.

    Cuando el usuario haga una pregunta:
    1. Utiliza la herramienta de búsqueda para encontrar información relevante.
    2. Evalúa críticamente las fuentes y selecciona las más confiables.
    3. Sintetiza la información en una respuesta cohesiva.
    4. Proporciona citas adecuadas para todas las afirmaciones importantes.
    5. Identifica áreas de incertidumbre o controversia cuando existan.

    Estructura tu respuesta de la siguiente manera:
    - Resumen ejecutivo (2-3 oraciones)
    - Hallazgos principales (con citas)
    - Consideraciones y limitaciones
    - Conclusión
    - Referencias

    Mantén un tono académico pero accesible, evitando jerga innecesaria.
    """,
    mcp_servers=["brave"]
)

async def realizar_investigación(tema: str):
    """Realiza una investigación sobre un tema específico."""
    print(f"Investigando sobre: {tema}")
    print("Por favor, espera mientras se realiza la búsqueda...")
    
    async with connection_manager.connect() as connection_map:
        cliente = gen_client(investigador_agente, connection_map, app.get_llm_config())
        
        # Formular la consulta de investigación
        consulta = f"""
        Realiza una investigación académica completa sobre el tema: {tema}
        
        Proporciona un análisis exhaustivo con fuentes confiables y actualizadas.
        Asegúrate de cubrir los diferentes aspectos y perspectivas sobre el tema.
        """
        
        # Obtener respuesta
        respuesta = await cliente.generate_str(consulta)
        
        # Formatear y mostrar el resultado
        print("\n" + "=" * 80)
        print(f"INFORME DE INVESTIGACIÓN: {tema.upper()}")
        print("=" * 80)
        print(respuesta)
        print("=" * 80)
        
        return respuesta

async def guardar_resultado(tema: str, contenido: str):
    """Guarda el resultado de la investigación en un archivo."""
    # Crear un nombre de archivo basado en el tema
    nombre_archivo = tema.lower().replace(" ", "_") + "_investigacion.txt"
    
    # Guardar el contenido
    with open(nombre_archivo, "w", encoding="utf-8") as f:
        f.write(f"INFORME DE INVESTIGACIÓN: {tema.upper()}\n")
        f.write("=" * 80 + "\n")
        f.write(contenido)
        f.write("\n" + "=" * 80)
    
    print(f"\nEl informe ha sido guardado en el archivo: {nombre_archivo}")

async def main():
    # Configurar el parser de argumentos
    parser = argparse.ArgumentParser(description="Agente Investigador Académico")
    parser.add_argument("tema", nargs="?", default=None, help="Tema a investigar")
    args = parser.parse_args()
    
    if args.tema:
        # Modo de línea de comandos
        tema = args.tema
    else:
        # Modo interactivo
        print("=== AGENTE INVESTIGADOR ACADÉMICO ===")
        tema = input("Ingresa el tema a investigar: ")
    
    # Realizar la investigación
    resultado = await realizar_investigación(tema)
    
    # Preguntar si desea guardar el resultado
    guardar = input("\n¿Deseas guardar este informe? (s/n): ")
    if guardar.lower() in ["s", "si", "sí", "y", "yes"]:
        await guardar_resultado(tema, resultado)

if __name__ == "__main__":
    asyncio.run(main())
```

Este script crea un agente investigador que puede:
1. Buscar información sobre un tema específico
2. Sintetizar los hallazgos en un informe estructurado
3. Guardar el resultado en un archivo para referencia futura

Puedes ejecutarlo de dos formas:
- Interactivamente: `python investigador_agente.py`
- Con un tema específico: `python investigador_agente.py "impacto del cambio climático en la agricultura"`

## Agente Personalizado para Casos de Uso Específicos

Veamos un ejemplo más especializado: un agente para análisis de archivos PDF:

```python
# analizador_pdf.py
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client
import asyncio
import os
from typing import List
from pydantic import BaseModel, Field

# Modelos para respuestas estructuradas
class PuntosClave(BaseModel):
    título: str = Field(description="Título o tema del punto clave")
    descripción: str = Field(description="Explicación detallada del punto")
    páginas: List[int] = Field(description="Números de página donde aparece este punto")

class AnálisisPDF(BaseModel):
    título_documento: str = Field(description="Título del documento PDF")
    autor: str = Field(description="Autor del documento, si está disponible")
    resumen_ejecutivo: str = Field(description="Resumen conciso del contenido completo")
    puntos_clave: List[PuntosClave] = Field(description="Puntos clave identificados")
    conclusión: str = Field(description="Conclusión general del análisis")

# Configuración básica
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(app.get_mcp_config())

# Definir agente especializado
analizador_pdf_agente = Agent(
    name="AnalizadorPDF",
    instruction="""
    Eres un asistente especializado en analizar documentos PDF académicos y técnicos.
    Tu objetivo es extraer la información más relevante y presentarla de manera
    estructurada y comprensible.

    Para cada documento:
    1. Identifica el título, autor(es) y fecha de publicación.
    2. Determina el tipo de documento (artículo de investigación, informe técnico, libro, etc.).
    3. Extrae los puntos clave y hallazgos principales.
    4. Identifica la metodología utilizada, si aplica.
    5. Resume las conclusiones y recomendaciones.
    6. Destaca términos técnicos importantes y su significado.

    Estructura tu análisis de manera clara y concisa, utilizando listas y secciones
    cuando sea apropiado.
    """,
    mcp_servers=["filesystem", "pdf-reader"]  # Necesitarías un servidor MCP para lectura de PDFs
)

async def analizar_pdf(ruta_archivo: str):
    """Analiza un archivo PDF y proporciona un resumen estructurado."""
    if not os.path.exists(ruta_archivo):
        return {"error": f"El archivo {ruta_archivo} no existe."}
    
    if not ruta_archivo.lower().endswith(".pdf"):
        return {"error": f"El archivo {ruta_archivo} no es un PDF."}
    
    print(f"Analizando PDF: {ruta_archivo}")
    print("Este proceso puede tomar varios minutos dependiendo del tamaño del documento...")
    
    async with connection_manager.connect() as connection_map:
        cliente = gen_client(analizador_pdf_agente, connection_map, app.get_llm_config())
        
        # Formular la consulta
        consulta = f"""
        Analiza el siguiente documento PDF ubicado en: {ruta_archivo}
        
        Proporciona un análisis exhaustivo que incluya:
        - Título y autor
        - Resumen ejecutivo
        - Puntos clave con números de página
        - Conclusiones principales
        
        Estructura la información de manera clara y organizada.
        """
        
        try:
            # Generar respuesta estructurada
            resultado = await cliente.generate_structured(consulta, AnálisisPDF)
            return resultado
        except Exception as e:
            return {"error": f"Error al analizar el PDF: {str(e)}"}

async def main():
    ruta_archivo = input("Ingresa la ruta al archivo PDF: ")
    
    resultado = await analizar_pdf(ruta_archivo)
    
    if "error" in resultado:
        print(f"Error: {resultado['error']}")
    else:
        print("\n" + "=" * 80)
        print(f"ANÁLISIS DE: {resultado.título_documento}")
        print(f"Autor: {resultado.autor}")
        print("=" * 80)
        
        print("\nRESUMEN EJECUTIVO:")
        print(resultado.resumen_ejecutivo)
        
        print("\nPUNTOS CLAVE:")
        for i, punto in enumerate(resultado.puntos_clave, 1):
            print(f"{i}. {punto.título}")
            print(f"   {punto.descripción}")
            print(f"   Páginas: {', '.join(map(str, punto.páginas))}")
            print()
        
        print("CONCLUSIÓN:")
        print(resultado.conclusión)
        print("=" * 80)

if __name__ == "__main__":
    asyncio.run(main())
```

Este agente especializado utiliza modelos de Pydantic para estructurar su análisis de documentos PDF, lo que facilita su integración en aplicaciones más complejas.

## Integrando MCP-Agent en Aplicaciones

MCP-Agent puede integrarse fácilmente en diferentes tipos de aplicaciones:

### Integración con API Web (FastAPI)

```python
# api_app.py
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
import asyncio
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client

app = FastAPI(title="API de Asistente IA")

# Configuración de MCP-Agent
mcp_app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(mcp_app.get_mcp_config())

# Definir agente
asistente_agente = Agent(
    name="AsistenteAPI",
    instruction="Eres un asistente que responde consultas a través de una API.",
    mcp_servers=["brave"]
)

# Modelo para las solicitudes
class ConsultaRequest(BaseModel):
    consulta: str
    usuario_id: str = None

# Modelo para las respuestas
class ConsultaResponse(BaseModel):
    respuesta: str
    tiempo_procesamiento: float

@app.post("/consulta", response_model=ConsultaResponse)
async def procesar_consulta(request: ConsultaRequest, background_tasks: BackgroundTasks):
    """Endpoint para procesar consultas al asistente."""
    tiempo_inicio = asyncio.get_event_loop().time()
    
    async with connection_manager.connect() as connection_map:
        cliente = gen_client(asistente_agente, connection_map, mcp_app.get_llm_config())
        
        # Personalizar consulta si hay ID de usuario
        prefijo = f"[Usuario: {request.usuario_id}] " if request.usuario_id else ""
        consulta_completa = f"{prefijo}{request.consulta}"
        
        # Generar respuesta
        respuesta = await cliente.generate_str(consulta_completa)
    
    tiempo_fin = asyncio.get_event_loop().time()
    tiempo_total = round(tiempo_fin - tiempo_inicio, 2)
    
    # Registrar la consulta en segundo plano
    if request.usuario_id:
        background_tasks.add_task(
            registrar_consulta, 
            usuario_id=request.usuario_id,
            consulta=request.consulta,
            respuesta=respuesta
        )
    
    return ConsultaResponse(
        respuesta=respuesta,
        tiempo_procesamiento=tiempo_total
    )

async def registrar_consulta(usuario_id: str, consulta: str, respuesta: str):
    """Registra la consulta en una base de datos o sistema de logs."""
    # Implementación del registro (simulada)
    print(f"Registrando consulta para usuario {usuario_id}")
    # En una implementación real, guardarías en base de datos

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Integración con Streamlit

```python
# streamlit_app.py
import streamlit as st
import asyncio
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client

# Configuración de Streamlit
st.set_page_config(
    page_title="Asistente MCP-Agent",
    page_icon="🤖",
    layout="wide"
)

# Función para ejecutar código asíncrono en Streamlit
def run_async(coro):
    loop = asyncio.new_event_loop()
    task = loop.create_task(coro)
    return loop.run_until_complete(task)

# Inicializar estado de sesión
if "messages" not in st.session_state:
    st.session_state.messages = []

# Cargar configuración de MCP-Agent
@st.cache_resource
def cargar_configuracion():
    app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
    connection_manager = MCPConnectionManager(app.get_mcp_config())
    return app, connection_manager

app, connection_manager = cargar_configuracion()

# Definir agente
asistente_agente = Agent(
    name="AsistenteStreamlit",
    instruction="""
    Eres un asistente amigable que ayuda a los usuarios en una aplicación web Streamlit.
    Proporciona respuestas claras, útiles y concisas. Puedes buscar información en línea
    cuando sea necesario.
    """,
    mcp_servers=["brave"]
)

# Título de la aplicación
st.title("💬 Asistente con MCP-Agent")
st.subheader("Pregúntame lo que quieras")

# Mostrar mensajes anteriores
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# Entrada del usuario
prompt = st.chat_input("¿En qué puedo ayudarte hoy?")

if prompt:
    # Añadir mensaje del usuario al historial
    st.session_state.messages.append({"role": "user", "content": prompt})
    
    # Mostrar mensaje del usuario
    with st.chat_message("user"):
        st.markdown(prompt)
    
    # Generar respuesta
    with st.chat_message("assistant"):
        message_placeholder = st.empty()
        message_placeholder.markdown("⏳ Pensando...")
        
        async def generar_respuesta():
            async with connection_manager.connect() as connection_map:
                cliente = gen_client(asistente_agente, connection_map, app.get_llm_config())
                return await cliente.generate_str(prompt)
        
        respuesta = run_async(generar_respuesta())
        
        # Mostrar respuesta
        message_placeholder.markdown(respuesta)
    
    # Añadir respuesta al historial
    st.session_state.messages.append({"role": "assistant", "content": respuesta})
```

## Conclusión

En esta sección, hemos aprendido a:

1. **Configurar MCP-Agent** mediante archivos YAML para gestionar nuestra aplicación.
2. **Definir agentes** con instrucciones específicas y acceso a servidores MCP.
3. **Construir clientes** para procesar consultas de usuarios utilizando nuestros agentes.
4. **Implementar procesamiento estructurado** utilizando modelos Pydantic.
5. **Desarrollar agentes especializados** para casos de uso específicos.
6. **Integrar MCP-Agent** en diferentes tipos de aplicaciones como APIs y aplicaciones web.

MCP-Agent proporciona una forma poderosa y flexible de crear agentes de IA que pueden acceder a diversas fuentes de datos y realizar acciones complejas, todo a través del Protocolo de Contexto de Modelo. Esta abstracción de alto nivel facilita enormemente el desarrollo de aplicaciones de IA contextualmente conscientes y capaces.

En la siguiente sección, exploraremos patrones de flujo de trabajo avanzados que permiten construir sistemas de IA aún más sofisticados.
