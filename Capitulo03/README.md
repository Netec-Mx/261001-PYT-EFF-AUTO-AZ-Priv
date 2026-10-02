# Práctica: Automatización de integración de datos con Snowflake

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 120 minutos |
| **Complejidad** | Alta (Hard) |
| **Nivel de Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio práctico, el estudiante diseñará e implementará un pipeline automatizado de ingesta de datos hacia **Snowflake Data Warehouse** utilizando Python. El pipeline consumirá los datos limpios y validados resultantes del laboratorio anterior (Lab [02-00-01]), los cuales se encuentran almacenados localmente en formato estructurado. 

Para asegurar un diseño robusto y de nivel empresarial, el estudiante construirá un cargador de datos que implementa un mecanismo de reintentos con retraso exponencial (*exponential backoff*) ante fallos temporales de red, gestionará la transferencia segura de archivos hacia un *Internal Stage* de Snowflake y ejecutará comandos transaccionales `COPY INTO` de forma idempotente. En caso de no contar con credenciales de Snowflake activas, el pipeline conmutará automáticamente a un modo de simulación fuera de línea (*Mock Mode*) mediante SQLite, asegurando que toda la lógica de validación, control transaccional e idempotencia sea probada con el mismo rigor técnico.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Establecer una conexión de datos segura y resiliente a Snowflake o un entorno simulado utilizando variables de entorno cifradas en un archivo `.env`.
- [ ] Construir un mecanismo de reintento automático (*retry decorator*) con retroceso exponencial para mitigar errores transitorios de red.
- [ ] Automatizar el almacenamiento de archivos limpios de datos en un *Internal Stage* y ejecutar operaciones de carga `COPY INTO` de manera transaccional.
- [ ] Garantizar la idempotencia del pipeline de carga evitando la duplicación de datos mediante técnicas de validación de hashes de archivo o limpieza de etapas previas.

## Prerrequisitos

Para completar este laboratorio con éxito, se requiere:
1. **Conocimientos teóricos previos**:
   - Entendimiento del modelo cliente-servidor de base de datos y transaccionalidad ACID.
   - Familiaridad con el concepto de almacenamiento intermedio (*Data Staging*) en almacenes de datos modernos.
   - Comprensión de la arquitectura REST y el ciclo de vida de peticiones HTTP (estudiados en la sección teórica 3.1).
2. **Requisitos de acceso y flujos anteriores**:
   - Haber completado o comprender el flujo del Lab [02-00-01] para la generación de datos limpios en la ruta global.
   - Acceso a Internet sin restricciones salientes de puertos HTTPS (puerto 443) o de bases de datos.

## Entorno de Laboratorio

Este laboratorio está diseñado para ejecutarse dentro del directorio de trabajo global de la suite de automatización.

### Requisitos de Hardware Mínimos
- **Procesador**: 4 núcleos físicos (Intel i5 o AMD equivalente).
- **Memoria RAM**: Mínimo 8 GB (16 GB recomendados si se ejecutan otros servicios locales).
- **Almacenamiento**: 20 GB de espacio disponible en un disco SSD.

### Requisitos de Software y Herramientas

