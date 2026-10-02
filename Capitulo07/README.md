# Práctica: Productividad del desarrollo asistida por IA

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 120 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio práctico, asumirás el rol de un Lead Developer encargado de auditar, documentar y optimizar una suite de automatización de ventas previamente desarrollada. Utilizando la extensión de **GitHub Copilot** y **Copilot Chat** en **VS Code**, refactorizarás consultas SQL de inserción en PostgreSQL para convertirlas en operaciones masivas de tipo *UPSERT* altamente optimizadas. Adicionalmente, diseñarás una suite de pruebas unitarias robusta empleando **pytest** para validar un planificador de tareas **APScheduler** y estructurarás la documentación completa del proyecto bajo el estándar OpenAPI y Google Style Docstrings, todo asistido por avanzadas técnicas de ingeniería de prompts.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
* [ ] **Refactorizar** consultas SQL tradicionales en PostgreSQL hacia operaciones masivas *UPSERT* (idempotentes) integrando sugerencias contextuales de GitHub Copilot.
* [ ] **Diseñar y estructurar** documentación técnica estandarizada (README.md exhaustivo y Docstrings en formato Google Style) mediante el uso de prompts dirigidos en Copilot Chat.
* [ ] **Construir** una suite de pruebas unitarias con `pytest` y mocks de tiempo para un planificador `APScheduler` guiando la generación de código mediante restricciones estrictas de prompt engineering.
* [ ] **Implementar** validaciones de salud y robustez de contenedores (healthchecks) en entornos multi-contenedor de Docker Compose asistido por IA.

## Prerrequisitos

Para completar con éxito este laboratorio, necesitarás:
1. **Conocimientos teóricos y prácticos:**
   * Fundamentos de base de datos relacionales (PostgreSQL), comandos SQL e índices.
   * Comprensión intermedia del lenguaje Python 3.12.2 y la biblioteca `pytest`.
   * Conceptos básicos de contenedorización y orquestación con Docker Compose.
2. **Accesos y licencias activas:**
   * Suscripción activa a **GitHub Copilot** (Licencia *Copilot Individual*, *Business* o *Enterprise*).
   * Cuenta de GitHub conectada y activa dentro del entorno de desarrollo.
3. **Código base del laboratorio anterior:**
   * Se asume que cuentas con la base del código de automatización (desarrollada idealmente en el Laboratorio 06-00-01) que incluye un planificador APScheduler y un cargador de base de datos PostgreSQL. En caso contrario, este laboratorio proveerá el código inicial para ejecutar la práctica de forma aislada.

## Entorno de Laboratorio

El laboratorio debe ejecutarse bajo las siguientes especificaciones de hardware y software:

### Requisitos de Hardware

| Componente | Requisito Mínimo | Requisito Recomendado |
| :--- | :--- | :--- |
| **Memoria RAM** | 8 GB | 16 GB |
| **Procesador** | Intel i5 (4 núcleos físicos) o AMD equivalente (Soporte VT-x/AMD-V en BIOS) | Intel i7 / Apple Silicon M1 o superior |
| **Almacenamiento** | 20 GB de espacio libre (SSD preferiblemente) | 40 GB de espacio libre (SSD NVMe) |
| **Conexión de Red** | Banda ancha sin restricciones de puertos salientes | Banda ancha sin restricciones de puertos salientes |

### Requisitos de Software

