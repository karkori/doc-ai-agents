# Patrones y Flujos de Trabajo Avanzados (Continuación)

[Volver al índice principal](./mcp-agent-python-esqueleto.md) | [Volver a la Parte 1](./seccion6-patrones-avanzados.md)

## Patrón Orchestrator-Workers

El patrón Orchestrator-Workers divide un problema complejo en subtareas, asigna esas subtareas a trabajadores especializados y luego integra sus respuestas en una solución cohesiva.

### Estructura del Patrón

```
                +---------------+
                | Orchestrator  |
                +-------+-------+
                        |
     +--------+---------+----------+-------+
     |        |         |          |       |
+----v----+   |    +----v----+ +---v---+   |
| Worker1 |   |    | Worker2 | |Worker3|   |
+---------+   |    +---------+ +-------+   |
     |        |         |          |       |
     +--------+---------+----------+-------+
                        |
                +-------v-------+
                | Orchestrator  |
                +---------------+
```

### Implementación Básica

```python
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client
import asyncio
from typing import List, Dict, Any
import json

# Configuración básica
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(app.get_mcp_config())

# Definir agente orquestador
agente_orquestador = Agent(
    name="Orquestador",
    instruction="""
    Tu rol es descomponer problemas complejos en subtareas y luego integrar los resultados.
    
    Cuando recibas un problema:
    1. Analiza y descompón el problema en 3-5 subtareas concretas y abordables.
    2. Para cada subtarea, describe claramente lo que se requiere.
    3. Cuando recibas los resultados de las subtareas, intégralos en una solución cohesiva.
    4. Asegúrate de que no haya contradicciones o lagunas en la solución final.
    """,
    mcp_servers=[]  # No necesita servidores MCP
)

# Definir agente trabajador genérico
agente_trabajador = Agent(
    name="Trabajador",
    instruction="""
    Eres un especialista enfocado en resolver subtareas específicas de manera detallada y precisa.
    Concéntrate solo en la tarea asignada sin derivar a otras áreas.
    Proporciona resultados concretos y procesables.
    """,
    mcp_servers=["brave", "filesystem"]  # Servidores MCP comunes para todos los trabajadores
)

async def orchestrator_workers_processing(problema: str) -> Dict[str, Any]:
    """Implementa el patrón Orchestrator-Workers para resolver problemas complejos."""
    print(f"Procesando problema: {problema}")
    
    async with connection_manager.connect() as connection_map:
        # Fase 1: Descomponer el problema en subtareas
        cliente_orquestador = gen_client(agente_orquestador, connection_map, app.get_llm_config())
        
        prompt_descomposicion = f"""
        Descompón el siguiente problema en subtareas:
        
        PROBLEMA: {problema}
        
        Responde en formato JSON con la siguiente estructura:
        {{
            "subtareas": [
                {{"id": 1, "descripcion": "Primera subtarea", "instrucciones": "Detalles de lo que se debe hacer"}},
                {{"id": 2, "descripcion": "Segunda subtarea", "instrucciones": "Detalles de lo que se debe hacer"}},
                ...
            ]
        }}
        
        Asegúrate de que las subtareas sean claras, específicas y abarquen todo el problema.
        """
        
        plan_json = await cliente_orquestador.generate_str(prompt_descomposicion)
        
        # Extraer el JSON del resultado (eliminando texto adicional si lo hay)
        try:
            # Buscar el primer '{' y el último '}'
            start_idx = plan_json.find('{')
            end_idx = plan_json.rfind('}') + 1
            
            if start_idx >= 0 and end_idx > start_idx:
                plan_json = plan_json[start_idx:end_idx]
            
            plan = json.loads(plan_json)
        except Exception as e:
            print(f"Error al parsear JSON: {e}")
            plan = {"subtareas": [
                {"id": 1, "descripcion": "Analizar el problema", "instrucciones": "Analiza detenidamente el problema completo"},
                {"id": 2, "descripcion": "Proponer solución", "instrucciones": "Propón una solución integral al problema"}
            ]}
        
        # Fase 2: Ejecutar subtareas en paralelo
        subtareas = plan.get("subtareas", [])
        print(f"Problema descompuesto en {len(subtareas)} subtareas")
        
        resultados_subtareas = []
        
        for subtarea in subtareas:
            cliente_trabajador = gen_client(agente_trabajador, connection_map, app.get_llm_config())
            
            prompt_subtarea = f"""
            SUBTAREA #{subtarea['id']}: {subtarea['descripcion']}
            
            INSTRUCCIONES: {subtarea['instrucciones']}
            
            CONTEXTO DEL PROBLEMA PRINCIPAL: {problema}
            
            Proporciona una solución detallada y completa para esta subtarea específica.
            """
            
            resultado = await cliente_trabajador.generate_str(prompt_subtarea)
            
            resultados_subtareas.append({
                "id": subtarea['id'],
                "descripcion": subtarea['descripcion'],
                "resultado": resultado
            })
        
        # Fase 3: Integrar los resultados
        resultados_texto = "\n\n".join([
            f"RESULTADO SUBTAREA #{r['id']}: {r['descripcion']}\n{r['resultado']}"
            for r in resultados_subtareas
        ])
        
        prompt_integracion = f"""
        Integra los siguientes resultados de subtareas en una solución completa y cohesiva:
        
        PROBLEMA ORIGINAL: {problema}
        
        {resultados_texto}
        
        Proporciona una solución final que integre todos estos resultados de manera coherente.
        Asegúrate de que no haya contradicciones o lagunas en la solución.
        """
        
        solucion_final = await cliente_orquestador.generate_str(prompt_integracion)
        
        return {
            "problema": problema,
            "plan": subtareas,
            "resultados_subtareas": resultados_subtareas,
            "solucion_final": solucion_final
        }
```

