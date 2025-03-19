# Implementando un Cliente MCP

[Volver al índice principal](./mcp-agent-python-esqueleto.md)

En esta sección, aprenderemos a crear un cliente MCP en Python que pueda conectarse a servidores MCP y utilizar sus capacidades. Veremos cómo establecer conexiones, consumir recursos, llamar a herramientas y utilizar prompts.

## Conexión a un Servidor MCP

El primer paso para implementar un cliente MCP es establecer una conexión con un servidor MCP. Para esto, utilizaremos las clases y funciones proporcionadas por el SDK de Python de MCP.

### Configuración Básica

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
import asyncio
import json

async def conectar_a_servidor():
    # Definir parámetros del servidor (asumiendo que el servidor se ejecuta como un proceso aparte)
    parametros = StdioServerParameters(
        command="python",  # Comando para ejecutar el servidor
        args=["ruta/al/servidor.py"]  # Argumentos (ruta al script del servidor)
    )
    
    # Establecer conexión con el servidor
    async with stdio_client(parametros) as (read, write):
        # Crear una sesión de cliente
        async with ClientSession(read, write) as session:
            # Inicializar la sesión
            await session.initialize()
            
            # Aquí irá el código para interactuar con el servidor
            print("Conexión establecida con el servidor MCP")
            
            # Ejemplo: mostrar capacidades del servidor
            capabilities = await session.get_capabilities()
            print(f"Capacidades del servidor: {json.dumps(capabilities, indent=2)}")

# Ejecutar la función asíncrona
if __name__ == "__main__":
    asyncio.run(conectar_a_servidor())
```

### Conexión a un Servidor Existente

Si el servidor MCP ya está en ejecución (por ejemplo, iniciado por Claude Desktop), puedes conectarte a él utilizando los parámetros adecuados:

```python
async def conectar_a_servidor_existente():
    # Conectar a un servidor que ya está en ejecución (ejemplo con SSE)
    from mcp.client.sse import sse_client
    
    # El servidor se ejecuta en http://localhost:3000
    async with sse_client("http://localhost:3000") as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            # Interactuar con el servidor...
```

### Manejo de Errores de Conexión

Es importante manejar los posibles errores de conexión:

```python
async def conectar_con_manejo_errores():
    try:
        parametros = StdioServerParameters(command="python", args=["servidor.py"])
        async with stdio_client(parametros) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                # Interactuar con el servidor...
    except ConnectionError as e:
        print(f"Error de conexión: {e}")
    except TimeoutError:
        print("La conexión ha excedido el tiempo de espera")
    except Exception as e:
        print(f"Error inesperado: {e}")
```

## Consumiendo Recursos

Una vez establecida la conexión, puedes consumir los recursos expuestos por el servidor MCP.

### Lectura de Recursos Básicos

```python
async def leer_recursos(session):
    # Leer un recurso simple
    datos = await session.read_resource("info://app")
    print(f"Información de la aplicación: {datos}")
    
    # Leer un recurso con parámetros
    usuario = await session.read_resource("users://1234")
    print(f"Datos del usuario: {usuario}")
```

### Lectura de Recursos Binarios

Algunos recursos pueden devolver contenido binario (como imágenes o archivos):

```python
async def leer_archivo(session, nombre_archivo):
    # Leer un archivo desde el servidor
    respuesta = await session.read_resource(f"files://{nombre_archivo}")
    
    # La respuesta incluye el contenido y el tipo MIME
    if "error" not in respuesta:
        contenido = respuesta["content"]
        tipo_mime = respuesta["mime_type"]
        
        # Guardar el archivo localmente
        with open(f"descargado_{nombre_archivo}", "wb") as f:
            f.write(contenido)
        
        print(f"Archivo descargado: descargado_{nombre_archivo} (tipo: {tipo_mime})")
    else:
        print(f"Error al leer el archivo: {respuesta['error']}")
