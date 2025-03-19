# Patrones y Flujos de Trabajo Avanzados (Continuación)

[Volver al índice principal](./mcp-agent-python-esqueleto.md) | [Volver a la Parte 1](./seccion6-patrones-avanzados.md) | [Volver a la Parte 2](./seccion6-patrones-avanzados-parte2.md)

## Patrón Swarm

El patrón Swarm implementa un sistema multi-agente donde un grupo de agentes similares colaboran en un problema complejo, compartiendo información y construyendo sobre las ideas de los demás. Este patrón se inspira en la inteligencia de enjambre observada en la naturaleza.

### Estructura del Patrón

```
                +---------------+
                | Coordinador   |
                +-------+-------+
                        |
                +-------v-------+
                | Problema      |
                +-------+-------+
                        |
    +----------+--------+----------+----------+
    |          |        |          |          |
+---v---+  +---v---+  +---v---+  +---v---+  +---v---+
|Agente1|  |Agente2|  |Agente3|  |Agente4|  |Agente5|
+---+---+  +---+---+  +---+---+  +---+---+  +---+---+
    |          |        |          |          |
    +----------+--------+----------+----------+
                        |
                +-------v-------+
                | Integrador    |
                +---------------+
```

### Implementación Básica

```python
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client
import asyncio
from typing import List, Dict, Any
import random

# Configuración básica
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(app.get_mcp_config())

# Definir agente coordinador
agente_coordinador = Agent(
    name="Coordinador",
    instruction="""
    Eres un coordinador que gestiona un grupo de pensadores colaborativos.
    Tu rol es distribuir un problema complejo entre varios agentes y
    coordinar sus contribuciones para llegar a una solución integrada.
    Debes ser claro en las instrucciones y asegurar diversidad de perspectivas.
    """,
    mcp_servers=[]
)

# Definir agente de enjambre (template)
def crear_agente_enjambre(estilo_pensamiento: str, id: int) -> Agent:
    """Crea un agente con un estilo de pensamiento específico."""
    return Agent(
        name=f"Pensador{id}",
        instruction=f"""
        Eres un pensador especializado en enfoque {estilo_pensamiento}.
        Aborda los problemas desde esta perspectiva específica.
        Complementa y amplía las ideas de otros pensadores.
        Sé original y evita simplemente repetir lo que otros han dicho.
        
        Puntos fuertes de tu estilo de pensamiento:
        - {get_strengths(estilo_pensamiento)}
        """,
        mcp_servers=["brave"]
    )

# Características de los diferentes estilos de pensamiento
def get_strengths(estilo: str) -> str:
    estilos = {
        "analítico": "Desglosar problemas complejos, identificar patrones y relaciones causales, evaluación objetiva de datos.",
        "creativo": "Generar ideas originales, pensar fuera de la caja, encontrar soluciones innovadoras, visualizar posibilidades.",
        "crítico": "Identificar fallas lógicas, cuestionar suposiciones, evaluar la calidad de argumentos y evidencia.",
        "pragmático": "Enfocarse en soluciones prácticas, considerar limitaciones reales, centrarse en la implementabilidad.",
        "sistémico": "Ver el panorama completo, comprender interconexiones, considerar efectos a largo plazo y consecuencias indirectas."
    }
    return estilos.get(estilo, "Adaptabilidad y pensamiento flexible.")

# Agente integrador
agente_integrador = Agent(
    name="Integrador",
    instruction="""
    Eres un integrador que sintetiza diversas perspectivas en una solución coherente.
    Tu rol es analizar las contribuciones de diferentes pensadores,
    identificar elementos valiosos, resolver contradicciones y
    articular una solución integral que incorpore lo mejor de cada perspectiva.
    Encuentra el equilibrio entre las distintas ideas y crea una síntesis superior.
    """,
    mcp_servers=[]
)

async def swarm_processing(
    problema: str, 
    num_agentes: int = 5, 
    estilos_pensamiento: List[str] = None
) -> Dict[str, Any]:
    """
    Implementa el patrón Swarm para abordar un problema complejo con múltiples agentes.
    
    Args:
        problema: Descripción del problema a resolver
        num_agentes: Número de agentes en el enjambre
        estilos_pensamiento: Lista de estilos de pensamiento para los agentes
        
    Returns:
        Diccionario con la solución integrada y contribuciones individuales
    """
    print(f"Iniciando procesamiento de enjambre para: {problema}")
    print(f"Número de agentes: {num_agentes}")
    
    # Estilos de pensamiento predeterminados si no se especifican
    if not estilos_pensamiento:
        estilos_pensamiento = ["analítico", "creativo", "crítico", "pragmático", "sistémico"]
    
    # Asegurar que tenemos suficientes estilos para el número de agentes
    while len(estilos_pensamiento) < num_agentes:
        estilos_pensamiento.extend(estilos_pensamiento)
    
    # Seleccionar estilos aleatorios si hay más estilos que agentes
    if len(estilos_pensamiento) > num_agentes:
        estilos_seleccionados = random.sample(estilos_pensamiento, num_agentes)
    else:
        estilos_seleccionados = estilos_pensamiento[:num_agentes]
    
    print(f"Estilos de pensamiento: {', '.join(estilos_seleccionados)}")
    
    async with connection_manager.connect() as connection_map:
        # Fase 1: Definir el problema con el coordinador
        cliente_coordinador = gen_client(agente_coordinador, connection_map, app.get_llm_config())
        
        prompt_coordinacion = f"""
        Define el siguiente problema para ser abordado por un grupo de {num_agentes} pensadores:
        
        PROBLEMA: {problema}
        
        Descompón este problema en aspectos clave que deberían ser considerados.
        Proporciona una descripción clara del desafío y sus dimensiones principales.
        """
        
        problema_elaborado = await cliente_coordinador.generate_str(prompt_coordinacion)
        
        # Fase 2: Generar soluciones con agentes del enjambre
        agentes_enjambre = [
            crear_agente_enjambre(estilo, i+1)
            for i, estilo in enumerate(estilos_seleccionados)
        ]
        
        contribuciones = []
        
        # Primera ronda: todos los agentes reciben el problema original
        for i, agente in enumerate(agentes_enjambre):
            cliente_agente = gen_client(agente, connection_map, app.get_llm_config())
            
            prompt_inicial = f"""
            Aborda el siguiente problema desde tu perspectiva única como pensador {estilos_seleccionados[i]}:
            
            PROBLEMA:
            {problema_elaborado}
            
            Proporciona tus ideas, observaciones y contribuciones iniciales.
            Enfócate en los aspectos donde tu estilo de pensamiento puede aportar más valor.
            """
            
            contribucion = await cliente_agente.generate_str(prompt_inicial)
            
            contribuciones.append({
                "agente_id": i+1,
                "estilo": estilos_seleccionados[i],
                "ronda": 1,
                "contribucion": contribucion
            })
            
            print(f"Agente {i+1} ({estilos_seleccionados[i]}) ha contribuido en la ronda 1")
        
        # Segunda ronda: los agentes reciben todas las contribuciones anteriores
        todas_contribuciones_r1 = "\n\n".join([
            f"CONTRIBUCIÓN DE PENSADOR {c['agente_id']} ({c['estilo'].upper()}):\n{c['contribucion']}"
            for c in contribuciones
        ])
        
        for i, agente in enumerate(agentes_enjambre):
            cliente_agente = gen_client(agente, connection_map, app.get_llm_config())
            
            prompt_segunda_ronda = f"""
            Observa las contribuciones de todos los pensadores y construye sobre ellas:
            
            PROBLEMA:
            {problema_elaborado}
            
            CONTRIBUCIONES EXISTENTES:
            {todas_contribuciones_r1}
            
            Ahora, proporciona nuevas ideas o mejora las existentes.
            No repitas simplemente lo ya dicho; añade valor con tu perspectiva única.
            Puedes expandir ideas prometedoras, combinar conceptos o llenar vacíos identificados.
            Enfócate en crear una solución más completa desde tu perspectiva {estilos_seleccionados[i]}.
            """
            
            contribucion_r2 = await cliente_agente.generate_str(prompt_segunda_ronda)
            
            contribuciones.append({
                "agente_id": i+1,
                "estilo": estilos_seleccionados[i],
                "ronda": 2,
                "contribucion": contribucion_r2
            })
            
            print(f"Agente {i+1} ({estilos_seleccionados[i]}) ha contribuido en la ronda 2")
        
        # Fase 3: Integrar todas las contribuciones
        cliente_integrador = gen_client(agente_integrador, connection_map, app.get_llm_config())
        
        todas_contribuciones = "\n\n".join([
            f"CONTRIBUCIÓN DE PENSADOR {c['agente_id']} ({c['estilo'].upper()}) - RONDA {c['ronda']}:\n{c['contribucion']}"
            for c in contribuciones
        ])
        
        prompt_integracion = f"""
        Integra las siguientes contribuciones en una solución cohesiva y completa:
        
        PROBLEMA:
        {problema_elaborado}
        
        CONTRIBUCIONES:
        {todas_contribuciones}
        
        Sintetiza estas perspectivas diversas en una solución integral.
        Resuelve contradicciones, combina ideas complementarias y asegura una respuesta completa.
        Incluye lo mejor de cada perspectiva para crear una solución que sea mayor que la suma de sus partes.
        """
        
        solucion_integrada = await cliente_integrador.generate_str(prompt_integracion)
        
        return {
            "problema_original": problema,
            "problema_elaborado": problema_elaborado,
            "contribuciones": contribuciones,
            "solucion_integrada": solucion_integrada
        }
```