| Herramienta / Tecnología | Versión Exacta | Enlace de Descarga / Referencia |
| :--- | :--- | :--- |
| **Visual Studio Code** | 1.87.2 | [Descarga VS Code 1.87](https://code.visualstudio.com/updates/v1_87) |
| **Extensión GitHub Copilot** | 1.173.0 | [Extensión en VS Marketplace](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) |
| **Extensión Copilot Chat** | 0.13.0 | [Extensión en VS Marketplace](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat) |
| **Python** | 3.12.2 | [Python 3.12.2 Download](https://www.python.org/downloads/release/python-3122/) |
| **Docker Engine / Desktop** | 25.0.3 | [Docker Desktop Release Notes](https://docs.docker.com/desktop/release-notes/#2503) |
| **Docker Compose** | 2.24.5 | [Docker Compose Releases](https://github.com/docker/compose/releases/tag/v2.24.5) |
| **PostgreSQL (Imagen Docker)**| 16.2-alpine | [PostgreSQL en Docker Hub](https://hub.docker.com/r/_/postgres) |
| **pytest** | 8.1.1 | [Documentación de pytest](https://docs.pytest.org/en/8.1.x/) |
| **APScheduler** | 3.10.4 | [Documentación de APScheduler](https://apscheduler.readthedocs.io/en/3.10.4/) |

### Configuración del Directorio de Trabajo

Toda la práctica se desarrollará en el siguiente directorio local:
```bash
/home/usuario/workspace/automatizacion_ventas
```

Ejecuta el siguiente comando para validar la estructura base o inicializarla si es necesario:
```bash
mkdir -p /home/usuario/workspace/automatizacion_ventas/src
mkdir -p /home/usuario/workspace/automatizacion_ventas/tests
mkdir -p /home/usuario/workspace/automatizacion_ventas/data/validated
```

---

## Instrucciones Paso a Paso

### Paso 1: Configuración del Código Base e Inicio de Auditoría de IA

**Objective**  
Establecer el código inicial de extracción, carga e hilos de programación en el espacio de trabajo local, y realizar un análisis automatizado de rendimiento y seguridad utilizando los comandos de **GitHub Copilot Chat** en VS Code.

**Instructions**  

1. Crea el archivo base de carga de datos en `/home/usuario/workspace/automatizacion_ventas/src/db_loader.py` con el siguiente código secuencial e ineficiente (el cual emula una versión inicial de desarrollo):

```python
import os
import psycopg2
from typing import List, Dict, Any

class DatabaseLoader:
    def __init__(self):
        self.conn = psycopg2.connect(
            host=os.getenv("DB_HOST", "localhost"),
            database=os.getenv("DB_NAME", "automation_db"),
            user=os.getenv("DB_USER", "postgres_user"),
            password=os.getenv("DB_PASSWORD", "secure_password_123"),
            port=os.getenv("DB_PORT", "5432")
        )
        self.cursor = self.conn.cursor()

    def create_table(self):
        # Genera tabla sin índices ni restricciones complejas
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS ventas (
                id VARCHAR(255) PRIMARY KEY,
                monto DOUBLE PRECISION,
                moneda VARCHAR(3),
                fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );
        """)
        self.conn.commit()

    def insertar_ventas(self, transacciones: List[Dict[str, Any]]):
        # Proceso secuencial e ineficiente que causa cuellos de botella
        for tx in transacciones:
            try:
                self.cursor.execute("""
                    INSERT INTO ventas (id, monto, moneda)
                    VALUES (%s, %s, %s);
                """, (tx['id'], tx['monto'], tx['moneda']))
            except Exception as e:
                print(f"Error insertando registro: {e}")
                self.conn.rollback()
            else:
                self.conn.commit()

    def cerrar_conexion(self):
        self.cursor.close()
        self.conn.close()
```

2. Abre **Visual Studio Code** en el directorio `/home/usuario/workspace/automatizacion_ventas`.
3. Abre el archivo `/home/usuario/workspace/automatizacion_ventas/src/db_loader.py`.
4. Abre el panel de **GitHub Copilot Chat** (`Ctrl + Shift + I` o haciendo clic en el icono de chat de la barra lateral de VS Code).
5. Envía el siguiente prompt estructurado en la ventana de chat para auditar el código existente:

> **Prompt de Auditoría:**  
> `@workspace /explain Actúa como un Arquitecto de Base de Datos y Especialista en Seguridad en Python. Analiza el archivo 'src/db_loader.py'. Identifica vulnerabilidades de seguridad, ineficiencias de rendimiento debido al procesamiento secuencial de registros, y la falta de robustez transaccional. Propón una estrategia conceptual para mejorar esto usando transacciones en bloque (batching) e inserciones idempotentes (UPSERT).`

**Expected output**  
El panel de **Copilot Chat** debe responder con un desglose estructurado en Markdown, resaltando:
* La ineficiencia del bucle `for` que realiza transacciones individuales por cada registro (`commit` y `rollback` en cada iteración).
* El riesgo latente de caídas si ocurren registros duplicados (violación de clave primaria sin control).
* La falta de uso de variables de entorno seguras e inicializaciones robustas.
* La propuesta técnica de usar `executemany` o la cláusula `ON CONFLICT DO UPDATE` (UPSERT) nativa de PostgreSQL.

**Verification**  
Verifica que la sugerencia de Copilot Chat comprenda la diferencia entre transacciones individuales y por lotes (*batching*). Deberás ver una respuesta explicativa similar a la siguiente:

```text
1. Rendimiento Deficiente: El código realiza un commit individual por cada fila, lo que incrementa drásticamente la latencia de red de base de datos.
2. Inestabilidad en Duplicados: El intento de inserción de una clave existente detiene o cancela transacciones.
3. Propuesta: Implementar una consulta masiva usando 'ON CONFLICT (id) DO UPDATE...' y el método 'execute_values' de psycopg2.extras para inserciones por lote.
```

---

### Paso 2: Refactorización a UPSERT Masivo Optimizado Asistido por IA

**Objective**  
Utilizar la autogeneración contextual e inline de GitHub Copilot para refactorizar la inserción secuencial de datos en un proceso de *UPSERT* (idempotente) y en bloque (*bulk insert*) altamente performante.

**Instructions**  

1. En el archivo `src/db_loader.py`, elimina por completo el método `insertar_ventas`.
2. Escribe el siguiente comentario de guía (Inline Prompt) justo debajo del método `create_table` para indicar a Copilot lo que deseas generar de forma explícita:

```python
    # Refactorizado asistido por IA: Inserción masiva de tipo UPSERT utilizando psycopg2.extras.execute_values
    # Debe ser idempotente con ON CONFLICT (id) DO UPDATE SET monto = EXCLUDED.monto, moneda = EXCLUDED.moneda
    # Utilizar logging adecuado en lugar de sentencias print de depuración.
    # Recibe una lista de diccionarios, maneja excepciones de base de datos específicas de psycopg2.
```

3. Presiona `Enter` y espera a que GitHub Copilot te sugiera el código inline. Si no se activa automáticamente, presiona `Ctrl + Space` (o `Cmd + Space` en macOS) para forzar la sugerencia. Presiona `Tab` para aceptar la sugerencia completa que se alinee con el estándar requerido.
4. Completa la implementación del cargador asegurando que se utilice `psycopg2.extras.execute_values` para mejorar la velocidad de carga de datos masivos. El código generado asistido por Copilot debe quedar estructurado de la siguiente forma:

```python
import os
import logging
from typing import List, Dict, Any
import psycopg2
from psycopg2.extras import execute_values
from psycopg2 import DatabaseError

## Configuración de registro estructurado de eventos
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - [%(filename)s:%(lineno)d] - %(message)s"
)
logger = logging.getLogger(__name__)

class DatabaseLoader:
    """
    Clase encargada de orquestar la conexión y carga idempotente 
    de transacciones financieras en PostgreSQL.
    """
    def __init__(self) -> None:
        try:
            self.conn = psycopg2.connect(
                host=os.getenv("DB_HOST", "localhost"),
                database=os.getenv("DB_NAME", "automation_db"),
                user=os.getenv("DB_USER", "postgres_user"),
                password=os.getenv("DB_PASSWORD", "secure_password_123"),
                port=os.getenv("DB_PORT", "5432"),
                connect_timeout=5
            )
            self.cursor = self.conn.cursor()
            logger.info("Conexión con PostgreSQL establecida de manera exitosa.")
        except DatabaseError as e:
            logger.error(f"Error crítico al conectar a la base de datos: {e}")
            raise

    def create_table(self) -> None:
        """Crea la tabla ventas y añade restricciones básicas."""
        try:
            self.cursor.execute("""
                CREATE TABLE IF NOT EXISTS ventas (
                    id VARCHAR(255) PRIMARY KEY,
                    monto DOUBLE PRECISION NOT NULL,
                    moneda VARCHAR(3) NOT NULL,
                    fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                );
            """)
            self.conn.commit()
            logger.info("Tabla 'ventas' verificada/creada exitosamente.")
        except DatabaseError as e:
            self.conn.rollback()
            logger.error(f"Error al estructurar tabla: {e}")
            raise

    def insertar_ventas_masivo(self, transacciones: List[Dict[str, Any]]) -> None:
        """
        Inserta un lote de transacciones utilizando el patrón UPSERT masivo con execute_values.
        
        Args:
            transacciones: Lista de diccionarios que contienen 'id', 'monto' y 'moneda'.
        """
        if not transacciones:
            logger.warning("Lote de transacciones vacío. Omitiendo proceso de base de datos.")
            return

        query = """
            INSERT INTO ventas (id, monto, moneda)
            VALUES %s
            ON CONFLICT (id) DO UPDATE SET
                monto = EXCLUDED.monto,
                moneda = EXCLUDED.moneda,
                fecha = CURRENT_TIMESTAMP;
        """
        
        # Transformar lista de diccionarios a lista de tuplas para execute_values
        valores = [(tx['id'], tx['monto'], tx['moneda']) for tx in transacciones]

        try:
            execute_values(self.cursor, query, valores)
            self.conn.commit()
            logger.info(f"Procesamiento masivo completado. Registros procesados/actualizados: {len(transacciones)}")
        except DatabaseError as e:
            self.conn.rollback()
            logger.error(f"Fallo masivo en la transacción. Rollback ejecutado. Error: {e}")
            raise

    def cerrar_conexion(self) -> None:
        """Libera de manera segura los recursos de cursor y conexión."""
        if self.cursor:
            self.cursor.close()
        if self.conn:
            self.conn.close()
        logger.info("Recursos de base de datos liberados correctamente.")
```

**Expected output**  
Un archivo Python `src/db_loader.py` limpio, que utiliza métodos por lotes e incorpora manejo robusto de excepciones de `psycopg2` y el uso del módulo estándar `logging` en sustitución de `print`.

**Verification**  
Analiza sintácticamente el archivo ejecutando:
```bash
python -m py_compile src/db_loader.py
```
No deben registrarse errores de compilación sintáctica en el terminal.

---

### Paso 3: Documentación Completa del Proyecto bajo el Estándar Google Style y README.md

**Objective**  
Generar los docstrings estructurados de las clases y un documento global `README.md` detallado de la automatización para cumplir con las políticas de gobernanza de software técnico, haciendo uso directo de prompts de documentación en **Copilot Chat**.

**Instructions**  

1. Crea primero la clase planificadora del servicio en el archivo `/home/usuario/workspace/automatizacion_ventas/src/scheduler.py` con el siguiente código base inicial:

```python
import os
import time
import logging
from apscheduler.schedulers.background import BackgroundScheduler
from src.db_loader import DatabaseLoader

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class SalesAutomationScheduler:
    def __init__(self):
        self.scheduler = BackgroundScheduler()
        self.loader = None

    def ejecutar_tarea_extraccion(self):
        logger.info("Iniciando ciclo automático de extracción y carga...")
        # Datos simulados de entrada
        datos_ejemplo = [
            {"id": "tx_001", "monto": 1500.50, "moneda": "USD"},
            {"id": "tx_002", "monto": 450.00, "moneda": "EUR"},
            {"id": "tx_003", "monto": 99.99, "moneda": "USD"}
        ]
        try:
            self.loader = DatabaseLoader()
            self.loader.create_table()
            self.loader.insertar_ventas_masivo(datos_ejemplo)
        except Exception as e:
            logger.error(f"Fallo en la tarea programada: {e}")
        finally:
            if self.loader:
                self.loader.cerrar_conexion()

    def iniciar(self):
        self.scheduler.add_job(self.ejecutar_tarea_extraccion, 'interval', seconds=30, id='job_extraccion')
        self.scheduler.start()
        logger.info("Programador APScheduler inicializado y corriendo.")

    def detener(self):
        self.scheduler.shutdown()
        logger.info("Programador APScheduler detenido.")
```

2. Selecciona todo el contenido del archivo `src/scheduler.py`.
3. Abre **Copilot Chat** e introduce el siguiente prompt diseñado para la generación de documentación en base a guías de estilo rigurosas:

> **Prompt de Documentación de Código:**  
> `@workspace /doc Documenta todo el código de 'src/scheduler.py' siguiendo rigurosamente el formato de Docstring de Google (Google Style Python Docstrings). Incluye anotaciones de tipo ('Type Hints') para todos los parámetros y retornos de funciones. Asegura que cada clase y método describa claramente su propósito, argumentos, excepciones lanzadas y efectos colaterales.`

4. Copia el código documentado sugerido por Copilot y reemplaza el contenido de `src/scheduler.py`. El archivo resultante debe lucir con la siguiente estructura limpia:

```python
import os
import time
import logging
from typing import Optional
from apscheduler.schedulers.background import BackgroundScheduler
from src.db_loader import DatabaseLoader

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger(__name__)

class SalesAutomationScheduler:
    """
    Controla y orquesta la ejecución periódica de tareas de extracción de datos de ventas.

    Esta clase encapsula la instancia de un programador en segundo plano (BackgroundScheduler)
    y gestiona el ciclo de vida del flujo ETL de ventas hacia la base de datos.

    Attributes:
        scheduler (BackgroundScheduler): Instancia del planificador APScheduler.
        loader (Optional[DatabaseLoader]): Cargador de base de datos inicializado durante la ejecución.
    """
    def __init__(self) -> None:
        """Inicializa el programador y establece el estado inicial del cargador."""
        self.scheduler: BackgroundScheduler = BackgroundScheduler()
        self.loader: Optional[DatabaseLoader] = None

    def ejecutar_tarea_extraccion(self) -> None:
        """
        Ejecuta el proceso cíclico de generación de datos ficticios y carga en base de datos.

        Este método es invocado por el disparador (trigger) del planificador. Realiza la
        conexión, la creación de estructuras de tabla de manera idempotente y la inserción 
        masiva de la información simulada.

        Raises:
            Exception: Captura y registra de forma segura cualquier excepción ocurrida en el proceso.
        """
        logger.info("Iniciando ciclo automático de extracción y carga...")
        datos_ejemplo = [
            {"id": "tx_001", "monto": 1500.50, "moneda": "USD"},
            {"id": "tx_002", "monto": 450.00, "moneda": "EUR"},
            {"id": "tx_003", "monto": 99.99, "moneda": "USD"}
        ]
        try:
            self.loader = DatabaseLoader()
            self.loader.create_table()
            self.loader.insertar_ventas_masivo(datos_ejemplo)
        except Exception as e:
            logger.critical(f"Fallo crítico en la tarea programada: {e}", exc_info=True)
        finally:
            if self.loader:
                try:
                    self.loader.cerrar_conexion()
                except Exception as close_err:
                    logger.error(f"Error al cerrar la conexión de base de datos: {close_err}")

    def iniciar(self) -> None:
        """
        Registra la tarea de extracción con un intervalo fijo de ejecución e inicia el hilo.

        La tarea se configura para ejecutarse de forma recurrente cada 30 segundos.
        """
        self.scheduler.add_job(
            self.ejecutar_tarea_extraccion, 
            'interval', 
            seconds=30, 
            id='job_extraccion',
            replace_existing=True
        )
        self.scheduler.start()
        logger.info("Programador APScheduler inicializado y corriendo con éxito.")

    def detener(self) -> None:
        """Detiene el hilo del programador liberando todos los recursos asociados."""
        if self.scheduler.running:
            self.scheduler.shutdown()
            logger.info("Programador APScheduler detenido de forma controlada.")
```

5. Generar ahora la documentación general del proyecto. Abre un archivo en blanco llamado `/home/usuario/workspace/automatizacion_ventas/README.md`.
6. Selecciona el archivo vacío e ingresa el siguiente prompt interactivo en **Copilot Chat** para construir la estructura organizativa del proyecto:

> **Prompt para README.md Técnico:**  
> `Actúa como un Escritor Técnico Senior de Software. Genera un archivo README.md completo para nuestro proyecto ubicado en '/home/usuario/workspace/automatizacion_ventas'. El README debe estructurarse con las siguientes secciones en español: Introducción, Arquitectura de Automatización, Requisitos del Sistema (mencionando Python 3.12.2, PostgreSQL 16.2, APScheduler 3.10.4, Docker Compose 2.24.5), Configuración de Variables de Entorno, Guía de Despliegue con Docker y Pruebas Unitarias. No uses marcadores de posición genéricos (placeholders), completa todas las instrucciones con rutas reales del proyecto.`

7. Copia la salida del chat y pégala en el archivo `README.md`.

**Expected output**  
El archivo `README.md` estructurado correctamente con explicaciones detalladas y sin placeholders, conteniendo instrucciones precisas para correr el proyecto.

**Verification**  
En VS Code, presiona `Ctrl + Shift + V` para previsualizar el archivo `README.md` en formato renderizado y confirma que cuenta con todos los encabezados requeridos.

---

### Paso 4: Creación de la Suite de Pruebas para APScheduler con Ingeniería de Prompts

**Objective**  
Diseñar y escribir una suite de pruebas unitarias robustas en `/home/usuario/workspace/automatizacion_ventas/tests/test_scheduler.py` con `pytest` para el planificador de tareas de automatización, utilizando mocks para simular las dependencias de base de datos.

**Instructions**  

1. Crea un nuevo archivo en el espacio de trabajo con la ruta `/home/usuario/workspace/automatizacion_ventas/tests/test_scheduler.py`.
2. Para asegurar un código de pruebas de alta calidad sin dependencias innecesarias, usaremos un **Prompt Estructurado** en Copilot Chat para diseñar el script de pruebas unitarias:

> **Prompt de Pruebas Unitarias:**  
> `Actúa como un QA Automation Engineer Senior experto en Python y pytest 8.1.1. Escribe el contenido del archivo 'tests/test_scheduler.py' para probar la clase 'SalesAutomationScheduler' de 'src.scheduler'. Requisitos técnicos obligatorios:  
> 1. Utiliza pytest fixtures.  
> 2. Mockea por completo la dependencia 'DatabaseLoader' y sus métodos ('create_table', 'insertar_ventas_masivo', 'cerrar_conexion') usando unittest.mock de forma que no requiera una base de datos real activa para ejecutarse.  
> 3. Escribe al menos tres pruebas: una que verifique que 'iniciar' registra el job con el intervalo correcto, otra que valide que 'ejecutar_tarea_extraccion' llama adecuadamente a los métodos del loader en orden, y una tercera de caso adverso que pruebe el manejo seguro de excepciones de base de datos (DatabaseError) en el ciclo de ejecución.  
> 4. Sigue la guía de estilo PEP 8, incluye anotaciones de tipo y no dejes código de pruebas huérfano.`

3. Analiza la propuesta devuelta por Copilot, confirma la lógica de mockeado de la base de datos y copia el código en el archivo `tests/test_scheduler.py`:

```python
import pytest
from unittest.mock import MagicMock, patch
from psycopg2 import DatabaseError
from src.scheduler import SalesAutomationScheduler

@pytest.fixture
def scheduler_instance() -> SalesAutomationScheduler:
    """Fixture que provee una instancia limpia de SalesAutomationScheduler."""
    return SalesAutomationScheduler()

@patch('src.scheduler.DatabaseLoader')
def test_ejecutar_tarea_extraccion_exito(mock_loader_class: MagicMock, scheduler_instance: SalesAutomationScheduler) -> None:
    """Verifica que el ciclo de extracción funciona correctamente y libera los recursos."""
    # Configurar mock
    mock_loader_instance = MagicMock()
    mock_loader_class.return_value = mock_loader_instance

    # Ejecutar método de interés
    scheduler_instance.ejecutar_tarea_extraccion()

    # Validar llamadas y orden
    mock_loader_class.assert_called_once()
    mock_loader_instance.create_table.assert_called_once()
    mock_loader_instance.insertar_ventas_masivo.assert_called_once()
    mock_loader_instance.cerrar_conexion.assert_called_once()

@patch('src.scheduler.DatabaseLoader')
def test_ejecutar_tarea_extraccion_fallo_base_datos(mock_loader_class: MagicMock, scheduler_instance: SalesAutomationScheduler) -> None:
    """Comprueba que una excepción de base de datos no rompa el hilo de ejecución y cierre la conexión."""
    mock_loader_instance = MagicMock()
    # Simular una falla crítica en la creación de tablas
    mock_loader_instance.create_table.side_effect = DatabaseError("Fallo de conexión simulado")
    mock_loader_class.return_value = mock_loader_instance

    # Ejecutar - no debe propagar la excepción hacia arriba
    try:
        scheduler_instance.ejecutar_tarea_extraccion()
    except Exception as e:
        pytest.fail(f"La excepción no fue silenciada/manejada de forma interna: {e}")

    # Verificar que el recurso de base de datos fue liberado a pesar del fallo crítico
    mock_loader_instance.cerrar_conexion.assert_called_once()

def test_configuracion_job_planificador(scheduler_instance: SalesAutomationScheduler) -> None:
    """Verifica que el planificador añade de manera correcta el job con sus parámetros periódicos."""
    try:
        scheduler_instance.iniciar()
        # Obtener el job registrado
        job = scheduler_instance.scheduler.get_job('job_extraccion')
        
        assert job is not None
        assert job.id == 'job_extraccion'
        # Verificar que el intervalo de ejecución sea de 30 segundos
        assert job.trigger.interval.seconds == 30
    finally:
        scheduler_instance.detener()
```

4. Asegura que el archivo `__init__.py` exista dentro de `/home/usuario/workspace/automatizacion_ventas/tests/` para la resolución de módulos por parte de pytest:
```bash
touch /home/usuario/workspace/automatizacion_ventas/tests/__init__.py
```

5. Ejecuta la suite de pruebas unitarias desde la terminal:
```bash
export PYTHONPATH=.
pytest tests/test_scheduler.py -v
```

**Expected output**  
La terminal debe mostrar un reporte limpio indicando que las 3 pruebas unitarias simuladas pasaron de manera correcta (`passed`) en milisegundos sin requerir una base de datos PostgreSQL real levantada en el sistema local.

**Verification**  
Salida esperada en consola:
```text
tests/test_scheduler.py::test_ejecutar_tarea_extraccion_exito PASSED                     [ 33%]
tests/test_scheduler.py::test_ejecutar_tarea_extraccion_fallo_base_datos PASSED          [ 66%]
tests/test_scheduler.py::test_configuracion_job_planificador PASSED                      [100%]

================================== 3 passed in 0.15s ==================================
```

---

### Paso 5: Implementación de Healthchecks Automatizados para Contenedores Docker

**Objective**  
Diseñar y desplegar scripts de verificación de salud de base de datos e integración de servicios en Docker Compose, asistido por Copilot.

**Instructions**  

1. Crea el archivo de orquestación de Docker en la raíz: `/home/usuario/workspace/automatizacion_ventas/docker-compose.yml`.
2. Para asegurar un despliegue tolerante a fallos, utilizaremos a Copilot para diseñar una directiva `healthcheck` optimizada para el servicio PostgreSQL. Selecciona el archivo `docker-compose.yml` e introduce el siguiente prompt en Copilot Chat:

> **Prompt de Orquestación y Healthchecks:**  
> `Genera la configuración del archivo docker-compose.yml para nuestro entorno. Requisitos:  
> 1. Servicio 'db' usando la imagen oficial 'postgres:16.2-alpine'. Configura variables de entorno seguras: POSTGRES_DB=automation_db, POSTGRES_USER=postgres_user, POSTGRES_PASSWORD=secure_password_123. Puerto 5432 expuesto.  
> 2. Agrega una sección de 'healthcheck' al servicio 'db' que use 'pg_isready' con intervalos de 5 segundos, timeout de 5 segundos, reintentos de 5 y periodo de inicio de 10 segundos.  
> 3. Servicio 'app' para nuestra aplicación de automatización de ventas que construya a partir de un Dockerfile local. Debe depender del servicio 'db' con la condición 'service_healthy'.`

3. Revisa y escribe el código propuesto en `docker-compose.yml`:

```yaml
version: '3.8'

services:
  db:
    image: postgres:16.2-alpine
    container_name: automation-postgres
    environment:
      POSTGRES_DB: automation_db
      POSTGRES_USER: postgres_user
      POSTGRES_PASSWORD: secure_password_123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres_user -d automation_db"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  app:
    build: .
    container_name: sales-automation-app
    depends_on:
      db:
        condition: service_healthy
    environment:
      - DB_HOST=db
      - DB_NAME=automation_db
      - DB_USER=postgres_user
      - DB_PASSWORD=secure_password_123
      - DB_PORT=5432
    restart: on-failure

volumes:
  postgres_data:
```

4. Diseña el `Dockerfile` que empaqueta nuestra solución. Abre Copilot Chat y solicita la definición:

> **Prompt de Dockerfile:**  
> `Escribe un Dockerfile optimizado para nuestra automatización de ventas usando la imagen base ligera 'python:3.12.2-slim'. Debe instalar dependencias (psycopg2-binary, apscheduler), estructurar el espacio de trabajo en '/app', copiar el código fuente de 'src', y lanzar 'src/scheduler.py' como hilo de ejecución principal en modo persistente (debe correr de fondo).`

5. Basado en la respuesta, crea el archivo `/home/usuario/workspace/automatizacion_ventas/Dockerfile` con el siguiente contenido óptimo:

```dockerfile
FROM python:3.12.2-slim

## Evitar que Python escriba archivos .pyc en disco y forzar stdout/stderr sin buffer
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

## Instalar dependencias del sistema mínimas necesarias
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

## Instalar librerías de Python requeridas
RUN pip install --no-cache-dir \
    psycopg2-binary==2.9.9 \
    apscheduler==3.10.4

## Copiar el código fuente
COPY src/ /app/src/

ENV PYTHONPATH=/app

## Lanzar el programador en un script de ejecución continua para demostración en contenedor
CMD ["python", "-c", "import time; from src.scheduler import SalesAutomationScheduler; s = SalesAutomationScheduler(); s.iniciar(); import sys; sys.stdout.flush(); time.sleep(100)"]
```

6. Levanta los contenedores y valida la sincronización basada en el estado de salud:
```bash
docker compose up -d --build
```

**Expected output**  
El contenedor de la aplicación (`sales-automation-app`) entrará en modo de espera hasta que el servicio de base de datos (`automation-postgres`) ejecute su inicialización completa y responda con éxito al comando `pg_isready`. Solo en ese momento se levantará el servicio `app`.

**Verification**  
Ejecuta el comando para verificar el estado de los contenedores:
```bash
docker compose ps
```
Deberás observar que ambos contenedores se encuentran levantados y el contenedor `db` se reporta con el estado `(healthy)`:
```text
NAME                     IMAGE                  COMMAND                  SERVICE   CREATED         STATUS                   PORTS
automation-postgres      postgres:16.2-alpine   "docker-entrypoint.s…"   db        4 seconds ago   Up 2 seconds (healthy)   0.0.0.0:5432->5432/tcp
sales-automation-app     sales_automation_app   "python -c 'import t…"   app       4 seconds ago   Up 1 second              
```

---

## Validación y Pruebas

Para garantizar que los flujos de trabajo optimizados asistidos por IA y el pipeline de carga transaccional sean robustos y operen bajo estrictos contratos de tolerancia a fallos, realiza las siguientes actividades de verificación formal:

### Caso 1: Validación del Funcionamiento del UPSERT Masivo (Idempotencia)

1. Ejecuta manualmente el cargador de base de datos simulando una ingesta repetida del mismo identificador de venta.
2. Crea el archivo de prueba local de verificación `/home/usuario/workspace/automatizacion_ventas/verify_ingestion.py`:

```python
import os
import time
import psycopg2
from src.db_loader import DatabaseLoader

## Configuración de variables temporales para pruebas fuera de docker
os.environ["DB_HOST"] = "localhost"
os.environ["DB_NAME"] = "automation_db"
os.environ["DB_USER"] = "postgres_user"
os.environ["DB_PASSWORD"] = "secure_password_123"
os.environ["DB_PORT"] = "5432"

def validar_idempotencia():
    loader = DatabaseLoader()
    loader.create_table()
    
    # Lote 1: Datos iniciales
    lote_inicial = [
        {"id": "tx_999", "monto": 100.0, "moneda": "USD"}
    ]
    print("Insertando lote inicial...")
    loader.insertar_ventas_masivo(lote_inicial)
    
    # Consultar monto insertado
    loader.cursor.execute("SELECT monto FROM ventas WHERE id = 'tx_999';")
    monto_original = loader.cursor.fetchone()[0]
    print(f"Monto original registrado: {monto_original}")
    
    # Lote 2: Actualización del mismo registro con diferente valor (UPSERT)
    lote_actualizacion = [
        {"id": "tx_999", "monto": 150.50, "moneda": "USD"}
    ]
    print("Insertando lote de actualización (mismo ID con nuevo monto)...")
    loader.insertar_ventas_masivo(lote_actualizacion)
    
    # Consultar monto final
    loader.cursor.execute("SELECT monto FROM ventas WHERE id = 'tx_999';")
    monto_final = loader.cursor.fetchone()[0]
    print(f"Monto registrado después de UPSERT: {monto_final}")
    
    # Limpiar registro de prueba
    loader.cursor.execute("DELETE FROM ventas WHERE id = 'tx_999';")
    loader.conn.commit()
    
    loader.cerrar_conexion()
    
    assert monto_original == 100.0, "El valor inicial debió registrarse como 100.0"
    assert monto_final == 150.50, "La transacción no aplicó la actualización por conflicto de clave primaria (UPSERT)"
    print(">>> VALIDACIÓN DE IDEMPOTENCIA EXITOSA <<<")

if __name__ == "__main__":
    validar_idempotencia()
```

3. Ejecuta el script de validación local (asegúrate de que los contenedores estén activos):
```bash
python verify_ingestion.py
```

**Salida de éxito esperada:**
```text
Conexión con PostgreSQL establecida de manera exitosa.
Tabla 'ventas' verificada/creada exitosamente.
Insertando lote inicial...
Procesamiento masivo completado. Registros procesados/actualizados: 1
Monto original registrado: 100.0
Insertando lote de actualización (mismo ID con nuevo monto)...
Procesamiento masivo completado. Registros procesados/actualizados: 1
Monto registrado después de UPSERT: 150.5
Recursos de base de datos liberados correctamente.
>>> VALIDACIÓN DE IDEMPOTENCIA EXITOSA <<<
```

---

### Caso 2: Pruebas de Resiliencia ante Fallas Inyectadas (Caso Adverso)

Para validar los límites de la IA y comprobar la protección del flujo, alteraremos la estructura de base de datos eliminando temporalmente permisos del usuario para inducir fallos:

1. Ingresa directamente al contenedor de PostgreSQL y remueve temporalmente los permisos de actualización o corrompe la conexión forzando una desconexión de red de base de datos deteniendo el servicio `db`:
```bash
docker compose stop db
```

2. Ejecuta las pruebas unitarias locales con `pytest` para certificar que el mock simula de forma efectiva fallos de infraestructura sin colapsar el sistema de automatización:
```bash
pytest tests/test_scheduler.py -v
```

3. Verifica que la prueba `test_ejecutar_tarea_extraccion_fallo_base_datos` pase con éxito (indicativo de que los mecanismos de control de errores y logs sugeridos por Copilot manejan las excepciones críticas sin que la automatización aborte catastróficamente).

4. Restablece el servicio de base de datos para continuar:
```bash
docker compose start db
```

---

## Solución de Problemas

En caso de encontrar fallos comunes al interactuar con las sugerencias de Inteligencia Artificial y la orquestación del laboratorio, aplica las siguientes soluciones correctivas:

### Problema 1: Copilot sugiere sintaxis obsoleta de librerías o conectores SQL (ej. SQLAlchemy 1.4 o psycopg2 obsoleto)

* **Síntoma:** Al ejecutar el script de automatización o el cargador se registran excepciones como `AttributeError: module 'psycopg2' has no attribute 'extras'` o problemas debido al desuso de ciertas APIs heredadas.
* **Causa:** El modelo subyacente de la IA cuenta con sesgos provenientes de su set de datos de entrenamiento histórico donde abundan soluciones antiguas.
* **Solución (Prompt Correctivo):** No permitas sugerencias libres si notas discrepancias sintácticas. Reinicia el contexto del Chat en VS Code ingresando un prompt con restricciones exactas de versión:
  
  > **Prompt de Corrección:**  
  > `El código sugerido contiene APIs deprecadas. Refactoriza el método utilizando estrictamente las APIs modernas de 'psycopg2-binary' versión 2.9.9. Utiliza 'psycopg2.extras.execute_values' para el procesamiento del lote masivo y no utilices llamadas descontinuadas. Asegura el tipado según Python 3.12.2.`

---

### Problema 2: El contenedor de la app se detiene abruptamente de forma cíclica (bucle infinito de reinicios)

* **Síntoma:** Al ejecutar `docker compose ps` el contenedor `sales-automation-app` se reporta constantemente en estado `Restarting` o se detiene segundos después de levantarse.
* **Causa:** El programador `BackgroundScheduler` de APScheduler se ejecuta en un hilo de fondo (*non-blocking daemon thread*), lo que causa que el proceso principal de Python del Dockerfile finalice de inmediato al no encontrar más comandos pendientes en primer plano.
* **Solución:** Modifica el punto de entrada del Dockerfile o agrega un bucle de espera activa que mantenga al proceso principal vivo en primer plano de la siguiente manera:
  
  Escribe el comando `CMD` del Dockerfile asegurando un loop bloqueante:
  ```dockerfile
  CMD ["python", "-c", "import time; from src.scheduler import SalesAutomationScheduler; s = SalesAutomationScheduler(); s.iniciar(); import sys; sys.stdout.flush(); [time.sleep(1) for _ in iter(int, 1)]"]
  ```
  Esto garantizará que el contenedor Docker no termine de manera inmediata al finalizar la instrucción de inicialización de la suite.

---

## Limpieza

Para prevenir el consumo de recursos de memoria innecesarios en la máquina anfitriona y detener todos los hilos persistentes generados en la sesión práctica, ejecuta los siguientes comandos de liberación:

1. Detén y destruye todos los servicios, redes de integración y volúmenes persistentes de Docker Compose locales creados:
```bash
docker compose down -v
```

2. Verifica que ningún proceso huérfano de PostgreSQL o APScheduler de Python haya quedado remanente en segundo plano ejecutando consultas en el sistema de puertos locales:
```bash
## Limpieza de procesos Python locales del scheduler si existieran
pkill -f "src/scheduler.py" || true
pkill -f "verify_ingestion.py" || true
```

3. El directorio de trabajo `/home/usuario/workspace/automatizacion_ventas` puede ser archivado o eliminado para recuperar espacio libre en disco una vez evaluados los resultados por parte de tu instructor técnico.

---

## Resumen

En este laboratorio práctico has logrado optimizar y fortalecer una suite completa de automatización utilizando técnicas de desarrollo asistido por Inteligencia Artificial de forma controlada y robusta. 

### Puntos Clave Aprendidos
* **Ingeniería de Prompts Técnica:** La comunicación con la IA (GitHub Copilot) requiere delimitar explícitamente el rol del agente, definir las versiones exactas de las librerías objetivo (Python 3.12.2, PostgreSQL 16.2), y forzar el uso de tipado estático junto a un esquema estructurado de manejo de excepciones para generar código verdaderamente apto para producción.
* **Idempotencia de Base de Datos:** Las operaciones masivas *UPSERT* en PostgreSQL permiten que las tareas de automatización recurrentes se ejecuten múltiples veces sin provocar duplicaciones en los datos ni colapsos del sistema, resolviendo retos heredados de desarrollos previos ineficientes.
* **Pruebas y Mocks de Planificación:** La utilización asistida de IA agiliza el diseño de pruebas automatizadas complejas donde la sincronización temporal (APScheduler) y el acceso a red o bases de datos son simulados a través de `unittest.mock`, garantizando despliegues estables y agilidad en pipelines de CI/CD.

### Recursos Adicionales e Instrucciones de Referencia
* [Documentación Oficial de GitHub Copilot](https://docs.github.com/es/copilot)
* [Buenas Prácticas de Prompting para Desarrolladores de Software - OpenAI](https://platform.openai.com/docs/guides/prompt-engineering)
* [Documentación de Psycopg2 y execute_values](https://www.psycopg.org/docs/extras.html#psycopg2.extras.execute_values)
* [Estándar de Documentación de Código Python de Google](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings)
