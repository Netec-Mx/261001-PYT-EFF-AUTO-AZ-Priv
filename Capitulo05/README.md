# Práctica: Pruebas y escalamiento de una automatización

## Metadatos

| Característica | Detalle |
| :--- | :--- |
| **Duración** | 120 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio práctico, transformarás un script de automatización síncrono y frágil (que extrae datos de ventas mediante llamadas HTTP sencillas) en un sistema de grado de producción resiliente, observable y altamente concurrente. Diseñarás e implementarás una suite de pruebas automatizadas con `pytest` empleando *mocks* para aislar fallos de red; configurarás un motor de logs jerárquico y rotativo utilizando `loguru`; y, finalmente, refactorizarás el proceso de extracción para ejecutar peticiones de manera paralela utilizando `concurrent.futures.ThreadPoolExecutor`.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Implementar un esquema de pruebas unitarias y de integración utilizando `pytest` para validar la resiliencia del pipeline ante fallos de APIs.
- [ ] Refactorizar un pipeline de extracción síncrono para ejecutar peticiones HTTP concurrentes utilizando `concurrent.futures.ThreadPoolExecutor`.
- [ ] Configurar un sistema de logging jerárquico y estructurado con `loguru` para facilitar la observabilidad en ambientes de producción.

## Prerrequisitos

Para completar este laboratorio de manera exitosa, requieres:
1. **Conocimientos teóricos y prácticos:**
   - Comprensión sólida de la programación orientada a objetos (POO) en Python (clases, métodos, herencia, excepciones).
   - Familiaridad básica con el consumo de APIs REST mediante la biblioteca `requests` de Python.
   - Manejo básico de la terminal o línea de comandos (creación de directorios, ejecución de scripts, activación de entornos virtuales).
2. **Herramientas y suscripciones:**
   - Cuenta activa y acceso a la extensión de VS Code de **GitHub Copilot Extension (versión 1.173.0)** con una licencia comercial de suscripción activa de GitHub, Inc.
   - Conexión a Internet estable y sin restricciones de proxy para la instalación de paquetes a través de `pip` y descarga de dependencias.

## Entorno de Laboratorio

El laboratorio debe ser ejecutado bajo las siguientes especificaciones técnicas de hardware y software:

### Requerimientos de Hardware

- **Procesador:** Arquitectura x86_64 o ARM64 de al menos 4 núcleos físicos.
- **Memoria RAM:** Mínimo 8 GB (16 GB recomendado para la ejecución simultánea de IDE, pruebas y navegadores).
- **Almacenamiento:** Mínimo 20 GB de espacio disponible en disco de estado sólido (SSD preferiblemente).

### Requerimientos de Software