### Caso de Uso: Diseño de Estrategia Empresarial

```python
async def disenar_estrategia_empresarial(desafio: str) -> Dict[str, Any]:
    """Diseña una estrategia empresarial utilizando inteligencia colectiva."""
    
    estilos_pensamiento = [
        "estratégico", 
        "financiero", 
        "marketing", 
        "operacional",
        "innovador"
    ]
    
    resultado = await swarm_processing(
        problema=f"Diseñar una estrategia empresarial para: {desafio}",
        num_agentes=5,
        estilos_pensamiento=estilos_pensamiento
    )
    
    print("\n" + "=" * 80)
    print(f"ESTRATEGIA EMPRESARIAL PARA: {desafio}")
    print("=" * 80)
    print(resultado["solucion_integrada"])
    print("=" * 80)
    
    return resultado

# Uso:
# await disenar_estrategia_empresarial("Una startup que desarrolla servidores MCP personalizados para verticales industriales específicas")
```

## Combinando Patrones para Soluciones Complejas

Los patrones presentados pueden combinarse para crear sistemas más sofisticados. Aquí exploramos cómo construir un sistema de agentes avanzado utilizando múltiples patrones.

### Sistema ROES: Router-Orchestrator-Evaluator-Swarm

```
                +-------------+
                |   Router    |
                +------+------+
                       |
       +-----------+---+---+-----------+
       |           |       |           |
+------v----+ +----v-----+ +-----v----+ +----v-----+
| Sistema A | | Sistema B | | Sistema C| | Sistema D|
| (PROBLEM  | | (RESEARCH)| | (CREATIVE| | (CRITIC) |
|  SOLVING) |  +---------+  | WORK)    | +---------+
+-----+-----+              +-----+-----+
      |                          |
+-----v-----+              +-----v-----+
|Orchestrat.|              |  Swarm    |
+-----+-----+              +-----+-----+
      |                          |
+-----v-----+              +-----v-----+
|Evaluator/ |              |Evaluator/ |
|Optimizer  |              |Optimizer  |
+-----+-----+              +-----+-----+
      |                          |
      +----------+---------------+
                 |
          +------v------+
          | Integrador  |
          +-------------+
```