```

### Lectura de Múltiples Recursos

Puedes leer múltiples recursos en secuencia o en paralelo:

```python
async def leer_multiples_recursos(session):
    # Lectura secuencial
    recursos = ["info://app", "stats://usage", "config://settings"]
    resultados = {}
    
    for uri in recursos:
        resultados[uri] = await session.read_resource(uri)
    
    # Lectura paralela (más eficiente)
    import asyncio
    tareas = [session.read_resource(uri) for uri in recursos]
    resultados_paralelos = await asyncio.gather(*tareas)
    
    # Combinar resultados
    resultados_combinados = dict(zip(recursos, resultados_paralelos))
    return resultados_combinados
```

## Llamando a Herramientas

Las herramientas permiten ejecutar acciones en el servidor MCP. Aquí veremos cómo llamar a estas herramientas desde tu cliente.

### Llamada Básica a una Herramienta

```python
async def llamar_herramienta(session):
    # Llamar a una herramienta simple
    resultado = await session.call_tool("saludar", {
        "params": {
            "nombre": "María"
        }
    })
    
    print(f"Resultado: {resultado}")  # Debería imprimir: "¡Hola, María! Bienvenido a mi servidor MCP."
```

### Llamada a Herramienta con Múltiples Parámetros

```python
async def calcular_con_herramienta(session):
    # Llamar a una herramienta con múltiples parámetros
    resultado = await session.call_tool("calcular_precio", {
        "params": {
            "precio_base": 100.0,
            "cantidad": 3,
            "descuento": 0.1,  # 10% de descuento
            "impuesto": 0.21   # 21% de impuesto
        }
    })
    
    print("Desglose del precio:")
    for clave, valor in resultado.items():
        print(f"  {clave}: {valor}")
```

### Manejo de Errores en Llamadas a Herramientas

```python
async def llamar_herramienta_segura(session, nombre_herramienta, parametros):
    try:
        resultado = await session.call_tool(nombre_herramienta, {
            "params": parametros
        })
        return resultado
    except Exception as e:
        print(f"Error al llamar a la herramienta {nombre_herramienta}: {e}")
        return {"error": str(e)}
```

## Utilizando Prompts

Los prompts son plantillas predefinidas que el servidor MCP proporciona para ayudar a generar texto estructurado con un LLM.

### Obtención y Uso de un Prompt Simple

```python
async def usar_prompt(session):
    # Obtener un prompt para resumir artículos
    prompt_resultado = await session.get_prompt("resumen_articulo", {
        "texto": "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat."
    })
    
    # El resultado contiene el prompt formateado
    print(f"Prompt generado: {prompt_resultado}")
    
    # Este prompt podría enviarse a un modelo de lenguaje
    # respuesta = await enviar_a_modelo(prompt_resultado)
```

### Uso de Prompts Complejos

```python
async def analizar_producto_con_prompt(session):
    # Obtener un prompt para analizar un producto
    prompt_analisis = await session.get_prompt("analisis_producto", {
        "nombre_producto": "Cámara DSLR Pro X200",
        "descripcion": "Cámara digital profesional con sensor de 24MP, ISO 100-25600, grabación 4K/60fps y conectividad WiFi/Bluetooth.",
        "precio": 1299.99,
        "categoria": "Fotografía"
    })
    
    # El prompt generado podría enviarse a un modelo de lenguaje para su procesamiento
    print("Prompt de análisis de producto generado:")
    print(prompt_analisis)
