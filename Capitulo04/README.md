# Práctica: Monitoreo de pipelines y remediación controlada de pipelines

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 90 minutos |
| **Complejidad** | Difícil (Hard) |
| **Nivel de Bloom** | Aplicar (Apply) |

## Descripción General

En esta práctica, integrarás mecanismos avanzados de resiliencia, observabilidad y auto-recuperación en un flujo de trabajo automatizado desasistido. Configurarás un pipeline de datos robusto que simula la extracción de información desde una API externa inestable y la posterior persistencia de logs y estados de ejecución en una base de datos PostgreSQL local alojada en un contenedor Docker. 

El pipeline utilizará políticas de reintentos mediante exponencial backoff (retroceso exponencial con fluctuación aleatoria), capturará errores de forma estructurada a través de una jerarquía de excepciones personalizadas, y gestionará escenarios de fallo crítico persistente (*double-fault*). Si la API externa o la base de datos de orquestación fallan repetidamente, el pipeline transicionará de forma segura al estado `FALLIDO`, persistirá la traza en un archivo local de diagnóstico de emergencia como contingencia de última línea de defensa, y emitirá una alerta estructurada JSON que emula la integración con un webhook de Slack.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Implementar un manejador de excepciones jerárquico y estructurado que preserve la causa original de los errores a través de encadenamiento de excepciones (`raise ... from`).
- [ ] Diseñar y configurar políticas de remediación automática (*auto-healing*) utilizando decoradores de reintentos dinámicos con backoff exponencial.
- [ ] Centralizar y persistir el historial de estados operacionales (`INICIADO`, `REINTENTANDO`, `COMPLETADO`, `FALLIDO`) en una tabla de base de datos relacional para auditorías post-mortem.
- [ ] Desarrollar un mecanismo de persistencia auxiliar de contingencia para escenarios de doble fallo, asegurando la captura de telemetría diagnóstica aun si el motor de base de datos se encuentra fuera de línea.

## Prerrequisitos

Para completar con éxito este laboratorio, requieres:
- Comprensión sólida de la manipulación de flujos de control en Python (`try/except/finally`) y mecanismos de bloqueo o espera (`time.sleep`).
- Conocimiento intermedio de SQL transaccional para ejecutar consultas de verificación de esquemas y registros en PostgreSQL.
- Acceso previo completado a las configuraciones de almacenamiento del pipeline del Lab [03-00-01] y Lab [03-00-02].
- Un entorno de desarrollo con soporte para contenedores Docker configurado y habilitado.

## Entorno de Laboratorio

### Requisitos de Hardware y Software