Este sistema combina cuatro patrones:
1. **Router** para clasificar y dirigir consultas al subsistema adecuado
2. **Orchestrator-Workers** para resolución estructurada de problemas
3. **Swarm** para tareas creativas que se benefician de múltiples perspectivas
4. **Evaluator-Optimizer** aplicado a las salidas para garantizar calidad

### Implementación Conceptual

```python
async def roes_system(consulta: str) -> Dict[str, Any]:
    """
    Sistema ROES: Implementa una combinación de patrones Router-Orchestrator-Evaluator-Swarm
    para procesamiento avanzado de consultas.
    """
    print(f"Procesando consulta en sistema ROES: {consulta}")
    
    async with connection_manager.connect() as connection_map:
        # Paso 1: Enrutamiento
        tipo_consulta = await clasificar_tipo_consulta(consulta, connection_map)
        
        # Paso 2: Procesamiento según tipo
        if tipo_consulta == "problem_solving":
            # Usar patrón Orchestrator-Workers
            resultado_bruto = await orchestrator_workers_processing(consulta)
            subsistema = "Orchestrator-Workers"
        elif tipo_consulta == "research":
            # Usar patrón Parallel
            resultado_bruto = await parallel_processing(consulta)
            subsistema = "Parallel"
        elif tipo_consulta == "creative":
            # Usar patrón Swarm
            resultado_bruto = await swarm_processing(consulta)
            subsistema = "Swarm"
        else:  # "critic" o fallback
            # Usar router simple
            resultado_bruto = await router_processing(consulta)
            subsistema = "Router"
        
        solucion_inicial = resultado_bruto.get("solucion_final", 
                                              resultado_bruto.get("solucion_integrada", 
                                                                 resultado_bruto.get("informe_completo", 
                                                                                    resultado_bruto.get("respuesta", "No se generó solución"))))
        
        # Paso 3: Evaluación y optimización
        criterios = ["completitud", "precisión", "claridad", "aplicabilidad", "originalidad"]
        resultado_refinado = await evaluator_optimizer_processing(
            problema=consulta,
            criterios=criterios,
            umbral_calidad=8.0,
            max_iteraciones=2
        )
        
        # Paso 4: Integración y presentación
        return {
            "consulta_original": consulta,
            "subsistema_principal": subsistema,
            "solucion_inicial": solucion_inicial,
            "solucion_final": resultado_refinado["solucion_final"],
            "calificacion_final": resultado_refinado["calificacion_final"],
            "metadatos": {
                "tipo_consulta": tipo_consulta,
                "criterios_evaluacion": criterios,
                "resultado_subsistema": resultado_bruto,
                "resultado_refinado": resultado_refinado
            }
        }

async def clasificar_tipo_consulta(consulta: str, connection_map) -> str:
    """Clasifica la consulta en uno de los tipos principales para el sistema ROES."""
    # Implementación de clasificación...
    # Retorna uno de: "problem_solving", "research", "creative", "critic"
    return "problem_solving"  # Ejemplo simplificado
```