```

## Ejemplo Completo de Cliente MCP

A continuación, se muestra un ejemplo completo de un cliente MCP que interactúa con el servidor de biblioteca que creamos en la sección anterior:

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
import asyncio
import json
import sys

class ClienteBiblioteca:
    def __init__(self):
        self.session = None
    
    async def conectar(self):
        """Establece conexión con el servidor de biblioteca."""
        parametros = StdioServerParameters(
            command="python",
            args=["biblioteca_server.py"]  # Ruta al servidor de biblioteca
        )
        
        self.read, self.write = await stdio_client(parametros).__aenter__()
        self.session = await ClientSession(self.read, self.write).__aenter__()
        await self.session.initialize()
        print("Conexión establecida con la biblioteca.")
    
    async def cerrar(self):
        """Cierra la conexión con el servidor."""
        if self.session:
            await self.session.__aexit__(None, None, None)
        await self.read.__aexit__(None, None, None)
        print("Conexión cerrada.")
    
    async def listar_libros(self):
        """Obtiene y muestra la lista de todos los libros."""
        try:
            resultado = await self.session.read_resource("biblioteca://libros")
            libros = resultado["libros"]
            
            print("\n=== CATÁLOGO DE LIBROS ===")
            print(f"Total: {len(libros)} libros")
            print("-" * 50)
            
            for libro in libros:
                estado = "PRESTADO" if libro["prestado"] else "DISPONIBLE"
                print(f"ID: {libro['id']} | {libro['titulo']} ({libro['año']}) - {libro['autor']} | {estado}")
            
            print("-" * 50)
            return libros
        except Exception as e:
            print(f"Error al listar libros: {e}")
            return []
    
    async def ver_libro(self, libro_id):
        """Muestra información detallada de un libro específico."""
        try:
            libro = await self.session.read_resource(f"biblioteca://libros/{libro_id}")
            
            if "error" in libro:
                print(f"Error: {libro['error']}")
                return None
            
            print("\n=== DETALLES DEL LIBRO ===")
            print(f"ID: {libro['id']}")
            print(f"Título: {libro['titulo']}")
            print(f"Autor: {libro['autor']}")
            print(f"Año: {libro['año']}")
            print(f"Estado: {'Prestado' if libro['prestado'] else 'Disponible'}")
            
            # Mostrar información adicional si está disponible
            for clave, valor in libro.items():
                if clave not in ["id", "titulo", "autor", "año", "prestado"]:
                    print(f"{clave.capitalize()}: {valor}")
            
            return libro
        except Exception as e:
            print(f"Error al obtener detalles del libro: {e}")
            return None
    
    async def agregar_libro(self, titulo, autor, año):
        """Agrega un nuevo libro a la biblioteca."""
        try:
            resultado = await self.session.call_tool("agregar_libro", {
                "params": {
                    "titulo": titulo,
                    "autor": autor,
                    "año": int(año)
                }
            })
            
            print("\n=== LIBRO AGREGADO ===")
            print(f"ID: {resultado['id']}")
            print(f"Título: {resultado['titulo']}")
            print(f"Autor: {resultado['autor']}")
            print(f"Año: {resultado['año']}")
            print("El libro ha sido agregado exitosamente.")
            
            return resultado
        except Exception as e:
            print(f"Error al agregar libro: {e}")
            return None
    
    async def prestar_libro(self, libro_id):
        """Marca un libro como prestado."""
        try:
            resultado = await self.session.call_tool("prestar_libro", {
                "params": {
                    "libro_id": libro_id
                }
            })
            
            if "error" in resultado:
                print(f"Error: {resultado['error']}")
                return None
            
            print(f"\n✅ {resultado['mensaje']}")
            return resultado
        except Exception as e:
            print(f"Error al prestar libro: {e}")
            return None
    
    async def devolver_libro(self, libro_id):
        """Marca un libro como devuelto."""
        try:
            resultado = await self.session.call_tool("devolver_libro", {
                "params": {
                    "libro_id": libro_id
                }
            })
            
            if "error" in resultado:
                print(f"Error: {resultado['error']}")
                return None
            
            print(f"\n✅ {resultado['mensaje']}")
            return resultado
        except Exception as e:
            print(f"Error al devolver libro: {e}")
            return None
    
    async def buscar_libros(self, query):
        """Busca libros por título o autor."""
        try:
            resultado = await self.session.call_tool("buscar_libros", {
                "params": {
                    "query": query
                }
            })
            
            libros = resultado["resultados"]
            total = resultado["total"]
            
            print(f"\n=== RESULTADOS DE BÚSQUEDA PARA '{query}' ===")
            print(f"Se encontraron {total} libros.")
            
            if total > 0:
                print("-" * 50)
                for libro in libros:
                    estado = "PRESTADO" if libro["prestado"] else "DISPONIBLE"
                    print(f"ID: {libro['id']} | {libro['titulo']} ({libro['año']}) - {libro['autor']} | {estado}")
                print("-" * 50)
            
            return libros
        except Exception as e:
            print(f"Error al buscar libros: {e}")
            return []
    
    async def obtener_recomendacion(self, genero="", epoca=""):
        """Obtiene una recomendación de libros."""
        try:
            prompt = await self.session.get_prompt("recomendar_libro", {
                "genero": genero,
                "epoca": epoca
            })
            
            print("\n=== PROMPT DE RECOMENDACIÓN ===")
            print(prompt)
            print("\nEste prompt puede enviarse a un modelo de lenguaje para obtener recomendaciones.")
            
            # Aquí podrías enviar el prompt a un LLM
            # recomendaciones = await enviar_a_modelo(prompt)
            # print(recomendaciones)
            
            return prompt
        except Exception as e:
            print(f"Error al obtener recomendación: {e}")
            return None

async def ejecutar_menu():
    """Ejecuta un menú interactivo para el cliente de biblioteca."""
    cliente = ClienteBiblioteca()
    
    try:
        await cliente.conectar()
        
        while True:
            print("\n=== MENÚ DE BIBLIOTECA ===")
            print("1. Listar todos los libros")
            print("2. Ver detalles de un libro")
            print("3. Agregar nuevo libro")
            print("4. Prestar libro")
            print("5. Devolver libro")
            print("6. Buscar libros")
            print("7. Obtener recomendación")
            print("0. Salir")
            
            opcion = input("\nSeleccione una opción: ")
            
            if opcion == "1":
                await cliente.listar_libros()
            
            elif opcion == "2":
                libro_id = input("Ingrese el ID del libro: ")
                await cliente.ver_libro(libro_id)
            
            elif opcion == "3":
                titulo = input("Ingrese el título del libro: ")
                autor = input("Ingrese el autor: ")
                año = input("Ingrese el año de publicación: ")
                await cliente.agregar_libro(titulo, autor, año)
            
            elif opcion == "4":
                libro_id = input("Ingrese el ID del libro a prestar: ")
                await cliente.prestar_libro(libro_id)
            
            elif opcion == "5":
                libro_id = input("Ingrese el ID del libro a devolver: ")
                await cliente.devolver_libro(libro_id)
            
            elif opcion == "6":
                query = input("Ingrese texto a buscar en títulos o autores: ")
                await cliente.buscar_libros(query)
            
            elif opcion == "7":
                genero = input("Ingrese género (opcional): ")
                epoca = input("Ingrese época (opcional): ")
                await cliente.obtener_recomendacion(genero, epoca)
            
            elif opcion == "0":
                print("Saliendo del programa...")
                break
            
            else:
                print("Opción no válida. Intente de nuevo.")
    
    except Exception as e:
        print(f"Error en la aplicación: {e}")
    
    finally:
        await cliente.cerrar()

if __name__ == "__main__":
    asyncio.run(ejecutar_menu())
```