| Componente | Especificación de Arquitectura y Versión | Fuente de Descarga Oficial |
| :--- | :--- | :--- |
| **Sistema Operativo** | Linux (Ubuntu 22.04 LTS x86_64) o macOS / Windows con WSL2 | [Ubuntu Downloads](https://releases.ubuntu.com/22.04/) |
| **Python** | Python 3.12.2 (AMD64/ARM64) | [Python Release 3.12.2](https://www.python.org/downloads/release/python-3122/) |
| **Docker Engine** | Docker Engine v25.0.3 | [Docker Install Docs](https://docs.docker.com/engine/install/) |
| **Docker Compose** | Docker Compose v2.24.5 | [Docker Compose Install](https://docs.docker.com/compose/install/) |
| **Tenacity** | Tenacity v8.2.3 (PyPI) | [Tenacity PyPI](https://pypi.org/project/tenacity/8.2.3/) |
| **SQLAlchemy** | SQLAlchemy v2.0.27 (PyPI) | [SQLAlchemy PyPI](https://pypi.org/project/SQLAlchemy/2.0.27/) |
| **Psycopg2-Binary**| Psycopg2-Binary v2.9.9 (PyPI) | [Psycopg2-Binary PyPI](https://pypi.org/project/psycopg2-binary/2.9.9/) |
| **Loguru** | Loguru v0.7.2 (PyPI) | [Loguru PyPI](https://pypi.org/project/loguru/0.7.2/) |
| **Pytest** | Pytest v8.1.1 (PyPI) | [Pytest PyPI](https://pypi.org/project/pytest/8.1.1/) |
| **IDE** | VS Code v1.87.2 con extensión GitHub Copilot v1.173.0 | [VS Code Releases](https://code.visualstudio.com/Updates) |

### Comandos de Inicialización del Entorno

Ejecuta las siguientes instrucciones en tu terminal para preparar el directorio global de trabajo `/workspace/python-automation` y desplegar el contenedor local de PostgreSQL.

```bash
## 1. Crear el directorio de trabajo global
mkdir -p /workspace/python-automation/data/validated
cd /workspace/python-automation

## 2. Iniciar el contenedor de PostgreSQL para orquestación
## Nombre: automation-postgres, Puerto: 5432, Base de datos: automation_db
docker run -d \
  --name automation-postgres \
  -e POSTGRES_DB=automation_db \
  -e POSTGRES_USER=postgres_user \
  -e POSTGRES_PASSWORD=secure_password_123 \
  -p 5432:5432 \
  --restart unless-stopped \
  postgres:16.2-alpine

## 3. Crear un entorno virtual de Python local e instalar dependencias
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install tenacity==8.2.3 sqlalchemy==2.0.27 psycopg2-binary==2.9.9 loguru==0.7.2 pytest==8.1.1
```

---

## Instrucciones Paso a Paso

### Paso 1: Configurar el esquema de base de datos de orquestación

**Objetivo**: Crear la tabla que almacenará los estados de ejecución de las corridas del pipeline y permitirá análisis post-mortem de errores.

**Instrucciones**:

1. Crea el archivo de definición del esquema SQL en `/workspace/python-automation/schema.sql`.
2. El esquema debe contener una tabla llamada `pipeline_runs` que almacene metadatos estructurados sobre la ejecución del pipeline (identificador, nombre del pipeline, marca de tiempo de inicio y fin, estado de ejecución y carga útil de diagnóstico en formato JSON o texto detallado).

Crea el archivo con el siguiente contenido:

```sql
-- /workspace/python-automation/schema.sql
CREATE TABLE IF NOT EXISTS pipeline_runs (
    run_id VARCHAR(50) PRIMARY KEY,
    pipeline_name VARCHAR(100) NOT NULL,
    started_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    ended_at TIMESTAMP WITH TIME ZONE,
    status VARCHAR(20) NOT NULL, -- INICIADO, REINTENTANDO, COMPLETADO, FALLIDO
    retry_count INT DEFAULT 0,
    diagnostic_logs TEXT,
    error_context JSONB
);
```

3. Ejecuta la sentencia SQL en el contenedor de base de datos `automation-postgres` utilizando la herramienta interactiva de línea de comandos `psql` integrada en la imagen oficial:

```bash
docker exec -i automation-postgres psql -U postgres_user -d automation_db < /workspace/python-automation/schema.sql
```

**Resultado esperado**:
La base de datos creará la tabla de forma silenciosa si no existe. Ningún mensaje de error debe retornar en la consola.

**Verificación**:
Ejecuta la siguiente instrucción para verificar la existencia de la tabla recién creada:

```bash
docker exec -it automation-postgres psql -U postgres_user -d automation_db -c "\dt pipeline_runs"
```

La salida esperada debe ser similar a:

```text
            List of relations
 Schema |     Name      | Type  |     Owner     
--------+---------------+-------+---------------
 public | pipeline_runs | table | postgres_user
(1 row)
```

---

### Paso 2: Desarrollar el módulo de excepciones jerárquico

**Objetivo**: Implementar clases de excepciones personalizadas para clasificar de manera inequívoca los fallos en la automatización y preservar la traza técnica real mediante encadenamiento semántico.

**Instrucciones**:

1. Diseña un archivo Python `/workspace/python-automation/exceptions.py` que defina la jerarquía de errores de la automatización:
    * `PipelineBaseError` (clase base para todo error del flujo).
    * `DatabaseConnectivityError` (derivado de la base, para errores de almacenamiento u orquestación de red local).
    * `ExternalAPIError` (derivado de la base, para fallos HTTP, timeouts de la API simulada de extracción).
2. Cada excepción debe aceptar metadatos operacionales estructurados adicionales (`contexto`) e implementar un método `.to_dict()` para su posterior exportación a la base de datos o alertas de mensajería.

Crea el archivo `/workspace/python-automation/exceptions.py` con el siguiente código:

```python
## /workspace/python-automation/exceptions.py
import datetime

class PipelineBaseError(Exception):
    """Clase base de excepción para todas las anomalías de la automatización."""
    def __init__(self, mensaje: str, contexto: dict = None):
        super().__init__(mensaje)
        self.mensaje = mensaje
        self.timestamp = datetime.datetime.now(datetime.timezone.utc).isoformat()
        self.contexto = contexto if contexto is not None else {}

    def to_dict(self) -> dict:
        return {
            "error_type": self.__class__.__name__,
            "message": self.mensaje,
            "timestamp": self.timestamp,
            "context": self.contexto
        }


class DatabaseConnectivityError(PipelineBaseError):
    """Lanzada cuando existen anomalías de conexión o timeout con la base de datos de control."""
    def __init__(self, mensaje: str, host: str, original_exception: Exception = None, contexto: dict = None):
        ctx = contexto or {}
        ctx.update({"db_host": host, "raw_error": repr(original_exception)})
        super().__init__(mensaje, contexto=ctx)
        if original_exception:
            self.__cause__ = original_exception


class ExternalAPIError(PipelineBaseError):
    """Lanzada cuando falla la API de terceros (timeout, error de servidor o payload inválido)."""
    def __init__(self, mensaje: str, url: str, status_code: int = None, original_exception: Exception = None, contexto: dict = None):
        ctx = contexto or {}
        ctx.update({"request_url": url, "http_status": status_code, "raw_error": repr(original_exception)})
        super().__init__(mensaje, contexto=ctx)
        if original_exception:
            self.__cause__ = original_exception
```

**Resultado esperado**:
Un módulo modular de excepciones tipadas estricto que permita empaquetar metadatos y soportar el mecanismo de encadenamiento nativo de Python (`__cause__`).

**Verificación**:
Puedes verificar la estructura básica de importación ejecutando una prueba rápida de sintaxis en la consola de Python interactiva:

```bash
python3 -c "from exceptions import DatabaseConnectivityError, ExternalAPIError; print('Modulo de excepciones cargado correctamente.')"
```

---

### Paso 3: Configurar la lógica de reintentos con Tenacity

**Objetivo**: Implementar un decorador parametrizado para la función de extracción que aplique políticas de recuperación automatizadas (backoff exponencial y jitter) antes de declarar un fallo persistente.

**Instrucciones**:

1. En esta sección se diseñará una función inestable que emulará la extracción de datos de ventas de una API pública.
2. Utilizaremos la biblioteca `tenacity` para envolver la función con las siguientes políticas:
    * Reintentar ante cualquier excepción del tipo `ExternalAPIError`.
    * Detenerse después de **3 reintentos máximos** (4 intentos en total).
    * Usar espera exponencial (`wait_exponential(multiplier=1, min=2, max=6)`) con jitter para evitar sobrecargar los recursos.
    * Registrar logs detallados de cada intento fallido y reintento en el logger global.

Crea un script preliminar de simulación `/workspace/python-automation/extractor_simulador.py`:

```python
## /workspace/python-automation/extractor_simulador.py
import random
from loguru import logger
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type, before_sleep_log
from exceptions import ExternalAPIError

## Configuración del logger de loguru para salida estándar clara
logger.add("pipeline_execution.log", format="{time:YYYY-MM-DD HH:mm:ss} | {level} | {message}", level="INFO")

## Variable global para simular el comportamiento errático de la API externa
intentos_realizados = 0

def registrar_reintento(retry_state):
    logger.warning(
        f"Intento fallido #{retry_state.attempt_number}. "
        f"Tiempo de espera calculado para el próximo reintento: {retry_state.next_action.sleep} segundos."
    )

@retry(
    retry=retry_if_exception_type(ExternalAPIError),
    stop=stop_after_attempt(4),
    wait=wait_exponential(multiplier=1, min=2, max=6),
    before_sleep=registrar_reintento,
    reraise=True
)
def extraer_datos_ventas(url: str, forzar_exito_en_intento: int = 99) -> dict:
    global intentos_realizados
    intentos_realizados += 1
    
    logger.info(f"Iniciando llamada a API externa: {url} (Ejecución #{intentos_realizados})")
    
    # Simular la inestabilidad de la red
    if intentos_realizados < forzar_exito_en_intento:
        # Lanzamos un error de comunicación de bajo nivel envuelto en nuestra excepción
        error_interno = ConnectionResetError("Remote connection reset by peer")
        raise ExternalAPIError(
            mensaje=f"Error transitorio de red al consumir la API en {url}",
            url=url,
            status_code=503,
            original_exception=error_interno,
            contexto={"attempt": intentos_realizados}
        )
    
    # Retorno exitoso simulado si supera la barrera de fallos configurada
    return {
        "estado": "exito",
        "datos_extraidos": [
            {"id": 101, "monto": 1500.50, "fecha": "2026-10-25"},
            {"id": 102, "monto": 340.00, "fecha": "2026-10-25"}
        ]
    }
```

**Resultado esperado**:
La función `extraer_datos_ventas` reintentará automáticamente aplicando un tiempo de espera que aumenta exponencialmente en cada intento fallido.

**Verificación**:
Ejecuta la simulación configurando el éxito para el intento número 3 para observar el backoff exponencial:

```bash
python3 -c "
import extractor_simulador
extractor_simulador.intentos_realizados = 0
try:
    res = extractor_simulador.extraer_datos_ventas('https://api.ventas.ejemplo/v1/extract', forzar_exito_en_intento=3)
    print('RESULTADO:', res)
except Exception as e:
    print('FALLÓ CON:', type(e).__name__)
"
```

Comprueba los logs mostrados en consola. Debes ver los reintentos transitorios marcando esperas de cerca de 2 y 4 segundos secuencialmente:

```text
2026-10-25 10:00:00 | INFO | Iniciando llamada a API externa: https://api.ventas.ejemplo/v1/extract (Ejecución #1)
2026-10-25 10:00:00 | WARNING | Intento fallido #1. Tiempo de espera calculado para el próximo reintento: 2.0 segundos.
2026-10-25 10:00:02 | INFO | Iniciando llamada a API externa: https://api.ventas.ejemplo/v1/extract (Ejecución #2)
2026-10-25 10:00:02 | WARNING | Intento fallido #2. Tiempo de espera calculado para el próximo reintento: 4.0 segundos.
2026-10-25 10:00:06 | INFO | Iniciando llamada a API externa: https://api.ventas.ejemplo/v1/extract (Ejecución #3)
RESULTADO: {'estado': 'exito', 'datos_extraidos': [{'id': 101, 'monto': 1500.5, 'fecha': '2026-10-25'}, {'id': 102, 'monto': 340.0, 'fecha': '2026-10-25'}]}
```

---

### Paso 4: Implementar la orquestación y el estado en PostgreSQL

**Objetivo**: Codificar un orquestador que actualice de forma asíncrona o síncrona el estado de la corrida del pipeline en la base de datos de control, manejando el escenario crítico en el que la propia base de datos de control no responda.

**Instrucciones**:

1. Crea el script de orquestación `/workspace/python-automation/orquestador_ventas.py`.
2. Implementa conexiones robustas a la base de datos PostgreSQL utilizando SQLAlchemy v2.0.27.
3. El script debe realizar las siguientes tareas principales:
    * Registrar el inicio de la corrida como `INICIADO` y asignarle un UUID único.
    * Ejecutar el proceso de extracción decorado del paso anterior.
    * Si la extracción tiene éxito, guardar los datos limpios en la ruta intermedia `/workspace/python-automation/data/validated/datos_ventas.json` y actualizar la base de datos de control a `COMPLETADO`.
    * Si la extracción falla tras los reintentos permitidos, capturar la excepción, marcar el estado general como `FALLIDO` en la base de datos guardando el diagnóstico del error y alertar al equipo.
    * **Mecanismo de Doble Fallo (Double-Fault)**: Si la conexión a la base de datos PostgreSQL falla durante cualquier actualización de estado, el script no debe colapsar de forma silenciosa. Debe interceptar la excepción, generar un archivo JSON de emergencia localmente en la ruta `/workspace/python-automation/data/validated/emergencia_diagnostico.json` con toda la información del error y registrar con prioridad `CRITICAL` en los logs la caída del motor de control.

Crea el script `/workspace/python-automation/orquestador_ventas.py` con el siguiente código:

```python
## /workspace/python-automation/orquestador_ventas.py
import json
import os
import uuid
import datetime
from loguru import logger
from sqlalchemy import create_engine, text
from sqlalchemy.exc import SQLAlchemyError

from exceptions import DatabaseConnectivityError, ExternalAPIError
import extractor_simulador

## Constantes del entorno global de la automatización
DATABASE_URI = "postgresql://postgres_user:secure_password_123@localhost:5432/automation_db"
DATA_DIRECTORY = "/workspace/python-automation/data/validated"
EMERGENCY_FILE_PATH = os.path.join(DATA_DIRECTORY, "emergencia_diagnostico.json")

## Asegurar la existencia del directorio de almacenamiento local intermedio
os.makedirs(DATA_DIRECTORY, exist_ok=True)

class OrquestadorPipeline:
    def __init__(self, database_uri: str):
        self.db_uri = database_uri
        # Configurar engine con timeout corto para simular caídas de forma eficiente
        self.engine = create_engine(
            self.db_uri,
            connect_args={'connect_timeout': 3}
        )
        self.run_id = str(uuid.uuid4())
        self.pipeline_name = "pipeline_ventas_monitoreado"

    def _persistir_estado_db(self, status: str, retry_count: int = 0, error_payload: dict = None) -> bool:
        """
        Método auxiliar para persistir estados en PostgreSQL.
        Retorna True si tiene éxito; de lo contrario, lanza DatabaseConnectivityError.
        """
        query_upsert = text("""
            INSERT INTO pipeline_runs (run_id, pipeline_name, status, retry_count, error_context, ended_at)
            VALUES (:run_id, :pipeline_name, :status, :retry_count, :error_context, :ended_at)
            ON CONFLICT (run_id) 
            DO UPDATE SET 
                status = EXCLUDED.status,
                retry_count = EXCLUDED.retry_count,
                error_context = EXCLUDED.error_context,
                ended_at = EXCLUDED.ended_at;
        """)
        
        ended_at = None
        if status in ["COMPLETADO", "FALLIDO"]:
            ended_at = datetime.datetime.now(datetime.timezone.utc)

        try:
            with self.engine.begin() as conn:
                conn.execute(query_upsert, {
                    "run_id": self.run_id,
                    "pipeline_name": self.pipeline_name,
                    "status": status,
                    "retry_count": retry_count,
                    "error_context": json.dumps(error_payload) if error_payload else None,
                    "ended_at": ended_at
                })
            logger.info(f"Estado de la ejecución '{self.run_id}' persistido exitosamente en DB como: {status}")
            return True
        except SQLAlchemyError as sql_err:
            # Fallo de la base de datos de control
            raise DatabaseConnectivityError(
                mensaje="No se pudo escribir el estado del pipeline en PostgreSQL",
                host="localhost",
                original_exception=sql_err,
                contexto={"run_id": self.run_id, "attempted_status": status}
            ) from sql_err

    def _persistir_estado_emergencia(self, error_interno: Exception, estado_intentado: str):
        """
        Mapea el doble fallo escribiendo la telemetría operacional en un archivo JSON 
        de contingencia offline para preservar la observabilidad en ambientes degradados.
        """
        logger.critical("¡DOBLE FALLO DETECTADO! Registrando telemetría en almacenamiento local de contingencia.")
        
        datos_emergencia = {
            "run_id": self.run_id,
            "pipeline_name": self.pipeline_name,
            "timestamp_emergencia": datetime.datetime.now(datetime.timezone.utc).isoformat(),
            "estado_intentado": estado_intentado,
            "diagnostico_orquestador": {
                "mensaje_error_base": str(error_interno),
                "causa_raiz": repr(error_interno.__cause__) if hasattr(error_interno, '__cause__') and error_interno.__cause__ else None,
                "detalles_contexto": getattr(error_interno, 'contexto', {})
            }
        }
        
        try:
            with open(EMERGENCY_FILE_PATH, "w") as em_file:
                json.dump(datos_emergencia, em_file, indent=4)
            logger.warning(f"Archivo de diagnóstico de emergencia escrito con éxito en {EMERGENCY_FILE_PATH}")
        except Exception as file_err:
            logger.critical(f"Fallo catastrófico crítico: No se pudo escribir el log local de contingencia. {str(file_err)}")

    def ejecutar_pipeline(self, api_url: str, forzar_exito_en_intento: int = 99):
        logger.info(f"=== Iniciando Pipeline Run: {self.run_id} ===")
        
        # 1. Registrar inicio
        try:
            self._persistir_estado_db(status="INICIADO")
        except DatabaseConnectivityError as db_err:
            logger.error(f"Fallo de persistencia inicial: {db_err}")
            self._persistir_estado_emergencia(db_err, "INICIADO")
            return

        # 2. Ejecutar extracción con remediación por Tenacity
        extractor_simulador.intentos_realizados = 0  # Reiniciar contador simulado
        try:
            datos = extractor_simulador.extraer_datos_ventas(api_url, forzar_exito_en_intento)
            
            # Guardar el resultado en almacenamiento local persistente limpio
            salida_datos_path = os.path.join(DATA_DIRECTORY, "datos_ventas.json")
            with open(salida_datos_path, "w") as out_file:
                json.dump(datos, out_file, indent=4)
                
            logger.info("Fase de extracción completada exitosamente.")
            
            # Actualizar DB a COMPLETADO
            self._persistir_estado_db(status="COMPLETADO", retry_count=extractor_simulador.intentos_realizados - 1)
            
        except ExternalAPIError as api_err:
            # Captura cuando los intentos programados fallan
            logger.error(f"Error irrecuperable en la API externa tras reintentos acumulados. Diagnóstico: {api_err.mensaje}")
            
            # Intentar guardar el estado de FALLIDO con contexto JSON de la excepción enriquecida
            error_dict = api_err.to_dict()
            try:
                self._persistir_estado_db(
                    status="FALLIDO", 
                    retry_count=extractor_simulador.intentos_realizados - 1,
                    error_payload=error_dict
                )
                self.enviar_alerta_simulada(error_dict)
            except DatabaseConnectivityError as db_err:
                # Manejar doble fallo
                logger.error(f"Incapacidad de reportar fallo del pipeline en DB por: {db_err}")
                # Encadenamos la caída de DB para guardar todo el estado localmente
                self._persistir_estado_emergencia(db_err, "FALLIDO")
                self.enviar_alerta_simulada(error_dict)
                
        except Exception as error_imprevisto:
            logger.error(f"Error imprevisto en tiempo de ejecución: {error_imprevisto}")
            # Encadenar en excepción genérica enriquecida
            try:
                self._persistir_estado_db(
                    status="FALLIDO",
                    retry_count=0,
                    error_payload={"generic_error": str(error_imprevisto)}
                )
            except DatabaseConnectivityError as db_err:
                self._persistir_estado_emergencia(db_err, "FALLIDO")

    def enviar_alerta_simulada(self, payload_error: dict):
        """Simula el envío de una alerta estructurada mediante un Webhook a Slack."""
        payload_slack = {
            "channel": "#ops-alerts",
            "username": "Pipeline Resilience Bot",
            "attachments": [{
                "color": "#FF0000",
                "title": f"Fallo Crítico en {self.pipeline_name}",
                "text": f"La corrida del pipeline {self.run_id} finalizó con error.",
                "fields": [
                    {"title": "Tipo de Error", "value": payload_error.get("error_type"), "short": True},
                    {"title": "Mensaje", "value": payload_error.get("message"), "short": False},
                    {"title": "Intentos Fallidos", "value": str(extractor_simulador.intentos_realizados), "short": True}
                ],
                "fallback": "Error en el pipeline de automatización."
            }]
        }
        
        # Guardar la alerta de Slack generada como archivo para validación del laboratorio
        slack_alert_path = os.path.join(DATA_DIRECTORY, "slack_alert.json")
        with open(slack_alert_path, "w") as f:
            json.dump(payload_slack, f, indent=4)
        
        logger.warning(f"Simulación de Alerta de Slack registrada en {slack_alert_path}")
```

**Resultado esperado**:
El orquestador está listo para capturar fallos operacionales, registrar estados transaccionales precisos en PostgreSQL e interceptar problemas de infraestructura interna persistiendo datos en archivos locales planos si la base de datos de control colapsa.

**Verificación**:
Verifica que el script compila correctamente sin errores de sintaxis o importaciones:

```bash
python3 -c "import orquestador_ventas; print('Orquestador importado con éxito.')"
```

---

### Paso 5: Ejecutar simulaciones controladas de ejecución

**Objetivo**: Validar el comportamiento del sistema ante tres escenarios de control bien diferenciados: ejecución feliz con remediación exitosa, fallo catastrófico controlado en origen y caída crítica del motor de control (doble fallo).

**Instrucciones**:

#### Escenario A: Ejecución con anomalía transitoria corregida por reintentos (Happy Path con Auto-Healing)

1. En este escenario, la API fallará en las dos primeras llamadas pero tendrá éxito en el tercer intento. El pipeline debe marcarse como `COMPLETADO`.
2. Ejecuta el siguiente comando en la terminal:

```bash
python3 -c "
import orquestador_ventas
orq = orquestador_ventas.OrquestadorPipeline(orquestador_ventas.DATABASE_URI)
orq.ejecutar_pipeline('https://api.ventas.ejemplo/v1/extract', forzar_exito_en_intento=3)
"
```

3. Revisa la base de datos PostgreSQL para verificar que el estado registrado es `COMPLETADO` con conteo de reintentos igual a 2.

```bash
docker exec -it automation-postgres psql -U postgres_user -d automation_db -c "SELECT run_id, status, retry_count FROM pipeline_runs;"
```

La salida esperada de la consulta SQL de control debe ser similar a:

```text
                run_id                |  status   | retry_count 
--------------------------------------+-----------+-------------
 e98fa4c7-1ab3-47e1-88f1-8f553f47c0b1 | COMPLETADO|           2
(1 row)
```

#### Escenario B: Falla definitiva del servicio origen (Fallo persistente controlado)

1. En este escenario, la API fallará en los 4 intentos de extracción de forma persistente. El pipeline debe transicionar a `FALLIDO` y generar la alerta estructurada.
2. Limpia los registros anteriores para facilidad de análisis e inicia el orquestador forzando que nunca tenga éxito en la API (pásale un valor de éxito inalcanzable, por ejemplo, `forzar_exito_en_intento=99`):

```bash
docker exec -it automation-postgres psql -U postgres_user -d automation_db -c "TRUNCATE TABLE pipeline_runs;"

python3 -c "
import orquestador_ventas
orq = orquestador_ventas.OrquestadorPipeline(orquestador_ventas.DATABASE_URI)
orq.ejecutar_pipeline('https://api.ventas.ejemplo/v1/extract', forzar_exito_en_intento=99)
"
```

3. Verifica el estado en la base de datos de orquestación y comprueba la existencia de la alerta estructurada generada para Slack.

```bash
docker exec -it automation-postgres psql -U postgres_user -d automation_db -c "SELECT status, retry_count, error_context FROM pipeline_runs;"
```

La columna `error_context` debe contener los metadatos serializados del error de forma estructurada con su marca de tiempo, URL de petición e indicación del código HTTP fallido.

**Resultado esperado**:
El estado del registro en la tabla SQL debe cambiar a `FALLIDO`. Adicionalmente, el archivo de simulación de Slack `/workspace/python-automation/data/validated/slack_alert.json` debe contener el payload correcto con el formato detallado.

---

### Paso 6: Simular y verificar escenario de Doble Fallo (Double-Fault)

**Objetivo**: Comprobar que el pipeline de automatización no genera "puntos ciegos de telemetría" y ejecuta su contingencia de forma satisfactoria cuando tanto la API de origen como la base de datos PostgreSQL fallan simultáneamente.

**Instrucciones**:

1. Simularemos una caída total de infraestructura de red de control alterando temporalmente la cadena de conexión de la base de datos en el constructor o deteniendo temporalmente el contenedor Docker de PostgreSQL.
2. Detén el contenedor de PostgreSQL para simular la caída:

```bash
docker stop automation-postgres
```

3. Ejecuta el pipeline forzando que falle la API para disparar el flujo de error y el reporte de fallo:

```bash
python3 -c "
import orquestador_ventas
orq = orquestador_ventas.OrquestadorPipeline(orquestador_ventas.DATABASE_URI)
orq.ejecutar_pipeline('https://api.ventas.ejemplo/v1/extract', forzar_exito_en_intento=99)
"
```

4. Observa atentamente la salida de la consola. El script debe capturar el error de la conexión de la base de datos, no detenerse abruptamente con un traceback descontrolado, y escribir el archivo `/workspace/python-automation/data/validated/emergencia_diagnostico.json`.

**Resultado esperado**:
La salida en consola de loguru mostrará mensajes de alerta de nivel `CRITICAL` y advertencias sobre la escritura del log local.

**Verificación**:
1. Inspecciona el archivo de contingencia autogenerado en busca de metadatos de error válidos:

```bash
cat /workspace/python-automation/data/validated/emergencia_diagnostico.json
```

Debe mostrar un contenido similar al siguiente, confirmando que la excepción encadenada fue capturada y serializada localmente:

```json
{
    "run_id": "...",
    "pipeline_name": "pipeline_ventas_monitoreado",
    "timestamp_emergencia": "2026-10-25T...",
    "estado_intentado": "FALLIDO",
    "diagnostico_orquestador": {
        "mensaje_error_base": "No se pudo escribir el estado del pipeline en PostgreSQL",
        "causa_raiz": "OperationalError(...)",
        "detalles_contexto": {
            "db_host": "localhost",
            "attempted_status": "FALLIDO"
        }
    }
}
```

2. Restablece el contenedor Docker para dejar el entorno limpio:

```bash
docker start automation-postgres
```

---

## Validación y Pruebas

Para garantizar la calidad de la automatización construida, ejecutaremos un set de pruebas unitarias automatizadas con `pytest` que simulan y comprueban de manera estricta las reglas de resiliencia especificadas.

### 1. Pruebas de Calidad de Código con Pytest

Crea un archivo de pruebas automatizadas `/workspace/python-automation/test_pipeline.py` para validar los comportamientos bajo aislamiento controlado:

```python
## /workspace/python-automation/test_pipeline.py
import pytest
import os
import json
from exceptions import ExternalAPIError, DatabaseConnectivityError
import extractor_simulador
import orquestador_ventas

def test_jerarquia_excepciones():
    """Valida que las excepciones personalizadas preserven tipos y soporten serialización."""
    inner_err = ValueError("Invalid structure")
    ex = ExternalAPIError("Bad connection", "https://api.example.com", 500, inner_err, {"custom_meta": 123})
    
    assert ex.mensaje == "Bad connection"
    assert ex.__cause__ == inner_err
    assert ex.contexto["request_url"] == "https://api.example.com"
    assert ex.contexto["custom_meta"] == 123
    
    serialized = ex.to_dict()
    assert serialized["error_type"] == "ExternalAPIError"
    assert "timestamp" in serialized

def test_politica_reintentos_tenacity():
    """Verifica que la función de extracción se rinda tras exactamente 4 intentos si persiste el error."""
    extractor_simulador.intentos_realizados = 0
    
    with pytest.raises(ExternalAPIError):
        # Configurar para que nunca tenga éxito
        extractor_simulador.extraer_datos_ventas("https://api.test/fail", forzar_exito_en_intento=10)
        
    # Tenacity debe detener el reintento al 4to intento (1 original + 3 reintentos)
    assert extractor_simulador.intentos_realizados == 4

def test_contingencia_doble_fallo(tmp_path):
    """Prueba que el orquestador escriba el archivo local de contingencia si PostgreSQL no está disponible."""
    # Configurar una URI de base de datos inválida que falle inmediatamente
    orq = orquestador_ventas.OrquestadorPipeline("postgresql://invalid_user:wrong_pwd@127.0.0.1:9999/no_db")
    
    # Redefinir ruta del archivo de emergencia en el test utilizando tmp_path
    emergency_file_mock = os.path.join(tmp_path, "emergencia_diagnostico.json")
    orquestador_ventas.EMERGENCY_FILE_PATH = emergency_file_mock
    
    # Ejecutar pipeline y comprobar que se recupera elegantemente sin lanzar excepción global
    orq.ejecutar_pipeline("https://api.test/fail", forzar_exito_en_intento=1)
    
    assert os.path.exists(emergency_file_mock)
    with open(emergency_file_mock) as f:
        data = json.load(f)
        assert data["pipeline_name"] == "pipeline_ventas_monitoreado"
        assert "diagnostico_orquestador" in data
```

Ejecuta el suite de pruebas para confirmar el correcto funcionamiento de tus lógicas de control:

```bash
pytest -v /workspace/python-automation/test_pipeline.py
```

Deberías ver una salida exitosa que valide los tres casos de prueba:

```text
============================= test session starts ==============================
collected 3 items                                                              

test_pipeline.py::test_jerarquia_excepciones PASSED                      [ 33%]
test_pipeline.py::test_politica_reintentos_tenacity PASSED               [ 66%]
test_pipeline.py::test_contingencia_doble_fallo PASSED                   [100%]

============================== 3 passed in 0.12s ===============================
```

### 2. Prueba de Inyección Adversaria (Caso de Prueba de Seguridad)

Para garantizar la estabilidad operacional ante eventos maliciosos u operaciones de inyección de código, validaremos que nuestro pipeline y base de datos toleren payloads corruptos con caracteres especiales o secuencias SQL reservadas en los campos de diagnóstico sin comprometer el pipeline.

Ejecuta la siguiente simulación pasando datos maliciosos en los parámetros de error y verifica la correcta sanitización de SQLAlchemy en la base de datos de control:

```bash
python3 -c "
import orquestador_ventas
orq = orquestador_ventas.OrquestadorPipeline(orquestador_ventas.DATABASE_URI)
## Inyección simulada en el campo de diagnóstico que usualmente provocaría exploits
payload_malicioso = {
    'error_type': 'ExternalAPIError',
    'message': \"'; DROP TABLE pipeline_runs; --\",
    'malicious_payload': 'TRUE OR 1=1'
}
orq._persistir_estado_db('FALLIDO', retry_count=3, error_payload=payload_malicioso)
print('Payload de control procesado de forma segura sin consecuencias en la base de datos.')
"
```

Verifica que la tabla `pipeline_runs` permanezca intacta y el registro con la inyección haya sido guardado de forma segura como texto plano:

```bash
docker exec -it automation-postgres psql -U postgres_user -d automation_db -c "SELECT status, error_context FROM pipeline_runs WHERE status='FALLIDO';"
```

---

## Solución de Problemas

A continuación, se describen dos escenarios de error comunes identificados durante la ejecución de este laboratorio, junto con sus causas y planes de acción para resolverlos.

### Caso 1: Error de conexión `Is the server running on host "localhost" (127.0.0.1) and accepting TCP/IP connections on port 5432?`

- **Síntoma**: Al iniciar el pipeline en el Paso 5, este reporta de forma inmediata que la base de datos PostgreSQL no se encuentra disponible y genera archivos de emergencia de forma prematura.
- **Causa**: El contenedor Docker `automation-postgres` no se encuentra en ejecución, fue detenido o el puerto local `5432` está siendo ocupado por otro servicio de base de datos nativo del sistema host.
- **Solución**:
    1. Revisa el estado del contenedor ejecutando `docker ps -a`.
    2. Si el contenedor existe pero está detenido, inícialo con: `docker start automation-postgres`.
    3. Si existe una colisión de puertos en tu máquina local, detén tu base de datos nativa o modifica la publicación del puerto en el comando docker a `-p 5433:5432` y actualiza la constante `DATABASE_URI` en `orquestador_ventas.py` para apuntar a `localhost:5433`.

### Caso 2: El suite de pruebas de Pytest se congela o toma demasiado tiempo en ejecutarse

- **Síntoma**: Al ejecutar `pytest -v test_pipeline.py`, el proceso tarda varios segundos o minutos en la prueba de reintentos.
- **Causa**: El decorador `@retry` de Tenacity está configurado con valores de espera de la vida real (`multiplier=1, min=2, max=6`) lo que fuerza llamadas síncronas bloqueantes de `time.sleep` entre reintentos fallidos en tus casos de pruebas.
- **Solución**:
    1. Para tus entornos de pruebas automáticas de integración, se recomienda utilizar técnicas de "mocking" o parches de biblioteca sobre el temporizador para sobreescribir los intervalos de Tenacity, o utilizar variables de entorno para parametrizar de forma dinámica el comportamiento de reintentos a intervalos mínimos de milisegundos (`wait_fixed(0.01)`) cuando se detecta un contexto de pruebas unitarias (`TESTING=true`).

---

## Limpieza

Para liberar los recursos del sistema y dejar el espacio de trabajo en su estado original, realiza los siguientes pasos de limpieza en tu terminal:

1. Detén y remueve de forma definitiva el contenedor Docker de PostgreSQL:

```bash
docker stop automation-postgres
docker rm automation-postgres
```

2. Elimina los archivos temporales de ejecución y datos generados en el directorio del laboratorio:

```bash
rm -rf /workspace/python-automation/data/validated/*
rm -f /workspace/python-automation/pipeline_execution.log
```

3. Desactiva el entorno virtual de Python si lo mantienes activo:

```bash
deactivate
```

---

## Resumen

En esta sesión práctica, has construido un flujo de automatización robusto bajo patrones modernos de resiliencia y monitoreo utilizando Python, Tenacity y PostgreSQL. 

A lo largo del laboratorio, lograste:
1. **Modelar una jerarquía de excepciones dedicada**: Esto te permite separar errores lógicos de negocio de fallas físicas de infraestructura de red de datos.
2. **Implementar políticas de remediación automatizada**: A través de reintentos controlados con retroceso exponencial (*exponential backoff*), mitigando picos de fallas de red sin sobrecargar los recursos.
3. **Persistir estados de orquestación transaccional**: Centralizando el monitoreo histórico en PostgreSQL utilizando mapeadores relacionales con SQLAlchemy v2.0.27.
4. **Diseñar resiliencia ante doble fallo (double-fault resilience)**: Creando canales de salida diagnósticos locales JSON como mecanismos de contingencia offline.

Estos mecanismos son cruciales en la arquitectura de datos moderna, donde la observabilidad en ambientes desasistidos (*headless*) es la única garantía de mantener la integridad operacional.