## Mejores Prácticas para Sistemas Multi-Agente

Al implementar estos patrones avanzados, considera las siguientes mejores prácticas:

### 1. Diseño de Instrucciones Claras

Las instrucciones para cada agente deben ser:
- **Específicas y enfocadas**: Define claramente el rol y las responsabilidades.
- **Complementarias**: Asegura que los agentes se complementen sin duplicar funcionalidades.
- **Independientes**: Minimiza dependencias rígidas entre agentes.

```python
# Ejemplo de instrucciones bien diseñadas para un agente evaluador
agente_evaluador = Agent(
    name="Evaluador",
    instruction="""
    PROPÓSITO:
    Evaluar soluciones según criterios específicos y proporcionar retroalimentación constructiva.
    
    RESPONSABILIDADES:
    1. Analizar soluciones objetivamente contra criterios predefinidos
    2. Calificar cada criterio en escala numérica (1-10)
    3. Identificar fortalezas específicas para mantener
    4. Identificar debilidades concretas para mejorar
    5. Proporcionar sugerencias accionables para optimización
    
    LIMITACIONES:
    - No generar nuevas soluciones (solo evaluar)
    - No considerar factores fuera de los criterios especificados
    
    FORMATO DE SALIDA:
    Proporcionar evaluación en formato JSON con estructura específica...
    """,
    mcp_servers=[]
)
```

### 2. Gestión de Complejidad

- **Modularidad**: Divide responsabilidades en componentes manejables.
- **Abstracción**: Crea capas de abstracción para simplificar interacciones complejas.
- **Registro y monitoreo**: Implementa registro detallado para depuración.