Este ejemplo completo demuestra un cliente MCP interactivo que se conecta al servidor de biblioteca y permite:

1. Listar todos los libros disponibles
2. Ver detalles de un libro específico
3. Agregar nuevos libros a la colección
4. Gestionar préstamos y devoluciones
5. Buscar libros por título o autor
6. Obtener recomendaciones personalizadas

El cliente gestiona correctamente la conexión, las llamadas a recursos y herramientas, y el manejo de errores, proporcionando una interfaz de usuario amigable a través de la consola.

## Integración con Modelos de Lenguaje

El ejemplo anterior podría extenderse para integrar un modelo de lenguaje que procese los prompts generados:

```python
async def procesar_con_llm(cliente, texto, libro_id):
    """Analiza un libro utilizando un LLM."""
    # Primero, obtenemos los detalles del libro
    libro = await cliente.ver_libro(libro_id)
    
    if not libro:
        return "No se pudo obtener información del libro."
    
    # Obtenemos el prompt para análisis
    prompt = await cliente.session.get_prompt("analizar_libro", {
        "titulo": libro["titulo"],
        "autor": libro["autor"],
        "sinopsis": texto  # El usuario proporciona una sinopsis
    })
    
    # Aquí conectaríamos con una API de LLM (ejemplo con Anthropic)
    import anthropic
    
    client = anthropic.Anthropic(
        api_key="tu-clave-api"
    )
    
    message = client.messages.create(
        model="claude-3-sonnet-20240229",
        max_tokens=1000,
        system="Eres un crítico literario experto.",
        messages=[{"role": "user", "content": prompt}]
    )
    
    # Retornamos la respuesta del modelo
    return message.content[0].text
```