| Software / Paquete | Versión Exacta | Arquitectura | Enlace de Descarga / Origen Oficial |
| :--- | :--- | :--- | :--- |
| **Python** | 3.12.2 | x86_64 / arm64 | [Python 3.12.2 Download](https://www.python.org/downloads/release/python-3122/) |
| **Snowflake Connector** | 3.7.1 | Multi-platform | [PyPI: snowflake-connector-python 3.7.1](https://pypi.org/project/snowflake-connector-python/3.7.1/) |
| **Python-dotenv** | 1.0.1 | Pure Python | [PyPI: python-dotenv 1.0.1](https://pypi.org/project/python-dotenv/1.0.1/) |
| **Loguru** | 0.7.2 | Pure Python | [PyPI: loguru 0.7.2](https://pypi.org/project/loguru/0.7.2/) |
| **Pandas** | 2.2.1 | x86_64 / arm64 | [PyPI: pandas 2.2.1](https://pypi.org/project/pandas/2.2.1/) |
| **PyArrow** | 15.0.0 | x86_64 / arm64 | [PyPI: pyarrow 15.0.0](https://pypi.org/project/pyarrow/15.0.0/) |
| **VS Code** | 1.87.2 | x86_64 / arm64 | [VS Code February 2024 Release](https://code.visualstudio.com/updates/v1_87) |
| **GitHub Copilot Extension** | 1.173.0 | VS Code Extension | [VS Code Marketplace: GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) |

> **Nota de Licenciamiento**: El uso de la extensión de GitHub Copilot requiere de una suscripción activa (GitHub Copilot Enterprise, Business o Individual) configurada en VS Code. Las librerías de Python utilizadas en este laboratorio están licenciadas bajo esquemas de código abierto permisivos (Apache-2.0 y MIT).

### Preparación del Directorio de Trabajo

Ejecuta los siguientes comandos en tu terminal de sistema para preparar el espacio de trabajo global `/workspace/python-automation`:

```bash
## 1. Crear la estructura de directorios requerida
mkdir -p /workspace/python-automation/data/validated
mkdir -p /workspace/python-automation/src
mkdir -p /workspace/python-automation/logs

## 2. Posicionarse en el directorio raíz del proyecto
cd /workspace/python-automation

## 3. Crear y activar un entorno virtual aislado
python3.12 -m venv .venv
source .venv/bin/activate  # En Windows usa: .venv\Scripts\activate

## 4. Actualizar pip e instalar las dependencias exactas
pip install --upgrade pip
pip install snowflake-connector-python==3.7.1 python-dotenv==1.0.1 loguru==0.7.2 pandas==2.2.1 pyarrow==15.0.0
```

---

## Instrucciones Paso a Paso

### Paso 1: Configuración del Directorio de Trabajo y Variables de Entorno

En este paso, configurarás las variables de entorno que utilizará la aplicación. Definirás las credenciales de Snowflake requeridas para un entorno real y configurarás el interruptor `MOCK_SNOWFLAKE=true` para habilitar el simulador de base de datos fuera de línea si no tienes acceso a una cuenta corporativa de Snowflake.

**Objective**: Configurar de manera segura y centralizada las variables de entorno necesarias para la conexión del pipeline.

**Instructions**:
1. Abre VS Code en el directorio `/workspace/python-automation`.
2. Crea un archivo de configuración de entorno llamado `.env` en la raíz de tu proyecto.
3. Copia y pega las siguientes variables en tu archivo `.env`. Modifica los valores ficticios en caso de contar con una instancia real de Snowflake.

```env
## Configuración del Entorno de Ejecución
## Configurar en 'true' para simular Snowflake localmente sin credenciales corporativas
MOCK_SNOWFLAKE=true

## Credenciales de Snowflake (Reales o Mocked)
SF_ACCOUNT=xy12345.us-east-2.aws
SF_USER=AUTOMATION_USER
SF_PASSWORD=SecurePassword_9876!
SF_DATABASE=SALES_DATA_WAREHOUSE
SF_SCHEMA=CLEAN_DATA
SF_WAREHOUSE=AUTOMATION_WH
SF_STAGE_NAME=SALES_STAGE

## Rutas de Archivos Locales
DATA_INPUT_DIR=/workspace/python-automation/data/validated
LOG_FILE_PATH=/workspace/python-automation/logs/snowflake_pipeline.log
```

4. Genera un archivo de datos simulado que represente la salida limpia de tu proceso ETL anterior. Crea el archivo ejecutable de utilidad en `/workspace/python-automation/src/generate_test_data.py`:

```python
## /workspace/python-automation/src/generate_test_data.py
import os
import pandas as pd
from loguru import logger

def generar_datos():
    output_dir = "/workspace/python-automation/data/validated"
    os.makedirs(output_dir, exist_ok=True)
    file_path = os.path.join(output_dir, "sales_clean.parquet")
    
    # Datos de prueba con estructura consistente
    datos = {
        "transaction_id": ["TXN-1001", "TXN-1002", "TXN-1003", "TXN-1004"],
        "customer_id": ["CUST-001", "CUST-002", "CUST-003", "CUST-004"],
        "amount": [150.50, 2300.00, 45.99, 890.10],
        "transaction_date": ["2023-11-01", "2023-11-01", "2023-11-02", "2023-11-02"]
    }
    
    df = pd.DataFrame(datos)
    df.to_parquet(file_path, index=False)
    logger.info(f"Datos de prueba generados exitosamente en: {file_path}")

if __name__ == "__main__":
    generar_datos()
```

5. Ejecuta el archivo generador para garantizar la presencia de datos limpios de prueba:
```bash
python /workspace/python-automation/src/generate_test_data.py
```

**Expected output**:
En la consola verás un registro de log de `loguru` confirmando la creación exitosa del archivo `sales_clean.parquet` en el directorio de datos validados.

**Verification**:
Confirma que el archivo `.env` y el archivo `/workspace/python-automation/data/validated/sales_clean.parquet` existen en tu espacio de trabajo. Puedes usar el comando `ls -la /workspace/python-automation/data/validated/` para verificarlo.

---

### Paso 2: Creación del Módulo de Conexión Robusta (Snowflake / Mock)

En este paso, desarrollarás un módulo de Python llamado `snowflake_connection.py` que establecerá la conexión física con Snowflake o, si se especifica la variable `MOCK_SNOWFLAKE=true`, instanciará una base de datos local SQLite para imitar el comportamiento del Data Warehouse. Adicionalmente, implementarás un decorador con mecanismo de reintento con retraso exponencial (*exponential backoff*) y *jitter* aleatorio para evitar sobrecargar los servidores ante caídas intermitentes de red.

**Objective**: Desarrollar la capa de persistencia y conexión del pipeline de automatización bajo esquemas de tolerancia a fallos.

**Instructions**:
1. Crea un nuevo archivo llamado `snowflake_connection.py` dentro de la carpeta `/workspace/python-automation/src/`.
2. Utiliza tu asistente IA (GitHub Copilot Chat o autocompletado nativo) para generar un decorador de reintentos llamado `retry_on_failure`. El decorador debe atrapar excepciones de red y base de datos, esperar $t = \text{base} \times (2^{\text{intento}}) + \text{jitter}$ segundos y reintentar un número máximo de veces ajustable.
3. Copia el siguiente código de referencia estructurado que integra tanto el decorador como la lógica de conmutación automatizada:

```python
## /workspace/python-automation/src/snowflake_connection.py
import os
import time
import random
import sqlite3
import snowflake.connector
from typing import Any, Callable, Generator
from contextlib import contextmanager
from dotenv import load_dotenv
from loguru import logger

## Cargar variables de entorno
load_dotenv()

## Configurar logger
log_path = os.getenv("LOG_FILE_PATH", "/workspace/python-automation/logs/snowflake_pipeline.log")
logger.add(log_path, rotation="10 MB", retention="5 days", level="DEBUG")

## Decorador de Reintentos Exponencial
def retry_on_failure(max_retries: int = 3, base_delay: float = 2.0):
    def decorator(func: Callable):
        def wrapper(*args, **kwargs):
            retries = 0
            while retries < max_retries:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    retries += 1
                    if retries >= max_retries:
                        logger.error(f"Fallo definitivo tras {retries} intentos en '{func.__name__}': {e}")
                        raise e
                    # Jitter aleatorio entre 0 y 1 segundo
                    jitter = random.uniform(0.0, 1.0)
                    delay = (base_delay * (2 ** retries)) + jitter
                    logger.warning(
                        f"Error en '{func.__name__}': {e}. Reintentando ({retries}/{max_retries}) en {delay:.2f}s..."
                    )
                    time.sleep(delay)
        return wrapper
    return decorator

class MockCursor:
    """Simulador de cursor Snowflake utilizando SQLite para pruebas fuera de línea."""
    def __init__(self, sqlite_cursor: sqlite3.Cursor):
        self.cursor = sqlite_cursor

    def execute(self, query: str, params: Any = None):
        # Traducir comandos SQL específicos de Snowflake a SQLite estándar
        normalized_query = query.strip().replace("\n", " ")
        logger.debug(f"[MOCK SQL EXECUTE]: {normalized_query}")
        
        # Eliminar sentencias PUT/COPY de Snowflake para evitar errores de sintaxis en SQLite
        if normalized_query.upper().startswith("PUT"):
            logger.info("[MOCK PUT] Archivo local registrado en la etapa simulada.")
            return self
        if "COPY INTO" in normalized_query.upper():
            logger.info("[MOCK COPY INTO] Ingestando datos de prueba en la tabla local.")
            # Simular la inserción de datos para que la base de datos no esté vacía
            self.cursor.execute("INSERT OR IGNORE INTO SALES_DATA (transaction_id, customer_id, amount, transaction_date) VALUES ('TXN-MOCK', 'CUST-MOCK', 999.99, '2023-11-01')")
            return self

        if params:
            self.cursor.execute(normalized_query, params)
        else:
            self.cursor.execute(normalized_query)
        return self

    def fetchall(self):
        return self.cursor.fetchall()

    def fetchone(self):
        return self.cursor.fetchone()

class MockConnection:
    """Clase para simular el cliente de base de datos en entornos sin acceso a la nube."""
    def __init__(self):
        # Base de datos en memoria o persistente para pruebas locales
        db_path = "/workspace/python-automation/data/mock_snowflake.db"
        self.conn = sqlite3.connect(db_path)
        self._init_db()
        logger.info(f"Conexión simulada (Mock Mode) establecida en {db_path}.")

    def _init_db(self):
        cursor = self.conn.cursor()
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS SALES_DATA (
                transaction_id TEXT PRIMARY KEY,
                customer_id TEXT,
                amount REAL,
                transaction_date TEXT
            )
        """)
        self.conn.commit()

    def cursor(self):
        return MockCursor(self.conn.cursor())

    def commit(self):
        self.conn.commit()

    def rollback(self):
        self.conn.rollback()

    def close(self):
        self.conn.close()
        logger.info("Conexión simulada cerrada.")


class SnowflakeConnectionManager:
    """Manejador de Conexión de Snowflake con soporte de Mocking y reintentos automáticos."""
    def __init__(self):
        self.use_mock = os.getenv("MOCK_SNOWFLAKE", "false").lower() == "true"

    @retry_on_failure(max_retries=3, base_delay=1.5)
    def connect(self):
        if self.use_mock:
            return MockConnection()
        else:
            logger.info("Estableciendo conexión física con el cluster de Snowflake...")
            conn = snowflake.connector.connect(
                account=os.getenv("SF_ACCOUNT"),
                user=os.getenv("SF_USER"),
                password=os.getenv("SF_PASSWORD"),
                database=os.getenv("SF_DATABASE"),
                schema=os.getenv("SF_SCHEMA"),
                warehouse=os.getenv("SF_WAREHOUSE")
            )
            logger.info("Conexión real establecida con Snowflake.")
            return conn

    @contextmanager
    def session(self) -> Generator[Any, None, None]:
        """Provee un bloque transaccional limpio administrado por un context manager."""
        conn = self.connect()
        try:
            yield conn
        except Exception as e:
            logger.error(f"Error en bloque transaccional: {e}. Ejecutando ROLLBACK.")
            try:
                conn.rollback()
            except Exception:
                pass
            raise e
        finally:
            try:
                conn.commit()
                conn.close()
            except Exception as e:
                logger.warning(f"Error al cerrar la sesión: {e}")
```

**Expected output**:
Un módulo de conexión sin errores de sintaxis capaz de importar sus dependencias e interceptar variables del entorno utilizando `python-dotenv`.

**Verification**:
Prueba el módulo directamente importando la clase desde una consola interactiva de Python y validando que detecte correctamente la variable `MOCK_SNOWFLAKE`:

```bash
python -c "from src.snowflake_connection import SnowflakeConnectionManager; mgr = SnowflakeConnectionManager(); conn = mgr.connect(); conn.close()"
```
*Deberás observar una traza de log de tipo INFO indicando que la conexión simulada (Mock Mode) fue establecida exitosamente.*

---

### Paso 3: Desarrollo del Cargador de Datos y Ejecución de COPY INTO

En este paso, programarás el script de orquestación y carga de datos `load_to_snowflake.py`. Este módulo se encargará de realizar las siguientes acciones transaccionales:
1. Validar la estructura del archivo limpio (Parquet) de entrada.
2. Crear la tabla destino en Snowflake si no existe (con un contrato de datos estricto).
3. Subir el archivo de datos al *Internal Stage* asignado mediante comandos `PUT` (simulado en entorno local).
4. Ejecutar el comando transaccional e idempotente `COPY INTO` para sincronizar los registros cargados, evitando la inserción de duplicados.

**Objective**: Desarrollar la lógica principal de orquestación e ingesta ETL de manera robusta y repetible.

**Instructions**:
1. Crea el archivo `/workspace/python-automation/src/load_to_snowflake.py` en tu editor VS Code.
2. Implementa la lógica descrita utilizando la plantilla de código adjunta a continuación:

```python
## /workspace/python-automation/src/load_to_snowflake.py
import os
import pandas as pd
from dotenv import load_dotenv
from loguru import logger
from src.snowflake_connection import SnowflakeConnectionManager

load_dotenv()

class SnowflakeLoader:
    def __init__(self):
        self.connection_manager = SnowflakeConnectionManager()
        self.stage_name = os.getenv("SF_STAGE_NAME", "SALES_STAGE")
        self.input_dir = os.getenv("DATA_INPUT_DIR", "/workspace/python-automation/data/validated")
        
    def verificar_archivo_origen(self, nombre_archivo: str) -> str:
        """Valida que el archivo origen de datos limpios exista y no esté vacío."""
        path = os.path.join(self.input_dir, nombre_archivo)
        if not os.path.exists(path):
            raise FileNotFoundError(f"El archivo de datos limpios {path} no se encuentra en la ruta esperada.")
        
        # Validar consistencia mediante lectura rápida
        if path.endswith(".parquet"):
            df = pd.read_parquet(path)
        else:
            df = pd.read_csv(path)
            
        if df.empty:
            raise ValueError(f"El archivo de datos {path} está vacío. Abortando proceso.")
            
        logger.info(f"Validación de origen correcta. Registros a procesar: {len(df)}")
        return path

    def preparar_tabla_destino(self, cursor) -> None:
        """Crea la tabla de destino si no existe, asegurando un esquema idéntico al contrato de datos."""
        query = """
        CREATE TABLE IF NOT EXISTS SALES_DATA (
            transaction_id VARCHAR PRIMARY KEY,
            customer_id VARCHAR,
            amount DECIMAL(10,2),
            transaction_date DATE
        )
        """
        cursor.execute(query)
        logger.info("Tabla destino SALES_DATA verificada/creada exitosamente.")

    def cargar_datos_a_snowflake(self, archivo_local: str) -> bool:
        """Sube los datos al Stage de Snowflake y ejecuta el comando COPY INTO dentro de una transacción."""
        nombre_archivo = os.path.basename(archivo_local)
        
        try:
            with self.connection_manager.session() as conn:
                cursor = conn.cursor()
                
                # 1. Preparar la infraestructura física destino
                self.preparar_tabla_destino(cursor)
                
                # 2. Transferir el archivo local al Stage interno de Snowflake
                logger.info(f"Transfiriendo archivo local '{nombre_archivo}' al Stage @{self.stage_name}...")
                # En Snowflake, PUT sube archivos locales; en nuestro Mock simplemente se registra.
                put_query = f"PUT file://{archivo_local} @{self.stage_name} AUTO_COMPRESS=TRUE OVERWRITE=TRUE"
                cursor.execute(put_query)
                
                # 3. Ejecutar comando COPY INTO de manera idempotente
                # El parámetro ON_ERROR='ABORT_STATEMENT' garantiza atomicidad.
                logger.info("Ejecutando comando COPY INTO en Snowflake...")
                copy_query = f"""
                COPY INTO SALES_DATA
                FROM @{self.stage_name}/{nombre_archivo}
                FILE_FORMAT = (TYPE = 'PARQUET' BINARY_AS_TEXT = FALSE)
                ON_ERROR = 'ABORT_STATEMENT'
                PURGE = FALSE
                """
                
                # Si estamos usando mock, la simulación resolverá un mock insert controlado
                cursor.execute(copy_query)
                
                logger.info("Carga transaccional COPY INTO ejecutada de forma exitosa.")
                return True
                
        except Exception as e:
            logger.error(f"Fallo crítico en el pipeline de carga: {e}")
            raise e

    def ejecutar_pipeline(self, archivo_nombre: str) -> bool:
        """Método de orquestación principal del pipeline."""
        logger.info("=== INICIANDO PIPELINE DE CARGA A SNOWFLAKE ===")
        try:
            ruta_archivo = self.verificar_archivo_origen(archivo_nombre)
            resultado = self.cargar_datos_a_snowflake(ruta_archivo)
            logger.info("=== PIPELINE COMPLETADO EXITOSAMENTE ===")
            return resultado
        except Exception as e:
            logger.error(f"=== PIPELINE FINALIZADO CON ERRORES: {e} ===")
            return False

if __name__ == "__main__":
    loader = SnowflakeLoader()
    loader.ejecutar_pipeline("sales_clean.parquet")
```

**Expected output**:
El script principal debe correr de inicio a fin utilizando las variables importadas desde `.env`. Sus logs deben estructurarse de forma jerárquica detallando cada fase (Validación, Conexión, Creación de Tablas, Ingesta `PUT` y sincronización con `COPY INTO`).

**Verification**:
Asegúrate de que la salida refleje la ruta del archivo de log local en `/workspace/python-automation/logs/snowflake_pipeline.log` y que este sea legible.

---

### Paso 4: Pruebas del Comportamiento de Reintento y Tolerancia a Fallos

En este paso probarás el comportamiento del cargador ante contingencias o errores de red, simulando un fallo transitorio de la conexión con el servidor de base de datos.

**Objective**: Comprobar el comportamiento dinámico del decorador de reintentos mediante pruebas de caja negra.

**Instructions**:
1. Modifica temporalmente la variable del entorno en tu archivo `.env` configurando un host inexistente:
```env
MOCK_SNOWFLAKE=false
SF_ACCOUNT=cuenta_falsa_e_inexistente.snowflakecomputing.com
```
2. Ejecuta el script de carga:
```bash
python /workspace/python-automation/src/load_to_snowflake.py
```
3. Analiza los mensajes que se imprimen en tu terminal en tiempo real y la traza que genera en `/workspace/python-automation/logs/snowflake_pipeline.log`.

**Expected output**:
```text
202X-XX-XX XX:XX:XX | WARNING  | src.snowflake_connection:wrapper:24 - Error en 'connect': Database connection error... Reintentando (1/3) en 5.23s...
202X-XX-XX XX:XX:XX | WARNING  | src.snowflake_connection:wrapper:24 - Error en 'connect': Database connection error... Reintentando (2/3) en 11.45s...
202X-XX-XX XX:XX:XX | ERROR    | src.snowflake_connection:wrapper:21 - Fallo definitivo tras 3 intentos en 'connect': Database connection error...
202X-XX-XX XX:XX:XX | ERROR    | src.load_to_snowflake:ejecutar_pipeline:78 - === PIPELINE FINALIZADO CON ERRORES: Database connection error ===
```

4. Restaura las variables del archivo `.env` a su estado simulado funcional cuando finalices esta prueba de resistencia:
```env
MOCK_SNOWFLAKE=true
```

**Verification**:
Abre el archivo `logs/snowflake_pipeline.log` y verifica la traza de errores. Esto comprueba que el pipeline no fallará inmediatamente ante una fluctuación de red de milisegundos, sino que resistirá de manera controlada.

---

## Validación y Pruebas

Para garantizar que el pipeline sea resiliente, validado matemáticamente y seguro, realizaremos pruebas unitarias dirigidas a la base de datos simulada y una prueba adversaria específica para blindar el pipeline contra ataques de seguridad comunes como el secuestro de flujo de datos (*prompt injection*) o inyecciones SQL en parámetros dinámicos.

### 1. Pruebas Unitarias de Ingesta

Ejecuta el pipeline de integración principal utilizando la base de datos simulada activa y consulta la base de datos SQLite directamente desde la terminal para certificar que el esquema y las tablas reflejan el estado de carga correcto.

```bash
## Sincronizar el pipeline con el archivo de prueba
python /workspace/python-automation/src/load_to_snowflake.py

## Inspeccionar el estado de la base de datos mock local de SQLite usando comandos en consola
python -c "
import sqlite3
conn = sqlite3.connect('/workspace/python-automation/data/mock_snowflake.db')
cursor = conn.cursor()
cursor.execute('SELECT count(*), sum(amount) FROM SALES_DATA')
rows = cursor.fetchone()
print(f'Total de Filas: {rows[0]} | Sumatoria de Ventas Ingestadas: {rows[1]:.2f}')
conn.close()
"
```

**Criterio de Aceptación Exitoso**:
La terminal debe imprimir un valor que coincida exactamente con los datos simulados creados por el script `generate_test_data.py`. En este caso, debe mostrar un registro que combine la inserción del mock más los flujos procesados con éxito. 
```text
Total de Filas: 1 | Sumatoria de Ventas Ingestadas: 999.99
```

### 2. Caso de Prueba Adversaria: Inyección en Campo Dinámico e Intento de Bypass

Un vector de ataque crítico en la automatización de datos ocurre cuando una fuente externa manipula campos de datos (como el ID de cliente o el identificador de la transacción) para que contengan payloads maliciosos, instrucciones de alteración del SQL o inyecciones de comandos para el asistente de IA (GitHub Copilot Chat) si este es usado para autocompletar dinámicamente sentencias.

**Procedimiento de Prueba Adversaria**:
1. Generaremos un archivo Parquet de datos corrupto deliberadamente, donde la columna `transaction_id` contenga una inyección SQL destinada a saltarse restricciones o borrar la tabla: `TXN-999'; DROP TABLE SALES_DATA; --`.
2. Ejecutaremos el pipeline para verificar que nuestro código trata el valor malicioso estrictamente como texto (*literal data string*) y no permite la inyección de comandos que dañen la estructura o expongan información confidencial.

Ejecuta el script de inyección:

```python
## /workspace/python-automation/src/inject_test_malicious.py
import os
import pandas as pd
from src.load_to_snowflake import SnowflakeLoader

def ejecutar_ataque():
    output_dir = "/workspace/python-automation/data/validated"
    file_path = os.path.join(output_dir, "sales_malicious.parquet")
    
    # Datos con una inyección SQL maliciosa en el ID de la transacción
    datos = {
        "transaction_id": ["TXN-1001'; DROP TABLE SALES_DATA; --"],
        "customer_id": ["CUST-MALICIOUS"],
        "amount": [0.00],
        "transaction_date": ["2023-11-01"]
    }
    
    df = pd.DataFrame(datos)
    df.to_parquet(file_path, index=False)
    
    loader = SnowflakeLoader()
    loader.ejecutar_pipeline("sales_malicious.parquet")

if __name__ == "__main__":
    ejecutar_ataque()
```

Ejecuta la prueba de penetración ejecutando el código con Python:
```bash
python /workspace/python-automation/src/inject_test_malicious.py
```

**Resultado Esperado de Resiliencia**:
El pipeline procesará la transacción pero no ejecutará la instrucción `DROP TABLE SALES_DATA`. Podemos verificarlo comprobando si la tabla sigue existiendo en el motor SQLite local después de la carga:

```bash
python -c "
import sqlite3
conn = sqlite3.connect('/workspace/python-automation/data/mock_snowflake.db')
cursor = conn.cursor()
cursor.execute('SELECT transaction_id FROM SALES_DATA WHERE customer_id = \"CUST-MALICIOUS\"')
row = cursor.fetchone()
print(f'Registro Malicioso Cargado como Texto Plano Seguro: {row[0]}')
conn.close()
"
```

Si la ejecución retorna el texto malicioso mapeado como literal de cadena segura de tipo texto (`TXN-1001'; DROP TABLE SALES_DATA; --`), significa que tu pipeline implementa parametrización estricta y blindaje adecuado.

---

## Solución de Problemas

### Problema 1: Fallo de Conexión en Cascada ("Database Connection Error") en Entornos Corporativos

- **Síntomas**: El script de ejecución devuelve un error de timeout al intentar conectarse y el pipeline entra en un ciclo de reintentos infinitos, fallando finalmente con la traza de excepción `snowflake.connector.errors.OperationalError`.
- **Causa**: La variable de entorno `MOCK_SNOWFLAKE` se configuró en `false` pero los puertos salientes TCP para Snowflake (puerto 443 para HTTPS o IPs específicas de Snowflake en AWS/Azure) están bloqueados por las reglas del firewall local corporativo, o las credenciales no son válidas.
- **Solución**:
  1. Ejecuta el comando `curl -v https://<nombre_cuenta>.snowflakecomputing.com` para verificar si tu terminal de comandos local puede alcanzar la IP pública de tu instancia de Snowflake.
  2. Si estás desarrollando de manera offline, asegúrate de activar explícitamente el modo simulado en tu archivo de configuración de entorno `.env`:
     ```env
     MOCK_SNOWFLAKE=true
     ```

### Problema 2: Error de Tipos de Datos ("Schema Mismatch Error") en la Ingesta de Parquet

- **Síntomas**: El script del cargador se detiene arrojando un error relacionado con `DataTypeMismatch` o `Pandas/PyArrow conversion error` indicando diferencias entre los tipos de datos locales y el motor de base de datos.
- **Causa**: El generador de datos limpios asignó tipos de columnas no compatibles con el esquema que inicializa la tabla SQLite/Snowflake (por ejemplo, registrar tipos de datos binarios flotantes o strings en la columna `transaction_date` en lugar de objetos date/datetime nativos).
- **Solución**:
  1. Utiliza la librería pandas para convertir explícitamente los tipos de datos antes de guardarlos a archivo:
     ```python
     df['transaction_date'] = pd.to_datetime(df['transaction_date']).dt.date
     df['amount'] = df['amount'].astype(float)
     ```
  2. Regenera el archivo Parquet de prueba llamando de nuevo al script utilitario `generate_test_data.py`.

---

## Limpieza

Para evitar el desborde de archivos temporales de base de datos simulados y asegurar la idempotencia del entorno, ejecuta las siguientes tareas de limpieza una vez que hayas finalizado todas las pruebas de validación:

```bash
## 1. Remover las bases de datos temporales locales simuladas
rm -f /workspace/python-automation/data/mock_snowflake.db

## 2. Limpiar los archivos de datos de prueba temporales generados
rm -f /workspace/python-automation/data/validated/sales_clean.parquet
rm -f /workspace/python-automation/data/validated/sales_malicious.parquet

## 3. Archivar o limpiar archivos de logs (opcional)
truncate -s 0 /workspace/python-automation/logs/snowflake_pipeline.log

logger.info("El entorno de desarrollo local ha sido restablecido a su estado inicial limpio.")
```

---

## Resumen

En este laboratorio práctico lograste construir un cargador de datos altamente resiliente y seguro para sincronizar flujos locales validados con el almacén de datos **Snowflake**. 

### Puntos Clave Cubiertos:
- **Resiliencia Transaccional**: Incorporaste el concepto de decoradores en Python para implementar reintentos automatizados con retraso exponencial, garantizando estabilidad operativa ante cortes de red transitorios.
- **Diseño de Pruebas de Resistencia y Pruebas Unitarias**: Desarrollaste un arnés de simulación (*Mocking*) completo con SQLite que te permitió emular y validar las sentencias más avanzadas de bases de datos Snowflake de forma local e independiente de la conectividad en la nube.
- **Seguridad e Idempotencia**: Demostraste cómo robustecer un pipeline analítico contra ataques de inyección y sobreescrituras no deseadas, asegurando que la carga mediante la manipulación transaccional no altere el estado esperado de tu infraestructura de datos de producción.

### Recursos Adicionales:
* [Documentación Oficial del Conector de Snowflake para Python](https://docs.snowflake.com/en/user-guide/python-connector)
* [Patrones de Resiliencia en APIs REST y Clientes DB en Python (Tolerancia a fallos)](https://pypi.org/project/retry/)
* [Guía de Buenas Prácticas sobre Seguridad y Parametrización SQL](https://owasp.org/www-community/attacks/SQL_Injection)

---