```python
# Ejemplo de helper para registro detallado
def log_agent_activity(agente_nombre: str, accion: str, entrada: str = None, salida: str = None):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    log_entry = {
        "timestamp": timestamp,
        "agente": agente_nombre,
        "accion": accion,
        "entrada": entrada[:500] + "..." if entrada and len(entrada) > 500 else entrada,
        "salida": salida[:500] + "..." if salida and len(salida) > 500 else salida
    }
    
    print(f"[{timestamp}] {agente_nombre} - {accion}")
    
    # Guardar en archivo o base de datos...
    with open("agent_logs.jsonl", "a") as f:
        f.write(json.dumps(log_entry) + "\n")
```

### 3. Optimización de Comunicación

- **Estructuración de mensajes**: Usa formatos consistentes como JSON.
- **Compresión de contexto**: Transmite solo información esencial entre agentes.
- **Evitar bottlenecks**: Optimiza patrones de comunicación para evitar cuellos de botella.

```python
# Ejemplo de función para extraer información esencial
def extraer_info_clave(texto_completo: str, max_tokens: int = 1000) -> str:
    """Extrae información clave de un texto largo para reducir tokens."""
    # Implementación simple: obtener primero el resumen con un agente
    resumen = obtener_resumen(texto_completo)
    
    # Luego extraer frases clave
    frases_clave = extraer_frases_importantes(texto_completo)
    
    # Combinar para optimizar contexto
    resultado = f"{resumen}\n\nPuntos clave:\n" + "\n".join([f"- {f}" for f in frases_clave])
    
    # Truncar si sigue siendo demasiado largo
    if len(resultado.split()) > max_tokens:
        return resultado.split()[:max_tokens].join(" ") + "..."
    
    return resultado
```

### 4. Validación y Control de Calidad

- **Comprobaciones de coherencia**: Verifica que las salidas de los agentes sean coherentes.
- **Testing sistemático**: Implementa casos de prueba para distintos escenarios.
- **Sistemas de fallback**: Diseña mecanismos de respaldo para manejar fallos.

```python
# Ejemplo de función de validación para salidas de agentes
def validar_salida_agente(salida: str, tipo_esperado: str) -> Dict[str, Any]:
    """Valida la salida de un agente y aplica correcciones si es necesario."""
    if tipo_esperado == "json":
        try:
            # Intentar extraer y parsear JSON
            start_idx = salida.find('{')
            end_idx = salida.rfind('}') + 1
            
            if start_idx >= 0 and end_idx > start_idx:
                return {
                    "valido": True,
                    "datos": json.loads(salida[start_idx:end_idx]),
                    "salida_original": salida
                }
            else:
                return {
                    "valido": False,
                    "error": "No se encontró estructura JSON",
                    "salida_original": salida
                }
        except Exception as e:
            return {
                "valido": False,
                "error": f"Error al parsear JSON: {str(e)}",
                "salida_original": salida
            }
    
    # Otras validaciones según tipo...
    
    return {
        "valido": True,
        "datos": salida,
        "salida_original": salida
    }
```

## Conclusión

Los patrones avanzados de agentes MCP permiten construir sistemas complejos y sofisticados que pueden abordar un amplio espectro de tareas y problemas. Al combinar estos patrones, puedes crear sistemas multi-agente altamente capaces, cada uno especializado en resolver diferentes aspectos de problemas complejos.

Estos patrones proporcionan una base sólida para el diseño de soluciones de IA que aprovechan al máximo el Protocolo de Contexto de Modelo, permitiendo la creación de sistemas que son:

- **Modulares y extensibles**: Fáciles de ampliar con nuevas capacidades
- **Robustos y resilientes**: Capaces de manejar diferentes tipos de consultas y escenarios
- **Escalables**: Pueden crecer en complejidad y funcionalidad según las necesidades
- **Mantenibles**: Estructurados de manera que facilitan la depuración y el mantenimiento

En la siguiente sección, exploraremos técnicas de depuración y pruebas para asegurar que nuestros sistemas de agentes MCP funcionen correctamente y proporcionen resultados confiables.