### Caso de Uso: Desarrollo de Plan de Negocios

```python
async def generar_plan_negocios(idea: str) -> Dict[str, Any]:
    """Genera un plan de negocios detallado para una idea de startup."""
    problema = f"""
    Desarrolla un plan de negocios completo para la siguiente idea de startup:
    
    {idea}
    
    El plan debe incluir:
    - Resumen ejecutivo
    - Análisis de mercado
    - Estrategia de marketing
    - Plan financiero
    - Análisis de riesgos
    """
    
    resultado = await orchestrator_workers_processing(problema)
    
    print("\n" + "=" * 80)
    print(f"PLAN DE NEGOCIOS: {idea}")
    print("=" * 80)
    print(resultado["solucion_final"])
    print("=" * 80)
    
    return resultado

# Uso:
# await generar_plan_negocios("Una plataforma SaaS para gestionar servidores MCP y automatizar integraciones de IA")
```

## Patrón Evaluator-Optimizer

El patrón Evaluator-Optimizer implementa un ciclo de generación, evaluación y refinamiento que mejora iterativamente las soluciones hasta alcanzar un umbral de calidad.

### Estructura del Patrón

```
     +-------+      +-----------+
     |       |----->|           |
     | Gener |      | Evaluador |
     | ador  |<-----|           |
     |       |      +-----------+
     +---^---+            |
         |                |
         |          ¿Cumple criterios?
         |           /       \
         |         Sí         No
         |        /            \
         |    +--v----+    +---v----+
         |    |        |   |        |
         |    | Salida |   | Optimi |
         |    |        |   | zador  |
         |    +--------+   +---^----+
         |                     |
         +---------------------+
```

### Implementación Básica

