# Creando un Servidor MCP Simple

[Volver al índice principal](./mcp-agent-python-esqueleto.md)

En esta sección, aprenderemos a crear un servidor MCP básico utilizando Python. Un servidor MCP es el componente fundamental que expone datos y funcionalidades a los agentes de IA a través del Protocolo de Contexto de Modelo.

## Estructura Básica de un Servidor MCP

### Configuración Inicial

Para crear un servidor MCP, comenzamos importando las clases necesarias y creando una instancia de `FastMCP`:

```python
from mcp.server.fastmcp import FastMCP

# Crear instancia del servidor con un nombre descriptivo
mcp = FastMCP(name="MiServidor")
```

El nombre del servidor debe ser descriptivo ya que será visible para los clientes y usuarios que interactúen con él.

### Ejecución del Servidor

Al final de nuestro archivo, necesitamos añadir el código para ejecutar el servidor:

```python
if __name__ == "__main__":
    # Iniciar el servidor usando el transporte estándar (stdio)
    mcp.run()
```

El método `run()` inicia el servidor y lo mantiene esperando solicitudes de los clientes. Por defecto, el servidor utiliza el transporte `stdio` (entrada/salida estándar), que es ideal para pruebas locales y uso con Claude Desktop.

### Servidor MCP Mínimo

Aquí tienes un ejemplo completo de un servidor MCP mínimo:

```python
from mcp.server.fastmcp import FastMCP

# Crear instancia del servidor
mcp = FastMCP(name="ServidorMinimo")

# Definir una herramienta simple
@mcp.tool()
def saludar(nombre: str) -> str:
    """Saluda al usuario por su nombre."""
    return f"¡Hola, {nombre}! Bienvenido a mi servidor MCP."

# Ejecutar el servidor
if __name__ == "__main__":
    mcp.run()
```

Este servidor expone una única herramienta llamada `saludar` que puede ser invocada por un cliente MCP.

## Definiendo Recursos

Los recursos en MCP son datos que los clientes pueden leer. Son similares a endpoints GET en una API REST y proporcionan información sin realizar acciones que modifiquen datos.

### Recurso Básico

```python
@mcp.resource("info://app")
def get_app_info():
    """Proporciona información básica sobre la aplicación."""
    return {
        "name": "Mi Aplicación",
        "version": "1.0.0",
        "author": "Tu Nombre",
        "description": "Un servidor MCP de ejemplo"
    }
```

### Recursos con Parámetros

Puedes definir recursos que acepten parámetros utilizando llaves `{}` en la URI:

```python
@mcp.resource("users://{user_id}")
def get_user(user_id: str):
    """Obtiene información de un usuario específico."""
    # Simular búsqueda en base de datos
    users = {
        "1": {"id": "1", "name": "Ana García", "role": "admin"},
        "2": {"id": "2", "name": "Carlos López", "role": "user"},
        "3": {"id": "3", "name": "Elena Martínez", "role": "editor"}
    }
    
    if user_id in users:
        return users[user_id]
    return {"error": "Usuario no encontrado"}
```

En este ejemplo, el parámetro `user_id` se extrae de la URI y se pasa a la función.

## Implementando Herramientas

Las herramientas son funciones que pueden ser llamadas por los clientes para realizar acciones. A diferencia de los recursos, las herramientas pueden modificar datos o tener efectos secundarios.

### Herramienta Simple

```python
@mcp.tool()
def sumar(a: int, b: int) -> int:
    """
    Suma dos números enteros.
    
    Args:
        a: Primer número
        b: Segundo número
        
    Returns:
        La suma de los dos números
    """
    return a + b
```

### Herramienta con Parámetros Opcionales

```python
@mcp.tool()
def formatear_texto(
    texto: str,
    mayusculas: bool = False,
    prefijo: str = "",
    sufijo: str = ""
) -> str:
    """
    Formatea un texto según los parámetros especificados.
    
    Args:
        texto: El texto a formatear
        mayusculas: Si es True, convierte el texto a mayúsculas
        prefijo: Texto a añadir al inicio
        sufijo: Texto a añadir al final
        
    Returns:
        El texto formateado
    """
    result = texto
    
    if mayusculas:
        result = result.upper()
        
    return f"{prefijo}{result}{sufijo}"
```

### Herramienta Asíncrona

Para operaciones que pueden llevar tiempo, como llamadas a APIs externas o procesamiento pesado, puedes definir herramientas asíncronas:

```python
import asyncio

@mcp.tool()
async def procesar_con_retraso(datos: str, retraso_segundos: int = 2) -> dict:
    """
    Simula el procesamiento de datos con un retraso.
    
    Args:
        datos: Los datos a procesar
        retraso_segundos: Tiempo de retraso en segundos
        
    Returns:
        Resultado del procesamiento
    """
    # Simular procesamiento que lleva tiempo
    await asyncio.sleep(retraso_segundos)
    
    return {
        "datos_originales": datos,
        "longitud": len(datos),
        "procesado_en": f"{retraso_segundos} segundos",
        "resultado": f"Procesado: {datos}"
    }
```