| Herramienta / Paquete | Versión Exacta | Enlace de Referencia Oficial | Licencia |
| :--- | :--- | :--- | :--- |
| **Python** (Intérprete) | 3.12.2 (amd64/arm64) | [Python 3.12.2](https://www.python.org/downloads/release/python-3122/) | PSF License |
| **pytest** (Testing Framework) | 8.1.1 | [pytest on PyPI](https://pypi.org/project/pytest/8.1.1/) | MIT License |
| **loguru** (Logging) | 0.7.2 | [loguru on PyPI](https://pypi.org/project/loguru/0.7.2/) | MIT License |
| **requests** (HTTP Client) | 2.31.0 | [requests on PyPI](https://pypi.org/project/requests/2.31.0/) | Apache 2.0 |
| **Visual Studio Code** | 1.87.2 o superior | [VS Code Downloads](https://code.visualstudio.com/) | Freeware / Source code under MIT |

### Directorios y Configuración Global

Toda la práctica se desarrollará dentro del siguiente directorio de trabajo por defecto:
` /home/usuario/workspace/automatizacion_ventas`

Cualquier ruta relativa especificada en el desarrollo de este laboratorio asume este directorio como la raíz de ejecución.

---

## Instrucciones Paso a Paso

### Paso 1: Configurar el entorno de trabajo y dependencias

**Objetivo:** Crear la estructura de directorios, inicializar el entorno virtual de Python y configurar las herramientas de asistencia por IA (GitHub Copilot Chat) dentro de VS Code.

#### Instrucciones

1. Abre una terminal de tu sistema operativo y ubícate en la raíz donde deseas crear tu espacio de trabajo.
2. Ejecuta los siguientes comandos para crear la estructura de directorios necesaria para la automatización:
   ```bash
   mkdir -p /home/usuario/workspace/automatizacion_ventas/src
   mkdir -p /home/usuario/workspace/automatizacion_ventas/tests
   mkdir -p /home/usuario/workspace/automatizacion_ventas/logs
   cd /home/usuario/workspace/automatizacion_ventas
   ```
3. Crea y activa un entorno virtual aislado utilizando Python 3.12.2:
   ```bash
   # Creación del entorno virtual
   python3 -m venv .venv
   
   # Activación en Linux/macOS
   source .venv/bin/activate
   ```
4. Actualiza `pip` e instala las dependencias exactas del laboratorio:
   ```bash
   pip install --upgrade pip
   pip install requests==2.31.0 pytest==8.1.1 loguru==0.7.2
   ```
5. Abre el directorio de trabajo en Visual Studio Code:
   ```bash
   code .
   ```
6. **Configuración de Asistencia por IA (GitHub Copilot):**
   - Asegúrate de que la extensión **GitHub Copilot Extension (1.173.0)** esté activa en tu VS Code.
   - Abre el panel lateral de **Copilot Chat** (`Ctrl + Shift + I` o mediante el icono de chat en la barra lateral).
   - Recuerda que durante este laboratorio utilizaremos "prompts" estructurados para interactuar con la IA de forma iterativa, tratando las sugerencias como guías que deben ser validadas técnicamente, distinguiendo las instrucciones temporales de chat (prompts) de los comportamientos por defecto del modelo.

#### Resultado Esperado

La estructura de tu proyecto dentro de VS Code debe lucir exactamente así:
```text
automatizacion_ventas/
├── .venv/
├── logs/
├── src/
├── tests/
```

#### Verificación

Para certificar que el entorno está correctamente configurado y con las versiones exactas solicitadas, ejecuta en la terminal:
```bash
python -c "import pytest, loguru, requests; print('pytest:', pytest.__version__); print('loguru:', loguru.__version__); print('requests:', requests.__version__)"
```

**Salida de consola esperada:**
```text
pytest: 8.1.1
loguru: 0.7.2
requests: 2.31.0
```

---

### Paso 2: Desarrollar el pipeline de extracción síncrono (Base)

**Objetivo:** Crear un script de automatización base síncrono orientado a objetos que recupere órdenes de venta de una API externa simulada.

#### Instrucciones

1. Crea un archivo vacío en `src/pipeline_ventas.py`.
2. Utiliza la interfaz de GitHub Copilot Chat para guiar la estructura inicial del archivo mediante la introducción del siguiente prompt estructurado:
   > **Prompt de Entrada para Copilot Chat:**
   > "Genera una clase en Python llamada `ExtractorVentas` que consuma una API REST simulada en la URL `https://api.ejemplo.com/v1/orders`. La clase debe recibir en su constructor (`__init__`) un token de API y un tiempo de espera (`timeout`) de 5 segundos. Debe contener un método llamado `obtener_orden_por_id(self, order_id: int) -> dict` que realice una llamada HTTP GET a `https://api.ejemplo.com/v1/orders/{order_id}` usando la librería `requests`. El método debe incluir cabeceras de autorización básica `Authorization: Bearer <token>`. Si el código de respuesta HTTP no es 200, debe lanzar una excepción personalizada de tipo `APIException` definida en el mismo archivo. Por favor, escribe la lógica de forma puramente síncrona, usando prints simples para reportar el progreso."
3. Evalúa la respuesta de Copilot y asegúrate de que se adecúe al código estructurado que se presenta a continuación. Guarda el siguiente código limpio en `src/pipeline_ventas.py`:

```python
## /home/usuario/workspace/automatizacion_ventas/src/pipeline_ventas.py
import requests
import time

class APIException(Exception):
    """Excepción personalizada para errores del servidor o respuestas no exitosas (HTTP != 200)."""
    pass

class ExtractorVentas:
    """Clase encargada de interactuar de forma síncrona con el endpoint de órdenes de venta."""
    
    def __init__(self, api_token: str, timeout: int = 5):
        if not api_token:
            raise ValueError("El token de API de entrada no puede estar vacío.")
        self.api_token = api_token
        self.timeout = timeout
        self.base_url = "https://api.ejemplo.com/v1/orders"

    def obtener_orden_por_id(self, order_id: int) -> dict:
        """
        Realiza una petición síncrona GET a la API para extraer una sola orden.
        """
        if not isinstance(order_id, int) or order_id <= 0:
            raise TypeError("El ID de la orden debe ser un entero positivo válido.")

        url = f"{self.base_url}/{order_id}"
        headers = {
            "Authorization": f"Bearer {self.api_token}",
            "Accept": "application/json"
        }

        print(f"[INFO] Iniciando descarga de orden ID: {order_id}")
        try:
            response = requests.get(url, headers=headers, timeout=self.timeout)
        except requests.RequestException as exc:
            print(f"[ERROR] Error de conexión de red al recuperar orden {order_id}: {exc}")
            raise APIException(f"Fallo crítico de conexión para orden {order_id}") from exc

        if response.status_code != 200:
            print(f"[WARNING] API retornó código {response.status_code} para orden {order_id}")
            raise APIException(f"La API retornó código HTTP {response.status_code} para orden {order_id}")

        print(f"[INFO] Orden {order_id} descargada exitosamente.")
        return response.json()

    def procesar_lote_ordenes(self, lista_ids: list[int]) -> list[dict]:
        """
        Descarga de manera secuencial (síncrona) un lote completo de IDs de órdenes.
        """
        resultados = []
        for order_id in lista_ids:
            try:
                datos_orden = self.obtener_orden_por_id(order_id)
                resultados.append(datos_orden)
            except APIException as e:
                print(f"[ERROR] Omitiendo orden {order_id} debido a un error: {e}")
        return resultados
```

#### Resultado Esperado

El script base de extracción se ha creado sin errores de sintaxis y define una clase de extracción orientada a objetos preparada para el manejo de excepciones de conexión y códigos HTTP de error.

#### Verificación

Para verificar la importación correcta y la sintaxis limpia de la clase, ejecuta en la terminal de tu entorno virtual activo:
```bash
python -m py_compile src/pipeline_ventas.py
```
*(No debería arrojar ningún output o error si el archivo está bien estructurado)*.

---

### Paso 3: Crear la suite de pruebas unitarias e integración con pytest

**Objetivo:** Desarrollar pruebas unitarias utilizando fixtures de `pytest` y la librería `unittest.mock` de la biblioteca estándar para aislar las llamadas de red y probar flujos exitosos (*happy path*) y rutas de error (*sad path*).

#### Instrucciones

1. Crea un archivo vacío para las pruebas en `tests/test_pipeline_ventas.py`.
2. Utilizaremos el asistente Copilot Chat para diseñar los *mocks* de nuestras peticiones HTTP, de forma que no golpeemos a la API real en internet.
   > **Prompt de Entrada para Copilot Chat:**
   > "Genera un conjunto de pruebas automatizadas con `pytest` para la clase `ExtractorVentas`. Necesito que crees una fixture de pytest que instancie la clase con un token simulado. Además, escribe tres pruebas unitarias utilizando el patrón Arrange-Act-Assert (AAA): 1. `test_obtener_orden_exito`: Usa `unittest.mock.patch` para simular que `requests.get` retorna un objeto respuesta con status_code=200 y un JSON de prueba. 2. `test_obtener_orden_error_http`: Simula un HTTP status 500 y comprueba con `pytest.raises` que se lance `APIException`. 3. `test_obtener_orden_excepcion_red`: Simula que `requests.get` lanza un `requests.RequestException` para asegurar que nuestra lógica lo captura y lo convierte en `APIException`."
3. Revisa con criterio técnico el código proporcionado por Copilot y asegúrate de consolidar el siguiente diseño estructurado en el archivo `tests/test_pipeline_ventas.py`:

```python
## /home/usuario/workspace/automatizacion_ventas/tests/test_pipeline_ventas.py
import pytest
from unittest.mock import patch, MagicMock
import requests

from src.pipeline_ventas import ExtractorVentas, APIException

@pytest.fixture
def extractor_valido():
    """Fixture que provee una instancia de ExtractorVentas limpia para cada prueba."""
    return ExtractorVentas(api_token="token_de_prueba_secreto_123", timeout=3)

def test_inicializacion_extractor_invalido():
    """Valida que no se permita instanciar la clase sin un token válido."""
    with pytest.raises(ValueError) as exc_info:
        ExtractorVentas(api_token="")
    assert "El token de API de entrada no puede estar vacío" in str(exc_info.value)

def test_obtener_orden_tipo_dato_invalido(extractor_valido):
    """Valida que se verifiquen los tipos de datos de entrada en el ID de la orden."""
    with pytest.raises(TypeError) as exc_info:
        extractor_valido.obtener_orden_por_id("ID_no_entero")
    assert "El ID de la orden debe ser un entero positivo" in str(exc_info.value)

@patch("src.pipeline_ventas.requests.get")
def test_obtener_orden_exito(mock_get, extractor_valido):
    """
    Escenario: Happy Path.
    Valida la extracción correcta cuando la API responde de forma exitosa (HTTP 200).
    """
    # Arrange (Preparar)
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"id": 101, "monto": 1500.50, "estado": "Completada"}
    mock_get.return_value = mock_response

    # Act (Actuar)
    resultado = extractor_valido.obtener_orden_por_id(101)

    # Assert (Asertar)
    assert resultado == {"id": 101, "monto": 1500.50, "estado": "Completada"}
    mock_get.assert_called_once_with(
        "https://api.ejemplo.com/v1/orders/101",
        headers={"Authorization": "Bearer token_de_prueba_secreto_123", "Accept": "application/json"},
        timeout=3
    )

@patch("src.pipeline_ventas.requests.get")
def test_obtener_orden_error_http(mock_get, extractor_valido):
    """
    Escenario: Sad Path.
    Valida que la clase lance correctamente APIException si el código HTTP es de error (HTTP 500).
    """
    # Arrange
    mock_response = MagicMock()
    mock_response.status_code = 500
    mock_get.return_value = mock_response

    # Act & Assert
    with pytest.raises(APIException) as exc_info:
        extractor_valido.obtener_orden_por_id(999)

    assert "La API retornó código HTTP 500" in str(exc_info.value)

@patch("src.pipeline_ventas.requests.get")
def test_obtener_orden_excepcion_red(mock_get, extractor_valido):
    """
    Escenario: Sad Path de Red.
    Valida que los fallos de conectividad del cliente HTTP se capturen y mapeen a una APIException limpia.
    """
    # Arrange
    mock_get.side_effect = requests.Timeout("El servidor tardó demasiado en responder")

    # Act & Assert
    with pytest.raises(APIException) as exc_info:
        extractor_valido.obtener_orden_por_id(102)

    assert "Fallo crítico de conexión" in str(exc_info.value)
```

#### Resultado Esperado

Se tiene una suite de pruebas estructurada y aislada que comprueba el correcto comportamiento del código utilizando aserciones nativas y validación de tipos, así como la correcta gestión de errores de integración simulados.

#### Verificación

Ejecuta la suite de pruebas unitarias desde la terminal en el directorio raíz del proyecto:
```bash
pytest -v tests/test_pipeline_ventas.py
```

Deberías ver una salida indicando que las 5 pruebas han pasado de manera exitosa:
```text
============================= test session starts ==============================
collected 5 items

tests/test_pipeline_ventas.py::test_inicializacion_extractor_invalido PASSED [ 20%]
tests/test_pipeline_ventas.py::test_obtener_orden_tipo_dato_invalido PASSED [ 40%]
tests/test_pipeline_ventas.py::test_obtener_orden_exito PASSED           [ 60%]
tests/test_pipeline_ventas.py::test_obtener_orden_error_http PASSED      [ 80%]
tests/test_pipeline_ventas.py::test_obtener_orden_excepcion_red PASSED   [100%]

============================== 5 passed in 0.15s ===============================
```

---

### Paso 4: Implementar logging robusto y rotativo con loguru

**Objetivo:** Reemplazar los prints básicos con un sistema de logging estructurado que almacene información detallada en archivos y rote automáticamente los ficheros para evitar saturar el almacenamiento en producción.

#### Instrucciones

1. Reemplazaremos la gestión clásica de logs de Python con `loguru` para tener soporte nativo para logs jerárquicos y formato consistente.
2. Modifica el archivo `/home/usuario/workspace/automatizacion_ventas/src/pipeline_ventas.py` para configurarlo e importarlo correctamente. Abre el archivo y reescribe los encabezados e importaciones, configurando un logger que escriba en consola y almacene registros rotativos en `/home/usuario/workspace/automatizacion_ventas/logs/pipeline.log`.
3. Ajusta los métodos de la clase `ExtractorVentas` para que usen la interfaz de `loguru.logger` en lugar de la función nativa `print`.

A continuación, se detalla el código completo modificado para `/home/usuario/workspace/automatizacion_ventas/src/pipeline_ventas.py`:

```python
## /home/usuario/workspace/automatizacion_ventas/src/pipeline_ventas.py
import requests
import sys
from loguru import logger

## Configuración única del Logger de Loguru
## Limpiamos manejadores por defecto para evitar duplicación en consola
logger.remove()

## Manejador 1: Consola con formato legible y coloreado para desarrollo
logger.add(
    sys.stderr,
    format="<green>{time:YYYY-MM-DD HH:mm:ss.SSS}</green> | <level>{level: <8}</level> | <cyan>{module}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>",
    level="DEBUG"
)

## Manejador 2: Archivo rotativo estructurado para producción e histórico
logger.add(
    "/home/usuario/workspace/automatizacion_ventas/logs/pipeline.log",
    format="{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | {module}:{function}:{line} - {message}",
    level="INFO",
    rotation="10 MB",     # Rota el archivo si excede los 10 Megabytes
    retention="5 days",   # Conserva hasta un máximo de 5 días de logs antiguos
    compression="zip"     # Comprime los logs archivados automáticamente en formato .zip
)

class APIException(Exception):
    """Excepción personalizada para errores del servidor o respuestas no exitosas (HTTP != 200)."""
    pass

class ExtractorVentas:
    """Clase encargada de interactuar de forma síncrona con el endpoint de órdenes de venta."""
    
    def __init__(self, api_token: str, timeout: int = 5):
        if not api_token:
            logger.error("Se intentó instanciar ExtractorVentas con un token vacío o nulo.")
            raise ValueError("El token de API de entrada no puede estar vacío.")
        self.api_token = api_token
        self.timeout = timeout
        self.base_url = "https://api.ejemplo.com/v1/orders"
        logger.info("ExtractorVentas inicializado correctamente con timeout de {}s.", timeout)

    def obtener_orden_por_id(self, order_id: int) -> dict:
        """
        Realiza una petición síncrona GET a la API para extraer una sola orden utilizando logging estructurado.
        """
        if not isinstance(order_id, int) or order_id <= 0:
            logger.error("Tipo de dato inválido para el ID de la orden suministrado: {}", order_id)
            raise TypeError("El ID de la orden debe ser un entero positivo válido.")

        url = f"{self.base_url}/{order_id}"
        headers = {
            "Authorization": f"Bearer {self.api_token}",
            "Accept": "application/json"
        }

        logger.debug("Iniciando descarga HTTP GET para orden ID: {}", order_id)
        try:
            response = requests.get(url, headers=headers, timeout=self.timeout)
        except requests.RequestException as exc:
            logger.error("Error crítico de conectividad al recuperar orden {}. Excepción: {}", order_id, str(exc))
            raise APIException(f"Fallo crítico de conexión para orden {order_id}") from exc

        if response.status_code != 200:
            logger.warning("Fallo en API. Endpoint retornó status_code {} para ID: {}", response.status_code, order_id)
            raise APIException(f"La API retornó código HTTP {response.status_code} para orden {order_id}")

        logger.info("Orden {} descargada de manera exitosa desde la API.", order_id)
        return response.json()

    def procesar_lote_ordenes(self, lista_ids: list[int]) -> list[dict]:
        """
        Descarga secuencialmente un lote completo de IDs de órdenes con trazabilidad de errores.
        """
        logger.info("Iniciando procesamiento secuencial para un lote de {} órdenes.", len(lista_ids))
        resultados = []
        for order_id in lista_ids:
            try:
                datos_orden = self.obtener_orden_por_id(order_id)
                resultados.append(datos_orden)
            except APIException as e:
                logger.warning("Orden {} omitida debido a una falla controlada: {}", order_id, str(e))
        logger.info("Procesamiento de lote finalizado. Descargas exitosas: {}/{}", len(resultados), len(lista_ids))
        return resultados
```

#### Resultado Esperado

El código ahora no cuenta con instrucciones de impresión rústicas (`print`), sino con un sistema unificado y profesional de logs en consola y archivo que facilitará la observabilidad en ambientes operativos.

#### Verificación

Para corroborar que los tests siguen funcionando perfectamente (y que los mocks no entran en conflicto con la inicialización de loguru), ejecuta las pruebas de nuevo en tu terminal:
```bash
pytest -v tests/test_pipeline_ventas.py
```
*(Todos los tests deben reportar un estado PASSED).*

Adicionalmente, valida que se haya creado el archivo físico de logs en la ruta correspondiente:
```bash
ls -la /home/usuario/workspace/automatizacion_ventas/logs/
```
*(Debe listar el archivo `pipeline.log`)*.

---

### Paso 5: Refactorizar para extracción concurrente (ThreadPoolExecutor)

**Objetivo:** Modificar la clase `ExtractorVentas` para soportar la ejecución concurrente multihilo de la descarga de datos utilizando `concurrent.futures.ThreadPoolExecutor`, minimizando los tiempos muertos provocados por la latencia de red.

#### Instrucciones

1. Añade soporte en tu script base para realizar descargas en paralelo utilizando un hilo de procesamiento por petición HTTP (un patrón de diseño óptimo para cuellos de botella de tipo I/O Bound).
2. Abre tu archivo `/home/usuario/workspace/automatizacion_ventas/src/pipeline_ventas.py` y añade el módulo estándar de concurrencia:
   ```python
   from concurrent.futures import ThreadPoolExecutor, as_completed
   ```
3. Implementa un método concurrente de procesamiento de lotes en la clase `ExtractorVentas`. El método debe aceptar el parámetro de entrada `max_workers` para limitar el número de hilos de forma segura.
4. Integra la nueva lógica modificando `/home/usuario/workspace/automatizacion_ventas/src/pipeline_ventas.py`. Asegúrate de que tu clase coincida exactamente con la implementación final detallada abajo:

```python
## /home/usuario/workspace/automatizacion_ventas/src/pipeline_ventas.py
import requests
import sys
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from loguru import logger

## Configuración del Logger de Loguru
logger.remove()
logger.add(
    sys.stderr,
    format="<green>{time:YYYY-MM-DD HH:mm:ss.SSS}</green> | <level>{level: <8}</level> | <cyan>{module}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>",
    level="DEBUG"
)
logger.add(
    "/home/usuario/workspace/automatizacion_ventas/logs/pipeline.log",
    format="{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | {module}:{function}:{line} - {message}",
    level="INFO",
    rotation="10 MB",
    retention="5 days",
    compression="zip"
)

class APIException(Exception):
    """Excepción personalizada para errores del servidor o respuestas no exitosas (HTTP != 200)."""
    pass

class ExtractorVentas:
    """Clase encargada de interactuar de forma concurrente con el endpoint de órdenes de venta."""
    
    def __init__(self, api_token: str, timeout: int = 5):
        if not api_token:
            logger.error("Se intentó instanciar ExtractorVentas con un token vacío o nulo.")
            raise ValueError("El token de API de entrada no puede estar vacío.")
        self.api_token = api_token
        self.timeout = timeout
        self.base_url = "https://api.ejemplo.com/v1/orders"
        logger.info("ExtractorVentas inicializado correctamente con timeout de {}s.", timeout)

    def obtener_orden_por_id(self, order_id: int) -> dict:
        """
        Realiza una petición síncrona GET a la API para extraer una sola orden.
        """
        if not isinstance(order_id, int) or order_id <= 0:
            logger.error("Tipo de dato inválido para el ID de la orden suministrado: {}", order_id)
            raise TypeError("El ID de la orden debe ser un entero positivo válido.")

        url = f"{self.base_url}/{order_id}"
        headers = {
            "Authorization": f"Bearer {self.api_token}",
            "Accept": "application/json"
        }

        logger.debug("Iniciando descarga HTTP GET para orden ID: {}", order_id)
        try:
            response = requests.get(url, headers=headers, timeout=self.timeout)
        except requests.RequestException as exc:
            logger.error("Error crítico de conectividad al recuperar orden {}. Excepción: {}", order_id, str(exc))
            raise APIException(f"Fallo crítico de conexión para orden {order_id}") from exc

        if response.status_code != 200:
            logger.warning("Fallo en API. Endpoint retornó status_code {} para ID: {}", response.status_code, order_id)
            raise APIException(f"La API retornó código HTTP {response.status_code} para orden {order_id}")

        logger.info("Orden {} descargada de manera exitosa desde la API.", order_id)
        return response.json()

    def procesar_lote_ordenes(self, lista_ids: list[int]) -> list[dict]:
        """
        Descarga secuencialmente un lote completo de IDs de órdenes con trazabilidad de errores.
        """
        logger.info("Iniciando procesamiento secuencial para un lote de {} órdenes.", len(lista_ids))
        resultados = []
        for order_id in lista_ids:
            try:
                datos_orden = self.obtener_orden_por_id(order_id)
                resultados.append(datos_orden)
            except APIException as e:
                logger.warning("Orden {} omitida debido a una falla controlada: {}", order_id, str(e))
        logger.info("Procesamiento de lote finalizado. Descargas exitosas: {}/{}", len(resultados), len(lista_ids))
        return resultados

    def procesar_lote_concurrente(self, lista_ids: list[int], max_workers: int = 4) -> list[dict]:
        """
        Descarga un lote de órdenes en paralelo utilizando un ThreadPoolExecutor para optimizar
        los tiempos muertos de latencia de red.
        """
        logger.info("Iniciando procesamiento CONCURRENTE de {} órdenes utilizando {} hilos de ejecución.", len(lista_ids), max_workers)
        resultados = []
        
        tiempo_inicio = time.perf_counter()
        
        # Uso seguro de ThreadPoolExecutor mediante un gestor de contexto
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            # Creamos un mapeo para asociar cada objeto Future con su respectivo ID de orden
            mapeo_futuros = {
                executor.submit(self.obtener_orden_por_id, order_id): order_id 
                for order_id in lista_ids
            }
            
            for futuro in as_completed(mapeo_futuros):
                id_orden = mapeo_futuros[futuro]
                try:
                    datos_orden = futuro.result()
                    resultados.append(datos_orden)
                except APIException as e:
                    logger.warning("La orden concurrente {} falló al procesarse: {}", id_orden, str(e))
                except Exception as e:
                    logger.critical("Error no controlado en el hilo de procesamiento de la orden {}: {}", id_orden, str(e))

        tiempo_total = time.perf_counter() - tiempo_inicio
        logger.info("Procesamiento concurrente finalizado en {:.4f} segundos. Éxitos: {}/{}", tiempo_total, len(resultados), len(lista_ids))
        return resultados
```

5. **Prueba unitaria para el flujo concurrente:** Abre el archivo de pruebas en `/home/usuario/workspace/automatizacion_ventas/tests/test_pipeline_ventas.py` y añade un nuevo caso de prueba al final para verificar que la descarga concurrente ejecute correctamente el *mapping* de hilos sin corromper el conjunto de resultados:

```python
## Añadir al final de /home/usuario/workspace/automatizacion_ventas/tests/test_pipeline_ventas.py

@patch("src.pipeline_ventas.requests.get")
def test_procesar_lote_concurrente_exito_y_error_mixto(mock_get, extractor_valido):
    """
    Escenario de prueba de integración de flujo concurrente.
    Valida la orquestación de hilos ante un lote que contiene llamadas exitosas y fallidas simultáneamente.
    """
    # Arrange
    # Configuramos respuestas dinámicas para requests.get basadas en el ID de la orden solicitado
    def side_effect_dinamico(url, *args, **kwargs):
        mock_response = MagicMock()
        if "100" in url:
            mock_response.status_code = 200
            mock_response.json.return_value = {"id": 100, "monto": 500.0}
        elif "200" in url:
            mock_response.status_code = 200
            mock_response.json.return_value = {"id": 200, "monto": 750.0}
        else:
            mock_response.status_code = 404  # Lanzará error
        return mock_response

    mock_get.side_effect = side_effect_dinamico
    lista_de_prueba = [100, 200, 999]  # 2 éxitos y 1 fallo esperado

    # Act
    resultados = extractor_valido.procesar_lote_concurrente(lista_de_prueba, max_workers=2)

    # Assert
    # Debe haber devuelto los datos de las dos peticiones exitosas únicamente
    assert len(resultados) == 2
    ids_descargados = {r["id"] for r in resultados}
    assert ids_descargados == {100, 200}
```

#### Resultado Esperado

Tu suite de pruebas contiene un caso avanzado de aserción para verificar la seguridad ante hilos (*thread-safety*) de los pipelines concurrentes y la tolerancia ante respuestas mixtas concurrentes de la API.

#### Verificación

Ejecuta la suite completa de pruebas unitarias y de concurrencia:
```bash
pytest -v tests/test_pipeline_ventas.py
```

La consola debe mostrar 6 pruebas superadas sin ningún tipo de error o desbordamiento en el procesamiento asíncrono:
```text
============================= test session starts ==============================
collected 6 items

tests/test_pipeline_ventas.py::test_inicializacion_extractor_invalido PASSED [ 16%]
tests/test_pipeline_ventas.py::test_obtener_orden_tipo_dato_invalido PASSED [ 33%]
tests/test_pipeline_ventas.py::test_obtener_orden_exito PASSED           [ 50%]
tests/test_pipeline_ventas.py::test_obtener_orden_error_http PASSED      [ 66%]
tests/test_pipeline_ventas.py::test_obtener_orden_excepcion_red PASSED   [ 83%]
tests/test_pipeline_ventas.py::test_procesar_lote_concurrente_exito_y_error_mixto PASSED [100%]

============================== 6 passed in 0.18s ===============================
```

---

## Validación y Pruebas

Para garantizar que tu solución es lo suficientemente robusta y cumple con los contratos técnicos definidos para el desarrollo de software de nivel industrial, completaremos una validación de resiliencia y analizaremos un caso de prueba adversario en el motor de ejecución.

### Caso de Prueba Adversario: Payload JSON Corrupto (Prueba Robustez)

¿Qué sucede si un servidor de API de un tercero devuelve un código de estado `HTTP 200` pero su contenido no es un JSON válido (sino un string corrupto o formato HTML inesperado por un fallo en el proxy inverso)? Nuestra lógica actual fallaría al llamar a `response.json()`. 

#### Instrucciones

1. Añadiremos una prueba unitaria adversarial que verifique la resiliencia y el comportamiento predecible del sistema ante respuestas corruptas. Modifica tu archivo `tests/test_pipeline_ventas.py` y agrega la siguiente prueba al final del mismo:

```python
## Añadir al final de /home/usuario/workspace/automatizacion_ventas/tests/test_pipeline_ventas.py

@patch("src.pipeline_ventas.requests.get")
def test_obtener_orden_json_corrupto(mock_get, extractor_valido):
    """
    Caso de Prueba Adversario.
    Valida el comportamiento cuando el servidor devuelve un HTTP 200 exitoso pero con un
    cuerpo JSON mal formado o corrupto. Debe lanzar APIException para evitar propagación silenciosa de datos vacíos.
    """
    # Arrange
    mock_response = MagicMock()
    mock_response.status_code = 200
    # Simulamos que response.json() lanza una excepción nativa de Python al intentar parsear strings corruptos
    mock_response.json.side_effect = ValueError("No se pudo decodificar JSON")
    mock_get.return_value = mock_response

    # Act & Assert
    with pytest.raises(APIException) as exc_info:
        extractor_valido.obtener_orden_por_id(505)

    # El flujo debe de haber interceptado la corrupción y encapsularla adecuadamente
    assert "Fallo crítico de conexión para orden 505" in str(exc_info.value) or "Error crítico" in str(exc_info.value)
```

2. Ejecuta la modificación en el pipeline (`src/pipeline_ventas.py`) para capturar fallos de parseo de JSON en el método `obtener_orden_por_id`:

```python
        # Ubicar sección final del método obtener_orden_por_id en src/pipeline_ventas.py
        # ...
        if response.status_code != 200:
            logger.warning("Fallo en API. Endpoint retornó status_code {} para ID: {}", response.status_code, order_id)
            raise APIException(f"La API retornó código HTTP {response.status_code} para orden {order_id}")

        try:
            datos_json = response.json()
        except ValueError as exc:
            logger.error("La API retornó un contenido corrupto no serializable como JSON para la orden {}. Error: {}", order_id, str(exc))
            raise APIException(f"Fallo crítico de formato JSON en orden {order_id}") from exc

        logger.info("Orden {} descargada de manera exitosa desde la API.", order_id)
        return datos_json
```

3. Ejecuta la suite de pruebas consolidada para verificar la respuesta del sistema ante fallos de corrupción de datos:
   ```bash
   pytest -v tests/test_pipeline_ventas.py
   ```

**Salida de consola esperada:**
```text
============================= test session starts ==============================
collected 7 items

tests/test_pipeline_ventas.py::test_inicializacion_extractor_invalido PASSED [ 14%]
tests/test_pipeline_ventas.py::test_obtener_orden_tipo_dato_invalido PASSED [ 28%]
tests/test_pipeline_ventas.py::test_obtener_orden_exito PASSED           [ 42%]
tests/test_pipeline_ventas.py::test_obtener_orden_error_http PASSED      [ 57%]
tests/test_pipeline_ventas.py::test_obtener_orden_excepcion_red PASSED   [ 71%]
tests/test_pipeline_ventas.py::test_procesar_lote_concurrente_exito_y_error_mixto PASSED [ 85%]
tests/test_pipeline_ventas.py::test_obtener_orden_json_corrupto PASSED   [100%]

============================== 7 passed in 0.22s ===============================
```

### Verificación del Archivo de Logs Físico

Asegúrate de que la automatización haya guardado la información de la corrida en tu máquina local:
```bash
cat /home/usuario/workspace/automatizacion_ventas/logs/pipeline.log
```

**Muestra del formato del archivo de logs esperado:**
```text
2024-03-20 14:00:01 | INFO     | pipeline_ventas:__init__:35 - ExtractorVentas inicializado correctamente con timeout de 3s.
2024-03-20 14:00:01 | DEBUG    | pipeline_ventas:obtener_orden_por_id:48 - Iniciando descarga HTTP GET para orden ID: 101
2024-03-20 14:00:01 | INFO     | pipeline_ventas:obtener_orden_por_id:63 - Orden 101 descargada de manera exitosa desde la API.
2024-03-20 14:00:01 | ERROR    | pipeline_ventas:obtener_orden_por_id:52 - Error crítico de conectividad al recuperar orden 102. Excepción: El servidor tardó demasiado en responder
```

---

## Solución de Problemas

### Problema 1: `ModuleNotFoundError: No module named 'src'` al ejecutar pytest

- **Síntomas:** Al ejecutar `pytest tests/test_pipeline_ventas.py`, obtienes un error que indica que el módulo `src` o `pipeline_ventas` no puede ser importado o localizado en el sistema de archivos.
- **Causa:** El directorio raíz del proyecto no está registrado en la variable de entorno `PYTHONPATH` de Python en tu sesión de terminal actual, lo que impide que el buscador de módulos localice la carpeta `src`.
- **Solución:** Ejecuta las pruebas exportando la ruta de búsqueda manualmente antes de invocar la suite de testeo:
  ```bash
  export PYTHONPATH="${PYTHONPATH}:/home/usuario/workspace/automatizacion_ventas"
  pytest -v tests/test_pipeline_ventas.py
  ```
  O alternativamente, ejecuta `pytest` utilizando el módulo de ejecución de Python en su lugar, el cual agrega el directorio actual por defecto al `sys.path`:
  ```bash
  python -m pytest -v tests/test_pipeline_ventas.py
  ```

### Problema 2: El archivo `pipeline.log` está vacío o no se genera en el directorio `logs`

- **Síntomas:** La consola imprime los logs coloreados de forma correcta pero no se crea ningún archivo dentro de `/home/usuario/workspace/automatizacion_ventas/logs/`.
- **Causa:** Problemas de permisos de escritura de Linux sobre el directorio de logs seleccionado o discrepancia en el nivel del log configurado. `loguru` descarta silenciosamente los mensajes si el nivel jerárquico es inferior al establecido (ej. estás enviando un log en nivel `DEBUG` pero configuraste el log en archivo para capturar solo desde `INFO` en adelante).
- **Solución:**
  1. Verifica que tu usuario tenga permisos de escritura sobre la carpeta ejecutando:
     ```bash
     chmod -R 755 /home/usuario/workspace/automatizacion_ventas/logs
     ```
  2. Revisa que el código contenga el nivel correcto `logger.add("/home/usuario/workspace/automatizacion_ventas/logs/pipeline.log", level="INFO")` y que en tu código de extracción estés enviando logs utilizando la función apropiada `logger.info(...)` o `logger.warning(...)`.

---

## Limpieza

Para restaurar tu entorno de desarrollo y evitar dejar archivos temporales u objetos huérfanos en tu estación de trabajo, ejecuta las siguientes instrucciones de limpieza en tu terminal:

1. Desactiva el entorno virtual activo:
   ```bash
   deactivate
   ```
2. Elimina los directorios de cache temporales generados automáticamente por `pytest` y por el compilador de Python en tiempo de ejecución:
   ```bash
   cd /home/usuario/workspace/automatizacion_ventas
   rm -rf .pytest_cache
   rm -rf src/__pycache__
   rm -rf tests/__pycache__
   ```
3. Si deseas eliminar el entorno virtual de forma definitiva para ahorrar almacenamiento físico en tu disco:
   ```bash
   rm -rf .venv
   ```

---

## Resumen

¡Felicidades! Has completado exitosamente la transición de un script síncrono frágil hacia un componente de automatización moderno, seguro y optimizado bajo un esquema de testing profesional de nivel empresarial.

### Logros Clave

1. **Aislamiento de Código mediante Mocks:** Aprendiste a estructurar pruebas unitarias con `pytest` y `unittest.mock` para simular condiciones adversas de red sin realizar llamadas reales, disminuyendo la fragilidad del pipeline.
2. **Concurrencia Multi-hilo Eficiente:** Redujiste considerablemente el tiempo total de procesamiento de lotes de llamadas mediante el uso de `concurrent.futures.ThreadPoolExecutor`, optimizando el hardware disponible.
3. **Observabilidad para Producción con Loguru:** Reemplazaste los prints simples de Python con un sistema de trazabilidad estructurado con formatos unificados, rotación física de logs por peso e histórico controlado para soporte en vivo.

### Recursos Adicionales

- [Documentación oficial del framework pytest](https://docs.pytest.org/)
- [Módulo estándar concurrent.futures de Python](https://docs.python.org/3/library/concurrent.futures.html)
- [Repositorio y Guías de Uso de Loguru](https://github.com/Delgan/loguru)
