# Conceptos Básicos de MCP

[Volver al índice principal](./mcp-agent-python-esqueleto.md)

## ¿Qué es MCP?

El Protocolo de Contexto de Modelo (Model Context Protocol o MCP) es un estándar abierto desarrollado por Anthropic, lanzado oficialmente en noviembre de 2024. MCP establece un marco estandarizado para la comunicación entre asistentes de inteligencia artificial y fuentes de datos externas.

A diferencia de las integraciones tradicionales de IA, que a menudo requieren implementaciones personalizadas para cada fuente de datos, MCP proporciona una interfaz estandarizada que permite a los modelos de lenguaje (LLMs) interactuar con diversas fuentes de datos y herramientas de manera uniforme y segura.

El MCP fue diseñado para abordar varios desafíos clave:

1. **Limitaciones de conocimiento**: Los LLMs solo conocen la información incluida en sus datos de entrenamiento y tienen fechas de corte después de las cuales no tienen información.

2. **Actualizaciones costosas**: Entrenar nuevas versiones de modelos requiere enormes recursos computacionales y tiempo (6+ meses), lo que hace que su conocimiento esté siempre "desactualizado".

3. **Falta de conocimiento especializado**: Los LLMs se entrenan con datos públicos generales y no pueden comprender profundamente datos específicos de empresas o sectores.

4. **Integración fragmentada**: Sin un estándar unificado, cada conexión a sistemas externos requiere desarrollo personalizado, aumentando costos y complejidad.

MCP resuelve estos problemas proporcionando una capa estandarizada de comunicación que permite a los modelos de IA acceder a datos actualizados, específicos y relevantes de manera consistente y segura.

## Componentes Principales

El ecosistema MCP se compone de tres elementos fundamentales:

### 1. Servidores MCP

Los Servidores MCP son programas que proporcionan funcionalidades y acceso a datos a los modelos de IA:

- Actúan como proveedores de contexto y capacidades para los LLMs
- Pueden ejecutarse localmente en el dispositivo del usuario o desplegarse como servicios remotos
- Exponen recursos, herramientas y prompts a través de endpoints bien definidos
- Gestionan la autenticación y el control de acceso a los recursos subyacentes
- Procesan las solicitudes de los clientes y devuelven resultados en formatos estandarizados

Ejemplos de servidores MCP incluyen servidores para sistemas de archivos, bases de datos, APIs web, y herramientas especializadas.

### 2. Clientes MCP

Los Clientes MCP son componentes que permiten a los modelos de IA comunicarse con los servidores MCP:

- Actúan como intermediarios entre los LLMs y los servidores MCP
- Gestionan el ciclo de vida de las conexiones y sesiones
- Implementan el protocolo de comunicación MCP (mensajes, formatos, etc.)
- Proporcionan abstracciones de alto nivel para interactuar con las capacidades del servidor
- Se integran con los modelos de IA para facilitar el acceso a datos externos

Los clientes MCP pueden implementarse en diferentes lenguajes y frameworks, adaptándose a las necesidades específicas de cada aplicación.

### 3. Hosts MCP

Los Hosts MCP son aplicaciones que integran modelos de IA y clientes MCP:

- Proporcionan interfaces de usuario para interactuar con los modelos de IA
- Gestionan múltiples conexiones con servidores MCP
- Orquestan el flujo de trabajo entre el usuario, el modelo y los servidores
- Pueden incluir funcionalidades adicionales como historiales de conversación, gestión de usuarios, etc.

Ejemplos de hosts MCP incluyen Claude Desktop, IDEs como Cursor, y aplicaciones personalizadas construidas sobre SDKs de MCP.

## Capacidades y Funcionalidades

Los servidores MCP pueden exponer tres tipos principales de capacidades:

### 1. Recursos (Resources)

Los recursos son fuentes de datos pasivas que los clientes pueden leer. Son análogos a endpoints GET en una API REST:

- Proporcionan acceso a datos estructurados o no estructurados
- Pueden ser estáticos o dinámicos
- Utilizan URIs para identificar y acceder a recursos específicos
- Permiten parámetros para consultas específicas

Ejemplos:
```
files://documents/{document_id}
database://tables/{table_name}/rows
api://weather/{location}
```

### 2. Herramientas (Tools)

Las herramientas son funciones que los modelos pueden invocar para realizar acciones. Son análogas a endpoints POST en una API REST:

- Permiten operaciones que modifican datos o generan efectos secundarios
- Aceptan parámetros para controlar su comportamiento
- Devuelven resultados estructurados
- Pueden incluir validación de parámetros y gestión de errores

Ejemplos:
```python
@mcp.tool()
def search_database(query: str, limit: int = 10) -> list:
    """Busca en la base de datos utilizando la consulta proporcionada."""
    # Implementación...
    return results
```

### 3. Prompts

Los prompts son plantillas predefinidas que ayudan a los modelos a generar respuestas estructuradas:

- Proporcionan instrucciones estandarizadas para tareas específicas
- Pueden incluir variables para personalización
- Mejoran la consistencia de las respuestas del modelo
- Facilitan flujos de trabajo comunes

Ejemplos:
```python
@mcp.prompt()
def summarize_document(text: str) -> str:
    return f"""
    Proporciona un resumen conciso del siguiente texto, destacando los puntos clave
    y manteniendo la información esencial:
    
    {text}
    """
```

## Ciclo de Vida de una Interacción MCP

Una interacción típica utilizando MCP sigue estos pasos:

1. **Inicio de conexión**: El cliente MCP establece conexión con el servidor MCP.
2. **Descubrimiento de capacidades**: El cliente consulta las capacidades del servidor (recursos, herramientas, prompts).
3. **Solicitud del usuario**: El usuario realiza una consulta o petición.
4. **Análisis por el LLM**: El modelo analiza la consulta y determina qué capacidades del servidor necesita.
5. **Llamada a recursos/herramientas**: El cliente MCP solicita datos o ejecuta acciones a través del servidor.
6. **Procesamiento por el servidor**: El servidor procesa la solicitud y devuelve resultados.
7. **Generación de respuesta**: El modelo utiliza los resultados para generar una respuesta informada.
8. **Presentación al usuario**: La respuesta se muestra al usuario a través de la interfaz.

Este ciclo puede repetirse varias veces durante una conversación, permitiendo interacciones complejas y contextuales.

## Protocolos de Transporte

MCP admite diferentes protocolos de transporte para la comunicación entre clientes y servidores:

1. **stdio**: Utiliza entrada/salida estándar para la comunicación, ideal para servidores locales ejecutados como procesos independientes.
2. **SSE (Server-Sent Events)**: Permite comunicación en tiempo real a través de conexiones HTTP, adecuado para servidores remotos.

La elección del protocolo depende del escenario de despliegue y los requisitos de la aplicación.

## Ventajas del Enfoque MCP

La adopción de MCP ofrece numerosas ventajas:

1. **Estandarización**: Proporciona una interfaz común para todas las integraciones de datos.
2. **Modularidad**: Permite añadir, eliminar o reemplazar fuentes de datos sin cambiar la aplicación.
3. **Seguridad**: Implementa controles de acceso y autenticación consistentes.
4. **Escalabilidad**: Facilita la expansión a nuevas fuentes de datos y herramientas.
5. **Interoperabilidad**: Permite que diferentes modelos y aplicaciones utilicen los mismos servidores.
6. **Mantenibilidad**: Simplifica la actualización y el mantenimiento de las integraciones.

MCP representa un avance significativo en la forma en que los modelos de IA interactúan con datos externos, estableciendo una base sólida para construir aplicaciones de IA más potentes, contextuales y útiles.