## Creando Prompts

Los prompts son plantillas predefinidas que ayudan a los LLMs a generar respuestas más efectivas para tareas específicas.

### Prompt Simple

```python
@mcp.prompt()
def resumen_articulo(texto: str) -> str:
    """Genera un prompt para resumir un artículo."""
    return f"""
    Por favor, genera un resumen conciso del siguiente artículo, 
    destacando los puntos clave y manteniendo la información esencial:
    
    {texto}
    
    Tu resumen debe ser claro, informativo y no exceder de 3 párrafos.
    """
```

### Prompt con Múltiples Parámetros

```python
@mcp.prompt()
def analisis_producto(
    nombre_producto: str,
    descripcion: str,
    precio: float,
    categoria: str
) -> str:
    """Genera un prompt para analizar un producto."""
    return f"""
    Realiza un análisis detallado del siguiente producto:
    
    Nombre: {nombre_producto}
    Categoría: {categoria}
    Precio: ${precio}
    Descripción: {descripcion}
    
    Tu análisis debe incluir:
    1. Fortalezas y debilidades del producto
    2. Comparación con productos similares en el mercado
    3. Sugerencias de marketing y posicionamiento
    4. Público objetivo recomendado
    
    Utiliza un tono profesional y objetivo en tu análisis.
    """
```

## Ejemplo Completo de Servidor MCP

A continuación, se presenta un ejemplo completo de un servidor MCP que integra recursos, herramientas y prompts para gestionar una biblioteca de libros:

```python
from mcp.server.fastmcp import FastMCP
import json
import os
from datetime import datetime

# Inicializar servidor MCP
mcp = FastMCP(name="BibliotecaServer")

# Simulación de una base de datos de libros
LIBROS_DB_FILE = "libros.json"

# Cargar o crear la base de datos de libros
def cargar_libros():
    if os.path.exists(LIBROS_DB_FILE):
        with open(LIBROS_DB_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    else:
        # Datos iniciales si no existe el archivo
        libros_iniciales = {
            "1": {"id": "1", "titulo": "Don Quijote", "autor": "Miguel de Cervantes", "año": 1605, "prestado": False},
            "2": {"id": "2", "titulo": "Cien años de soledad", "autor": "Gabriel García Márquez", "año": 1967, "prestado": False},
            "3": {"id": "3", "titulo": "1984", "autor": "George Orwell", "año": 1949, "prestado": True}
        }
        guardar_libros(libros_iniciales)
        return libros_iniciales

def guardar_libros(libros):
    with open(LIBROS_DB_FILE, "w", encoding="utf-8") as f:
        json.dump(libros, f, indent=2, ensure_ascii=False)

# Cargar libros al iniciar
libros = cargar_libros()

# --- RECURSOS ---

@mcp.resource("biblioteca://libros")
def obtener_libros():
    """Obtiene la lista completa de libros en la biblioteca."""
    return {"libros": list(libros.values())}

@mcp.resource("biblioteca://libros/{libro_id}")
def obtener_libro(libro_id: str):
    """Obtiene información detallada de un libro específico."""
    if libro_id in libros:
        return libros[libro_id]
    return {"error": "Libro no encontrado"}

@mcp.resource("biblioteca://estadisticas")
def obtener_estadisticas():
    """Obtiene estadísticas de la biblioteca."""
    total_libros = len(libros)
    libros_prestados = sum(1 for libro in libros.values() if libro["prestado"])
    libros_disponibles = total_libros - libros_prestados
    
    autores = set(libro["autor"] for libro in libros.values())
    años = [libro["año"] for libro in libros.values()]
    año_min = min(años) if años else 0
    año_max = max(años) if años else 0
    
    return {
        "total_libros": total_libros,
        "libros_prestados": libros_prestados,
        "libros_disponibles": libros_disponibles,
        "total_autores": len(autores),
        "libro_mas_antiguo": año_min,
        "libro_mas_reciente": año_max
    }

# --- HERRAMIENTAS ---

@mcp.tool()
def agregar_libro(titulo: str, autor: str, año: int) -> dict:
    """
    Agrega un nuevo libro a la biblioteca.
    
    Args:
        titulo: Título del libro
        autor: Nombre del autor
        año: Año de publicación
        
    Returns:
        Información del libro agregado
    """
    # Generar un nuevo ID (simple)
    nuevo_id = str(max(int(id) for id in libros.keys()) + 1)
    
    # Crear nuevo libro
    nuevo_libro = {
        "id": nuevo_id,
        "titulo": titulo,
        "autor": autor,
        "año": año,
        "prestado": False,
        "fecha_agregado": datetime.now().isoformat()
    }
    
    # Guardar en la "base de datos"
    libros[nuevo_id] = nuevo_libro
    guardar_libros(libros)
    
    return nuevo_libro

@mcp.tool()
def prestar_libro(libro_id: str) -> dict:
    """
    Marca un libro como prestado.
    
    Args:
        libro_id: ID del libro a prestar
        
    Returns:
        Resultado de la operación
    """
    if libro_id not in libros:
        return {"error": "Libro no encontrado"}
    
    if libros[libro_id]["prestado"]:
        return {"error": "El libro ya está prestado"}
    
    # Marcar como prestado
    libros[libro_id]["prestado"] = True
    libros[libro_id]["fecha_prestamo"] = datetime.now().isoformat()
    guardar_libros(libros)
    
    return {
        "success": True,
        "mensaje": f"Libro '{libros[libro_id]['titulo']}' marcado como prestado",
        "libro": libros[libro_id]
    }

@mcp.tool()
def devolver_libro(libro_id: str) -> dict:
    """
    Marca un libro como devuelto.
    
    Args:
        libro_id: ID del libro a devolver
        
    Returns:
        Resultado de la operación
    """
    if libro_id not in libros:
        return {"error": "Libro no encontrado"}
    
    if not libros[libro_id]["prestado"]:
        return {"error": "El libro no está prestado"}
    
    # Marcar como disponible
    libros[libro_id]["prestado"] = False
    if "fecha_prestamo" in libros[libro_id]:
        libros[libro_id]["ultimo_prestamo"] = libros[libro_id]["fecha_prestamo"]
        del libros[libro_id]["fecha_prestamo"]
    
    libros[libro_id]["fecha_devolucion"] = datetime.now().isoformat()
    guardar_libros(libros)
    
    return {
        "success": True,
        "mensaje": f"Libro '{libros[libro_id]['titulo']}' marcado como devuelto",
        "libro": libros[libro_id]
    }

@mcp.tool()
def buscar_libros(query: str) -> dict:
    """
    Busca libros por título o autor.
    
    Args:
        query: Texto a buscar en títulos o autores
        
    Returns:
        Lista de libros que coinciden con la búsqueda
    """
    query = query.lower()
    resultados = [
        libro for libro in libros.values()
        if query in libro["titulo"].lower() or query in libro["autor"].lower()
    ]
    
    return {
        "query": query,
        "resultados": resultados,
        "total": len(resultados)
    }

# --- PROMPTS ---

@mcp.prompt()
def recomendar_libro(genero: str = "", epoca: str = "") -> str:
    """Genera un prompt para recomendar libros."""
    restricciones = []
    if genero:
        restricciones.append(f"del género {genero}")
    if epoca:
        restricciones.append(f"de la época {epoca}")
    
    restricciones_texto = " ".join(restricciones)
    
    return f"""
    Basándote en tu conocimiento literario, recomienda 3 libros {restricciones_texto}.
    
    Para cada recomendación, incluye:
    1. Título y autor
    2. Breve sinopsis (2-3 oraciones)
    3. Por qué es una lectura recomendable
    4. Nivel de dificultad de lectura (principiante, intermedio, avanzado)
    
    Presenta tus recomendaciones en un formato claro y atractivo.
    """

@mcp.prompt()
def analizar_libro(titulo: str, autor: str, sinopsis: str) -> str:
    """Genera un prompt para analizar un libro."""
    return f"""
    Realiza un análisis literario del siguiente libro:
    
    Título: {titulo}
    Autor: {autor}
    Sinopsis: {sinopsis}
    
    Tu análisis debe incluir:
    1. Contexto histórico y cultural relevante
    2. Temas principales y simbolismo
    3. Estilo narrativo y técnicas literarias
    4. Impacto y relevancia en la literatura
    
    Utiliza ejemplos específicos para respaldar tus puntos cuando sea posible.
    """

# Ejecutar el servidor
if __name__ == "__main__":
    print(f"Iniciando servidor de Biblioteca MCP con {len(libros)} libros...")
    mcp.run()
```

Este ejemplo completo muestra un servidor MCP funcional que gestiona una biblioteca de libros, con capacidades para:

1. **Consultar información** (recursos):
   - Listar todos los libros
   - Ver detalles de un libro específico
   - Obtener estadísticas de la biblioteca

2. **Realizar acciones** (herramientas):
   - Agregar nuevos libros
   - Marcar libros como prestados/devueltos
   - Buscar libros por título o autor

3. **Generar contenido** (prompts):
   - Obtener recomendaciones de libros
   - Recibir análisis literarios

El servidor guarda los datos en un archivo JSON para persistencia básica, simulando una base de datos real.