```python
from mcp_agent import Agent, MCPApp, MCPConnectionManager, gen_client
import asyncio
from typing import Dict, Any, List
import json

# Configuración básica
app = MCPApp.from_yaml("mcp_agent.yaml", secrets_yaml="mcp_agent.secrets.yaml")
connection_manager = MCPConnectionManager(app.get_mcp_config())

# Definir agentes para el patrón
agente_generador = Agent(
    name="Generador",
    instruction="""
    Eres un generador creativo que produce soluciones iniciales a problemas.
    Tu objetivo es crear un primer borrador que aborde todos los requisitos.
    Sé detallado, preciso y asegúrate de cubrir todos los aspectos relevantes.
    """,
    mcp_servers=["brave"]
)

agente_evaluador = Agent(
    name="Evaluador",
    instruction="""
    Eres un evaluador crítico que analiza soluciones según criterios específicos.
    Tu trabajo es identificar fortalezas, debilidades y áreas de mejora.
    Proporciona calificaciones numéricas (1-10) para cada criterio y justifica tu evaluación.
    Sé justo pero exigente, con el objetivo de impulsar mejoras significativas.
    """,
    mcp_servers=[]
)

agente_optimizador = Agent(
    name="Optimizador",
    instruction="""
    Eres un optimizador que mejora soluciones basándose en evaluaciones críticas.
    Tu objetivo es abordar específicamente las debilidades identificadas
    y reforzar las fortalezas existentes.
    Mantén los elementos bien evaluados y mejora los aspectos débiles.
    """,
    mcp_servers=["brave"]
)

async def evaluator_optimizer_processing(
    problema: str, 
    criterios: List[str], 
    umbral_calidad: float = 8.0,
    max_iteraciones: int = 3
) -> Dict[str, Any]:
    """
    Implementa el patrón Evaluator-Optimizer para mejorar iterativamente soluciones.
    
    Args:
        problema: Descripción del problema a resolver
        criterios: Lista de criterios para evaluar la solución
        umbral_calidad: Calificación mínima promedio para considerar aceptable una solución
        max_iteraciones: Número máximo de iteraciones de mejora
    
    Returns:
        Diccionario con la solución final, historial de evaluaciones y métricas
    """
    print(f"Iniciando proceso Evaluator-Optimizer para: {problema}")
    print(f"Criterios de evaluación: {', '.join(criterios)}")
    
    async with connection_manager.connect() as connection_map:
        # Fase 1: Generar solución inicial
        cliente_generador = gen_client(agente_generador, connection_map, app.get_llm_config())
        
        solucion_actual = await cliente_generador.generate_str(
            f"Genera una solución para: {problema}\n\nConsidera estos criterios: {', '.join(criterios)}"
        )
        
        # Preparar historial
        historial = []
        iteracion = 0
        mejor_calificacion = 0
        
        # Ciclo de evaluación y optimización
        while iteracion < max_iteraciones:
            iteracion += 1
            print(f"\nIteración {iteracion}/{max_iteraciones}")
            
            # Fase 2: Evaluar la solución actual
            cliente_evaluador = gen_client(agente_evaluador, connection_map, app.get_llm_config())
            
            evaluacion_json = await cliente_evaluador.generate_str(
                f"""Evalúa esta solución para el problema: {problema}
                
                SOLUCIÓN A EVALUAR:
                {solucion_actual}
                
                CRITERIOS DE EVALUACIÓN:
                {chr(10).join([f"- {criterio}" for criterio in criterios])}
                
                Responde en formato JSON con calificaciones (1-10) para cada criterio.
                """
            )
            
            # Extraer el JSON
            try:
                start_idx = evaluacion_json.find('{')
                end_idx = evaluacion_json.rfind('}') + 1
                
                if start_idx >= 0 and end_idx > start_idx:
                    evaluacion = json.loads(evaluacion_json[start_idx:end_idx])
                else:
                    raise ValueError("No se encontró un objeto JSON válido")
            except Exception:
                # Crear una evaluación predeterminada
                evaluacion = {
                    "evaluaciones": [{"criterio": c, "calificacion": 5} for c in criterios],
                    "calificacion_promedio": 5.0,
                    "debilidades": ["No se pudo evaluar correctamente"]
                }
            
            # Guardar evaluación en historial
            historial.append({
                "iteracion": iteracion,
                "solucion": solucion_actual,
                "evaluacion": evaluacion
            })
            
            # Comprobar si cumple el umbral de calidad
            calificacion_actual = evaluacion.get("calificacion_promedio", 0)
            print(f"Calificación promedio: {calificacion_actual}/{umbral_calidad}")
            
            if calificacion_actual >= umbral_calidad:
                print(f"¡Solución alcanzó el umbral de calidad en la iteración {iteracion}!")
                mejor_calificacion = calificacion_actual
                break
            
            # Fase 3: Optimizar la solución
            cliente_optimizador = gen_client(agente_optimizador, connection_map, app.get_llm_config())
            
            debilidades = evaluacion.get("debilidades", ["No se especificaron debilidades"])
            debilidades_texto = "\n".join([f"- {d}" for d in debilidades])
            
            solucion_actual = await cliente_optimizador.generate_str(
                f"""Mejora esta solución basándote en la siguiente evaluación:
                
                PROBLEMA: {problema}
                
                SOLUCIÓN ACTUAL:
                {solucion_actual}
                
                DEBILIDADES A MEJORAR:
                {debilidades_texto}
                
                CRITERIOS DE EVALUACIÓN:
                {chr(10).join([f"- {criterio}" for criterio in criterios])}
                
                Proporciona una versión mejorada de la solución completa.
                """
            )
            
            print(f"Solución optimizada en iteración {iteracion}")
        
        # Seleccionar mejor solución del historial
        mejor_iteracion = max(historial, key=lambda x: x["evaluacion"].get("calificacion_promedio", 0))
        
        return {
            "problema": problema,
            "solucion_final": mejor_iteracion["solucion"],
            "calificacion_final": mejor_iteracion["evaluacion"].get("calificacion_promedio", 0),
            "iteraciones_totales": iteracion,
            "historial": historial
        }
```

### Caso de Uso: Generación de Documentación Técnica

```python
async def generar_documentacion_tecnica(componente: str) -> Dict[str, Any]:
    """Genera documentación técnica de alta calidad para un componente de software."""
    problema = f"Crear documentación técnica completa para: {componente}"
    
    criterios = [
        "Precisión técnica",
        "Claridad de explicaciones",
        "Estructura lógica",
        "Exhaustividad del contenido",
        "Ejemplos prácticos",
        "Consideración de la audiencia objetivo"
    ]
    
    resultado = await evaluator_optimizer_processing(
        problema=problema,
        criterios=criterios,
        umbral_calidad=8.5,
        max_iteraciones=3
    )
    
    print("\n" + "=" * 80)
    print(f"DOCUMENTACIÓN TÉCNICA PARA: {componente}")
    print(f"Calificación final: {resultado['calificacion_final']}/10")
    print("=" * 80)
    print(resultado["solucion_final"])
    print("=" * 80)
    
    return resultado

# Uso:
# await generar_documentacion_tecnica("API de Servidores MCP en Python")
```