Este código muestra cómo podrías integrar el cliente MCP con la API de Anthropic Claude para procesar los prompts generados y obtener análisis literarios.

## Creando una Interfaz Gráfica

Para aplicaciones más sofisticadas, puedes integrar el cliente MCP con una interfaz gráfica. Aquí hay un ejemplo simple utilizando Streamlit:

```python
# biblioteca_app.py
import streamlit as st
import asyncio
from cliente_biblioteca import ClienteBiblioteca  # Importar la clase que definimos antes

# Función para ejecutar código asíncrono desde Streamlit
def ejecutar_async(func, *args, **kwargs):
    loop = asyncio.new_event_loop()
    result = loop.run_until_complete(func(*args, **kwargs))
    loop.close()
    return result

# Crear una instancia del cliente
@st.cache_resource
def obtener_cliente():
    cliente = ClienteBiblioteca()
    ejecutar_async(cliente.conectar)
    return cliente

# Interfaz principal
st.title("Biblioteca Digital")
st.subheader("Sistema de Gestión de Libros con MCP")

cliente = obtener_cliente()

# Menú lateral
opcion = st.sidebar.selectbox(
    "Seleccione una opción",
    ["Listar Libros", "Ver Libro", "Agregar Libro", "Prestar/Devolver", "Buscar", "Recomendaciones"]
)

if opcion == "Listar Libros":
    st.header("Catálogo de Libros")
    if st.button("Actualizar Lista"):
        libros = ejecutar_async(cliente.listar_libros)
        
        # Mostrar en formato tabla
        if libros:
            datos = []
            for libro in libros:
                datos.append({
                    "ID": libro["id"],
                    "Título": libro["titulo"],
                    "Autor": libro["autor"],
                    "Año": libro["año"],
                    "Estado": "Prestado" if libro["prestado"] else "Disponible"
                })
            
            st.table(datos)

elif opcion == "Ver Libro":
    st.header("Detalles del Libro")
    libro_id = st.text_input("Ingrese el ID del libro:")
    
    if libro_id and st.button("Buscar"):
        libro = ejecutar_async(cliente.ver_libro, libro_id)
        
        if libro and "error" not in libro:
            st.subheader(f"{libro['titulo']} ({libro['año']})")
            st.write(f"**Autor:** {libro['autor']}")
            st.write(f"**Estado:** {'Prestado' if libro['prestado'] else 'Disponible'}")
            
            # Mostrar información adicional
            for clave, valor in libro.items():
                if clave not in ["id", "titulo", "autor", "año", "prestado"]:
                    st.write(f"**{clave.capitalize()}:** {valor}")

# ... Implementar las demás opciones del menú ...

# Al cerrar la aplicación (esto se ejecutará cuando Streamlit se cierre)
# Nota: En la práctica, Streamlit no garantiza la ejecución de este código al cerrar
def cerrar_cliente():
    ejecutar_async(cliente.cerrar)

# Registrar la función de cierre
import atexit
atexit.register(cerrar_cliente)
```

Este es un ejemplo básico de cómo podrías integrar el cliente MCP con Streamlit para crear una interfaz web amigable para la biblioteca.

## Conclusión

En esta sección, hemos aprendido cómo implementar un cliente MCP completo que puede:

1. Establecer conexiones con servidores MCP
2. Consumir recursos para obtener datos
3. Llamar a herramientas para realizar acciones
4. Utilizar prompts para generar contenido estructurado
5. Integrarse con modelos de lenguaje e interfaces gráficas

El Protocolo de Contexto de Modelo proporciona una forma estandarizada y potente para que los clientes interactúen con diferentes fuentes de datos y funcionalidades a través de servidores MCP, facilitando la creación de aplicaciones de IA más contextuales, informadas y útiles.
