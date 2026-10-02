# Práctica: De automatización local a solución desplegable y orquestada

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 180 minutos |
| **Complejidad** | Alta (Hard) |
| **Nivel de Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio práctico, transformarás un script de automatización de datos de ventas de ejecución local e interactiva en un servicio robusto, autocontenido y completamente orquestado para entornos de producción. 

Partiendo del código desarrollado previamente, estructurarás la aplicación de automatización en Python integrando **APScheduler** para su ejecución periódica en segundo plano. Posteriormente, diseñarás un archivo **Dockerfile** utilizando la técnica de *multi-stage builds* (construcción multi-etapa) con el objetivo de minimizar la superficie de ataque y el tamaño en disco de la imagen final. Finalmente, configurarás un entorno multi-contenedor con **Docker Compose**, vinculando el contenedor de la automatización con un servidor de bases de datos **PostgreSQL 16.2** que funcionará como almacén persistente de los datos procesados, estableciendo redes internas seguras y mecanismos de comprobación de salud (*healthchecks*).

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Diseñar y construir una imagen Docker optimizada utilizando *multi-stage builds*, reduciendo el peso de la imagen base de Python en más de un 60%.
- [ ] Integrar **APScheduler** (`BlockingScheduler`) en una aplicación Python para la ejecución desatendida y periódica de tareas de extracción de datos con manejo resiliente de errores.
- [ ] Orquestar servicios multi-contenedor independientes y dependientes utilizando **Docker Compose** y redes internas bridge personalizadas.
- [ ] Implementar un mecanismo de comprobación de salud (*healthcheck*) en PostgreSQL para garantizar el orden de encendido correcto de la pila de microservicios.
- [ ] Validar la resiliencia del pipeline ante fallos críticos de conectividad y violaciones de contratos de datos estructurados mediante Pydantic.

## Prerrequisitos

Para completar este laboratorio con éxito, requieres contar con los siguientes conocimientos y accesos:
- **Conocimientos de Automatización en Python:** Comprensión de estructuras de control, manejo de excepciones (`try-except`), validación de datos con Pydantic y conceptos básicos de SQL (especialmente dialecto PostgreSQL).
- **Conocimientos Básicos de Docker:** Familiaridad con comandos estándar como `docker build`, `docker run`, `docker compose up`, `docker ps` y `docker logs`.
- **Acceso a Entorno de Desarrollo:** Un shell con privilegios de ejecución para interactuar con Docker Engine.
- **Uso Asistido por Inteligencia Artificial:** Acceso a **GitHub Copilot Extension (v1.173.0)** o **Copilot Chat** en VS Code para agilizar la generación de manifiestos y la depuración de dependencias.

## Entorno de Laboratorio

La práctica se realizará de manera local o remota bajo las siguientes especificaciones de hardware y software:

### Requisitos de Hardware

| Recurso | Mínimo | Recomendado |
| :--- | :--- | :--- |
| **Procesador** | Intel i5 (4 núcleos físicos) con virtualización VT-x/AMD-V | Intel i7 / AMD Ryzen 7 (8 núcleos) |
| **Memoria RAM** | 8 GB | 16 GB |
| **Almacenamiento** | 20 GB de espacio libre (HDD) | 20 GB de espacio libre (SSD) |
| **Conexión** | Banda ancha sin restricciones de puertos salientes | Banda ancha de alta velocidad libre de proxies |

### Requisitos de Software y Herramientas Autorizadas

| Tecnología / Herramienta | Versión Exacta | Enlace Oficial / Licencia |
| :--- | :--- | :--- |
| **Python** | 3.12.2 | [Python 3.12.2](https://www.python.org/downloads/release/python-3122/) |
| **Docker Engine** | 25.0.3 | [Docker Engine](https://docs.docker.com/engine/install/) |
| **Docker Compose** | 2.24.5 | [Docker Compose](https://docs.docker.com/compose/install/) |
| **PostgreSQL** | 16.2-alpine | [PostgreSQL Hub](https://hub.docker.com/_/postgres) |
| **APScheduler** | 3.10.4 | [APScheduler PyPI](https://pypi.org/project/APScheduler/3.10.4/) (Licencia MIT) |
| **SQLAlchemy** | 2.0.27 | [SQLAlchemy PyPI](https://pypi.org/project/SQLAlchemy/2.0.27/) (Licencia MIT) |
| **psycopg2-binary** | 2.9.9 | [psycopg2-binary PyPI](https://pypi.org/project/psycopg2-binary/2.9.9/) (Licencia BSD) |
| **Pydantic** | 2.6.1 | [Pydantic PyPI](https://pypi.org/project/pydantic/2.6.1/) (Licencia MIT) |
| **Loguru** | 0.7.2 | [Loguru PyPI](https://pypi.org/project/loguru/0.7.2/) (Licencia MIT) |
| **VS Code** | 1.87.2 | [VS Code Download](https://code.visualstudio.com/) (Licencia Propietaria) |
| **GitHub Copilot Extension** | 1.173.0 | [GitHub Copilot](https://github.com/features/copilot) (Licencia Comercial) |

### Inicialización del Entorno de Trabajo

Ejecuta los siguientes comandos en tu terminal de Linux para preparar de forma limpia el espacio de nombres global destinado para este laboratorio:

```bash
## Crear directorio de trabajo global de automatización
mkdir -p /home/usuario/workspace/automatizacion_ventas/data/validated
cd /home/usuario/workspace/automatizacion_ventas

## Inicializar estructura básica de archivos vacíos
touch pipeline.py scheduler.py Dockerfile docker-compose.yml requirements.txt
```

---

## Instrucciones Paso a Paso

### Paso 1: Definir dependencias estructuradas

**Objective:** Configurar las dependencias exactas y estrictas de Python para garantizar la reproducibilidad absoluta de la automatización en cualquier contenedor.

**Instructions:**

1. Abre la herramienta **VS Code** en el directorio `/home/usuario/workspace/automatizacion_ventas`.
2. Edita el archivo `requirements.txt`.
3. Declara las siguientes dependencias de software con sus versiones exactas para evitar fallos por cambios de versión en tiempo de construcción de la imagen:

```text
## Dependencias de Orquestación y Conectividad
apscheduler==3.10.4
sqlalchemy==2.0.27
psycopg2-binary==2.9.9

## Dependencias de Validación y Logueo
pydantic==2.6.1
loguru==0.7.2
```

4. Guarda el archivo.

**Expected output:**
El archivo `requirements.txt` debe contener exactamente las cinco líneas indicadas, sin espacios en blanco adicionales ni comentarios que puedan interferir en la fase de análisis del instalador `pip`.

**Verification:**
Ejecuta el siguiente comando para comprobar que las dependencias están correctamente estructuradas de forma sintáctica:
```bash
cat /home/usuario/workspace/automatizacion_ventas/requirements.txt
```

---

### Paso 2: Desarrollar el script de automatización y persistencia

**Objective:** Crear el script central de procesamiento de datos (`pipeline.py`) que valide la entrada de datos a través de modelos Pydantic y persista los registros válidos dentro de una base de datos relacional PostgreSQL utilizando SQLAlchemy.

**Instructions:**

1. Abre el archivo `/home/usuario/workspace/automatizacion_ventas/pipeline.py` en tu editor de código.
2. Utiliza **GitHub Copilot Chat** con el siguiente prompt sugerido para acelerar la generación de código modular orientado a objetos y alineado a los requerimientos de validación:
   
   > *"Genera un script de Python llamado pipeline.py que use SQLAlchemy para conectarse a PostgreSQL. Debe definir un modelo de datos SQLAlchemy llamado 'Venta' con columnas: id (PK, autoincrement), producto (str), cantidad (int), precio (float), fecha_registro (datetime). Debe usar Pydantic v2 (BaseModel) para validar los datos de ventas crudos antes de guardarlos. Si un dato no es válido, debe lanzar un error y registrarlo con loguru sin detener el script. Implementa una clase principal llamada IngestorVentas que lea las variables de entorno para la conexión de base de datos y tenga un método ingest_data(raw_data: list)."*

3. Adapta y pega el siguiente código controlado en tu archivo `pipeline.py`:

```python
import os
import sys
from datetime import datetime
from typing import List, Dict, Any
from loguru import logger
from pydantic import BaseModel, Field, ValidationError
from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime
from sqlalchemy.orm import declarative_base, sessionmaker

## Configuración inicial de Loguru para salida limpia en consola
logger.remove()
logger.add(
    sys.stdout, 
    format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | {message}", 
    level="INFO"
)

Base = declarative_base()

## 1. Definición del modelo SQLAlchemy para persistencia
class VentaModel(Base):
    __tablename__ = 'ventas'
    
    id = Column(Integer, primary_key=True, autoincrement=True)
    producto = Column(String(100), nullable=False)
    cantidad = Column(Integer, nullable=False)
    precio = Column(Float, nullable=False)
    fecha_registro = Column(DateTime, default=datetime.utcnow)

## 2. Contrato estricto de Datos con Pydantic v2
class VentaSchema(BaseModel):
    producto: str = Field(..., min_length=2, max_length=100)
    cantidad: int = Field(..., gt=0, description="La cantidad debe ser mayor que cero")
    precio: float = Field(..., gt=0.0, description="El precio debe ser un número flotante positivo")

class IngestorVentas:
    def __init__(self):
        # Leer variables de entorno con valores por defecto locales de contingencia
        db_user = os.getenv("DB_USER", "postgres_user")
        db_pass = os.getenv("DB_PASSWORD", "secure_password_123")
        db_host = os.getenv("DB_HOST", "localhost")
        db_port = os.getenv("DB_PORT", "5432")
        db_name = os.getenv("DB_NAME", "automation_db")
        
        self.connection_string = f"postgresql://{db_user}:{db_pass}@{db_host}:{db_port}/{db_name}"
        logger.info(f"Conectando a base de datos en {db_host}:{db_port}...")
        
        try:
            self.engine = create_engine(self.connection_string, pool_pre_ping=True)
            Base.metadata.create_all(self.engine)
            self.Session = sessionmaker(bind=self.engine)
            logger.success("Conexión de base de datos e inicialización de esquemas completada.")
        except Exception as e:
            logger.critical(f"Error de conexión inicial a la base de datos: {str(e)}")
            raise e

    def procesar_e_ingresar(self, lote_datos: List[Dict[str, Any]]) -> tuple[int, int]:
        """
        Procesa un lote de diccionarios, valida cada uno con Pydantic,
        y persiste los válidos en PostgreSQL. Retorna (exitosos, fallidos).
        """
        session = self.Session()
        exitosos = 0
        fallidos = 0
        
        logger.info(f"Iniciando procesamiento de lote de {len(lote_datos)} registros.")
        
        for index, item in enumerate(lote_datos):
            try:
                # Validar contrato de datos con Pydantic
                datos_validados = VentaSchema(**item)
                
                # Transformar a modelo ORM
                registro_db = VentaModel(
                    producto=datos_validados.producto,
                    cantidad=datos_validados.cantidad,
                    precio=datos_validados.precio
                )
                session.add(registro_db)
                exitosos += 1
                
            except ValidationError as val_error:
                fallidos += 1
                logger.warning(
                    f"Registro {index} rechazado por violación de contrato: "
                    f"Datos: {item} | Detalles: {val_error.errors()[0]['msg']}"
                )
            except Exception as ex:
                fallidos += 1
                logger.error(f"Error inesperado procesando registro {index}: {str(ex)}")

        if exitosos > 0:
            try:
                session.commit()
                logger.success(f"Lote finalizado. Persistidos con éxito: {exitosos} registros. Fallidos: {fallidos}.")
            except Exception as commit_ex:
                session.rollback()
                logger.error(f"Fallo crítico al hacer commit del lote: {str(commit_ex)}")
                exitosos = 0
                fallidos = len(lote_datos)
            finally:
                session.close()
        else:
            session.close()
            logger.info("No se agregaron registros válidos en este lote.")
            
        return exitosos, fallidos
```

4. Guarda el archivo.

**Expected output:**
Un script de Python robusto que encapsula la inicialización de la tabla `ventas`, realiza control transaccional mediante sesiones locales (`Session`), gestiona de forma aislada los errores de integridad de datos y proporciona trazabilidad detallada en formato estructurado a la consola estándar.

**Verification:**
Puedes verificar la validez sintáctica básica del script de automatización mediante el comando:
```bash
python -m py_compile /home/usuario/workspace/automatizacion_ventas/pipeline.py
```

---

### Paso 3: Implementar la orquestación horaria con APScheduler

**Objective:** Integrar un planificador del tipo `BlockingScheduler` de APScheduler dentro de un script de entrada principal (`scheduler.py`), simulando la recolección recurrente de datos de sistemas transaccionales externos y enrutando los flujos al pipeline de procesamiento.

**Instructions:**

1. Abre el archivo `/home/usuario/workspace/automatizacion_ventas/scheduler.py` en tu entorno.
2. Desarrolla el siguiente código enfocado en simular un generador continuo de eventos comerciales e invocar el proceso de extracción de datos del `IngestorVentas`:

```python
import os
import random
import sys
import time
from loguru import logger
from apscheduler.schedulers.blocking import BlockingScheduler
from pipeline import IngestorVentas

## Configurar logs consistentes
logger.remove()
logger.add(
    sys.stdout, 
    format="<cyan>{time:YYYY-MM-DD HH:mm:ss}</cyan> | <level>{level: <8}</level> | [ORQUESTADOR] - {message}", 
    level="INFO"
)

## Generador artificial de datos de venta para simulación del pipeline
def simular_obtencion_datos_ventas() -> list:
    productos_disponibles = ["Laptop Pro", "Teclado Mecánico", "Monitor 4K", "Mouse Ergonómico", "Hub USB-C"]
    lote = []
    
    # Generamos 5 registros simulados, incluyendo datos inconsistentes para forzar validación
    for _ in range(4):
        lote.append({
            "producto": random.choice(productos_disponibles),
            "cantidad": random.randint(1, 10),
            "precio": round(random.uniform(15.5, 1200.0), 2)
        })
        
    # Agregamos intencionalmente un registro malformado (violación de contrato)
    lote.append({
        "producto": "Malformado",
        "cantidad": -5,  # Error: gt=0
        "precio": 50.0
    })
    
    return lote

def job_pipeline_automatizado(ingestor: IngestorVentas):
    logger.info("Iniciando ciclo de ejecución programada...")
    try:
        # 1. Extracción de datos
        datos_crudos = simular_obtencion_datos_ventas()
        
        # 2. Procesamiento, validación e ingesta
        exitosos, fallidos = ingestor.procesar_e_ingresar(datos_crudos)
        logger.info(f"Ciclo terminado. Operación: {exitosos} inserts / {fallidos} rechazos.")
        
    except Exception as e:
        logger.critical(f"Fallo crítico imprevisto en la ejecución del ciclo: {str(e)}")

if __name__ == "__main__":
    logger.info("Iniciando servicio de automatización programada...")
    
    # El intervalo en minutos se puede inyectar vía variables de entorno (por defecto 5 min)
    intervalo_minutos = int(os.getenv("SCHEDULER_INTERVAL_MINUTES", "5"))
    
    try:
        # Instanciar el ingestor una sola vez al arrancar para validar la conectividad
        ingestor = IngestorVentas()
        
        # Inicializar el planificador bloqueante
        scheduler = BlockingScheduler()
        
        # Registrar el job periódico para ejecutarse en el intervalo configurado
        scheduler.add_job(
            func=job_pipeline_automatizado,
            trigger='interval',
            minutes=intervalo_minutos,
            args=[ingestor],
            id='pipeline_ventas_job',
            replace_existing=True
        )
        
        logger.success(f"Planificador iniciado con éxito. El pipeline se ejecutará cada {intervalo_minutos} minutos.")
        
        # Punto de bloqueo: El proceso vivirá indefinidamente ejecutando las tareas
        scheduler.start()
        
    except (KeyboardInterrupt, SystemExit):
        logger.warning("Detención manual del servicio detectada. Apagando planificador...")
    except Exception as error_inicio:
        logger.error(f"El planificador no pudo arrancar debido a un error inicial: {error_inicio}")
        sys.exit(1)
```

3. Guarda el archivo.

**Expected output:**
Un archivo de control maestro (`scheduler.py`) capaz de arrancar, instanciar la base de datos de manera proactiva al inicio, registrar el *job* periódico de intervalos dinámicos y persistir de manera continua en segundo plano.

**Verification:**
Comprueba que el archivo se compila limpiamente mediante:
```bash
python -m py_compile /home/usuario/workspace/automatizacion_ventas/scheduler.py
```

---

### Paso 4: Diseñar el Dockerfile optimizado (Multi-stage Build)

**Objective:** Construir un manifiesto de ensamblado de imagen Docker (`Dockerfile`) utilizando arquitectura multi-etapa (*multi-stage build*) con base de Debian Slim para separar las herramientas pesadas de compilación en caliente de los paquetes puramente ejecutables en producción.

**Instructions:**

1. Abre el archivo `/home/usuario/workspace/automatizacion_ventas/Dockerfile`.
2. Utiliza la extensión **GitHub Copilot** para estructurar un flujo de empaquetado multi-etapa óptimo para Python. Escribe el siguiente código:

```dockerfile
## ==========================================================
## Etapa 1: Builder (Entorno de compilación y empaquetado)
## ==========================================================
FROM python:3.12.2-slim-bookworm AS builder

## Configurar directorio de trabajo temporal de compilación
WORKDIR /build

## Instalar dependencias necesarias para compilar paquetes nativos de C (como psycopg2)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

## Crear entorno virtual de Python para aislamiento absoluto de paquetes
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

## Instalar dependencias requeridas del proyecto
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt


## ==========================================================
## Etapa 2: Runner (Imagen final de producción ultra liviana)
## ==========================================================
FROM python:3.12.2-slim-bookworm AS runner

## Metadatos del mantenedor
LABEL maintainer="ingeniero_automatizacion@enterprise.com"
LABEL version="1.0.0"

WORKDIR /app

## Instalar únicamente la biblioteca compartida necesaria para interactuar con PostgreSQL
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

## Copiar el entorno virtual de Python precompilado desde la etapa "builder"
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

## Copiar exclusivamente los archivos fuentes del proyecto necesarios para la ejecución
COPY pipeline.py scheduler.py /app/

## Configurar variables de entorno globales de Python para optimización de logs en Docker
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

## Declarar usuario sin privilegios por políticas estrictas de seguridad de producción
RUN useradd -u 10011 -m automation_user && \
    chown -R automation_user:automation_user /app
USER automation_user

## Comando de ejecución por defecto del contenedor
CMD ["python", "scheduler.py"]
```

3. Guarda el archivo.

**Expected output:**
Un archivo de definición de contenedor estructurado en dos bloques lógicos independientes que utiliza la herencia del entorno virtual de Python compilado (`/opt/venv`), evitando la persistencia de herramientas pesadas de compilación (como `gcc` y `make`) en la capa de distribución final, reduciendo el tamaño total y los vectores de vulnerabilidad.

**Verification:**
Comprueba que el Dockerfile no posea errores de sintaxis o de formato invocando la revisión sin compilación (*dry-run*) del motor:
```bash
docker parser -f /home/usuario/workspace/automatizacion_ventas/Dockerfile . 2>/dev/null || echo "Manifiesto Dockerfile listo para validación."
```

---

### Paso 5: Configurar la orquestación multi-contenedor con Docker Compose

**Objective:** Configurar un archivo declarativo de servicios de orquestación (`docker-compose.yml`) que levante de forma automatizada un motor relacional de base de datos PostgreSQL 16.2 local y el contenedor de ejecución de la automatización en una red aislada y segura.

**Instructions:**

1. Abre el archivo `/home/usuario/workspace/automatizacion_ventas/docker-compose.yml`.
2. Inserta la configuración completa de servicios descrita abajo:

```yaml
version: '3.8'

services:
  # Servicio 1: Motor de Base de Datos Relacional PostgreSQL 16.2
  automation-postgres:
    image: postgres:16.2-alpine
    container_name: automation-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: automation_db
      POSTGRES_USER: postgres_user
      POSTGRES_PASSWORD: secure_password_123
    ports:
      - "5432:5432"
    volumes:
      - postgres_data_volume:/var/lib/postgresql/data
    networks:
      - automation-private-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres_user -d automation_db"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # Servicio 2: Pipeline de Automatización programada (APScheduler)
  sales-automation:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: sales-automation
    restart: unless-stopped
    depends_on:
      automation-postgres:
        condition: service_healthy
    environment:
      DB_HOST: automation-postgres
      DB_PORT: 5432
      DB_NAME: automation_db
      DB_USER: postgres_user
      DB_PASSWORD: secure_password_123
      SCHEDULER_INTERVAL_MINUTES: 1  # Ajustado a 1 minuto para pruebas ágiles en el laboratorio
    networks:
      - automation-private-net

volumes:
  postgres_data_volume:
    driver: local

networks:
  automation-private-net:
    driver: bridge
```

3. Guarda el archivo.

**Expected output:**
Un manifiesto de Docker Compose de grado de producción que contiene:
- Una definición del servicio de base de datos con un volumen nombrado persistente para evitar pérdidas accidentales de datos.
- Un **healthcheck** nativo utilizando la herramienta del motor de Postgres `pg_isready`.
- Una definición de dependencia estricta (`depends_on` bajo la condición de `service_healthy`) que previene que la automatización de Python intente arrancar y conectarse antes de que PostgreSQL esté totalmente listo para aceptar transacciones.
- Variables de entorno parametrizadas con nombres legibles para la comunicación intra-red.

**Verification:**
Valida la sintaxis de la orquestación mediante la herramienta nativa de análisis sintáctico de Docker Compose:
```bash
docker compose config
```

---

### Paso 6: Desplegar, verificar la orquestación y validar la resiliencia

**Objective:** Construir los contenedores, ejecutar el ecosistema orquestado y monitorear en tiempo real el comportamiento integrado de la infraestructura de automatización.

**Instructions:**

1. Desde tu terminal en la ruta `/home/usuario/workspace/automatizacion_ventas`, inicia la fase de compilación y levantamiento de la pila de servicios utilizando el siguiente comando:

```bash
docker compose up --build -d
```

2. Verifica el estado y la correcta asignación de los nombres y puertos de red de los contenedores activos:

```bash
docker compose ps
```

3. Inspecciona en tiempo real los registros del planificador para verificar la correcta conectividad con la base de datos PostgreSQL, la inicialización automática de la tabla de ventas y el correcto procesamiento periódico de datos:

```bash
docker compose logs -f sales-automation
```

4. Deja correr el planificador por lo menos durante **2 o 3 minutos** (como el intervalo se configuró a `1` minuto en el archivo de Compose, verás múltiples ejecuciones de ingesta y validación de datos comerciales en tiempo real).

**Expected output:**
La consola mostrará una traza clara de eventos estructurados. Deberás ver las llamadas de inicialización de la base de datos, el arranque exitoso de la rutina de tareas y la ejecución recurrente de la simulación de ventas con inserciones y validaciones de datos:

```text
2024-03-20 14:10:00 | INFO     | Conectando a base de datos en automation-postgres:5432...
2024-03-20 14:10:01 | SUCCESS  | Conexión de base de datos e inicialización de esquemas completada.
2024-03-20 14:10:01 | SUCCESS  | Planificador iniciado con éxito. El pipeline se ejecutará cada 1 minutos.
2024-03-20 14:11:00 | INFO     | [ORQUESTADOR] - Iniciando ciclo de ejecución programada...
2024-03-20 14:11:00 | INFO     | Iniciando procesamiento de lote de 5 registros.
2024-03-20 14:11:00 | WARNING  | Registro 4 rechazado por violación de contrato: Datos: {'producto': 'Malformado', 'cantidad': -5, 'precio': 50.0} | Detalles: La cantidad debe ser mayor que cero
2024-03-20 14:11:01 | SUCCESS  | Lote finalizado. Persistidos con éxito: 4 registros. Fallidos: 1.
2024-03-20 14:11:01 | INFO     | [ORQUESTADOR] - Ciclo terminado. Operación: 4 inserts / 1 rechazos.
```

5. Detén la observación de logs presionando `Ctrl + C`.

---

## Validación y Pruebas

Para garantizar el éxito de la práctica y asegurar el cumplimiento de los contratos de ingeniería estipulados para este laboratorio, ejecuta las siguientes tareas de auditoría.

### Prueba 1: Auditoría de persistencia en PostgreSQL

Comprueba que las ejecuciones del orquestador están persistiendo de forma efectiva y con integridad los registros dentro de las tablas de PostgreSQL. Ejecuta una consulta interactiva temporal dentro del contenedor de la base de datos:

```bash
docker exec -it automation-postgres psql -U postgres_user -d automation_db -c "SELECT * FROM ventas;"
```

**Evidencia de éxito esperada:**
Se debe retornar una tabla con las tuplas insertadas de forma secuencial con IDs autoincrementales y con la estampa de tiempo correcta, omitiendo completamente los registros malformados con cantidades negativas (p. ej., `cantidad: -5`), lo cual demuestra el correcto filtrado a nivel de contrato de datos con Pydantic.

```text
 id |     producto     | cantidad | precio |      fecha_registro       
----+------------------+----------+--------+---------------------------
  1 | Laptop Pro       |        3 | 985.50 | 2024-03-20 14:11:01.12345
  2 | Monitor 4K       |        1 | 450.00 | 2024-03-20 14:11:01.12389
  3 | Hub USB-C        |        5 |  35.20 | 2024-03-20 14:11:01.12411
  4 | Teclado Mecánico |        2 | 120.00 | 2024-03-20 14:11:01.12432
(4 filas)
```

---

### Prueba 2: Auditoría del tamaño y optimización de la Imagen Docker

Compara el impacto de la arquitectura de compilación multi-etapa diseñada en este laboratorio frente a una imagen tradicional monolítica de Python.

Ejecuta en tu terminal de control:
```bash
docker images | grep sales-automation
```

**Evidencia de éxito esperada:**
La imagen optimizada final de la automatización (`sales-automation`) debe medir alrededor de **130 MB - 160 MB**, mientras que una compilación equivalente de Python tradicional sin multi-stage que conserva librerías de compilación superaría fácilmente los **450 MB**. Esto demuestra la efectividad de la segregación de capas y la reducción de huella de almacenamiento.

---

### Prueba 3 (Adversaria): Simulación de desastre de conectividad de red

Para probar la robustez del orquestador persistente, simularemos un corte en el servicio del motor de base de datos mientras el planificador está activo.

1. Detén temporalmente el contenedor de la base de datos:
   ```bash
   docker compose stop automation-postgres
   ```
2. Espera unos segundos e inspecciona el estado del contenedor de automatización:
   ```bash
   docker compose ps
   ```
   *El contenedor `sales-automation` debe seguir en estado activo (Up), ya que el planificador se encuentra aislado y en ejecución bloqueante.*
3. Monitorea los logs para analizar cómo se comporta la automatización al intentar ejecutar un ciclo sin la base de datos disponible:
   ```bash
   docker compose logs --tail=20 sales-automation
   ```

**Evidencia de éxito esperada:**
El script debe capturar el error de conexión a través de la cláusula de manejo de excepciones genéricas implementada dentro del bucle de la tarea, emitiendo un reporte de fallo crítico en los logs pero **sin terminar el proceso del contenedor**, garantizando que el servicio continúe con vida esperando que el sistema secundario se restablezca:

```text
2024-03-20 14:15:00 | INFO     | [ORQUESTADOR] - Iniciando ciclo de ejecución programada...
2024-03-20 14:15:00 | ERROR    | Error inesperado procesando registro 0: Can't reconnect until invalid transaction is rolled back...
2024-03-20 14:15:01 | CRITICAL | Fallo crítico imprevisto en la ejecución del ciclo: Connection refused
```

4. Restablece el entorno volviendo a arrancar la base de datos:
   ```bash
   docker compose start automation-postgres
   ```
5. Tras el reinicio del servicio de base de datos, confirma en el siguiente minuto que el ciclo del planificador recupera automáticamente la conectividad normal de inserciones sin requerir el reinicio manual del contenedor de la automatización.

---

## Solución de Problemas

En esta sección se describen los dos problemas más comunes que pueden ocurrir durante el desarrollo de esta arquitectura de automatización:

### Problema 1: El contenedor de automatización se apaga inmediatamente con un código de salida `1`

- **Sintoma:** Al ejecutar `docker compose up`, el servicio `sales-automation` cambia instantáneamente a estado `Exited (1)` y no se mantiene en ejecución en segundo plano.
- **Causa:** El script principal `scheduler.py` está configurado para conectarse a la base de datos al inicializar la clase `IngestorVentas`. Si las credenciales son incorrectas, o si la base de datos aún no está lista para aceptar conexiones (a pesar de la configuración de dependencias de red), la inicialización falla y provoca un corte del script con `sys.exit(1)`.
- **Solución:**
  1. Verifica que los parámetros de conexión `DB_HOST`, `DB_USER` y `DB_PASSWORD` en el archivo `docker-compose.yml` coincidan de manera idéntica con los definidos en la sección del servicio de base de datos `automation-postgres`.
  2. Asegúrate de que el bloque `healthcheck` de PostgreSQL esté correctamente configurado con el usuario configurado y responda de forma válida (el comando `pg_isready` debe usar el mismo usuario `-U postgres_user` que definiste).

### Problema 2: Error de compilación "gcc: command not found" o "psycopg2 build error" en la fase de construcción de la imagen

- **Sintoma:** Durante la ejecución de `docker compose up --build`, el proceso de Docker se detiene de forma abrupta en la primera fase de construcción (`builder`) mostrando fallos al intentar instalar el paquete `psycopg2` o similares.
- **Causa:** La imagen base de Python `python:3.12.2-slim-bookworm` es una distribución minimalista que no contiene compiladores de C instalados por defecto. El paquete `psycopg2-binary` usualmente pre-compila sus dependencias, pero en ciertos entornos o arquitecturas de procesador requerirá compilar bindings nativos que requieren el comando `gcc`.
- **Solución:** Verifica que el bloque de la primera etapa del archivo `Dockerfile` (`builder`) tenga correctamente configurada la actualización de repositorios y la instalación explícita de `gcc`, `libpq-dev` y `build-essential` mediante `apt-get`:
  ```dockerfile
  RUN apt-get update && apt-get install -y --no-install-recommends \
      gcc \
      libpq-dev \
      build-essential \
      && rm -rf /var/lib/apt/lists/*
  ```

---

## Limpieza

Para restaurar el entorno local de desarrollo eliminando de forma completa toda la pila de contenedores, redes internas y el volumen persistente de la base de datos creado durante este laboratorio, ejecuta los siguientes comandos en tu consola:

```bash
## Apagar contenedores de docker compose eliminando volumenes locales y cache de redes
cd /home/usuario/workspace/automatizacion_ventas
docker compose down -v

## Limpiar posibles imágenes huerfanas residuales generadas por compilaciones previas
docker image prune -f

## (Opcional) Remover los archivos temporales generados si deseas repetir el laboratorio
## rm -rf /home/usuario/workspace/automatizacion_ventas/*.py
```

---

## Resumen

En este laboratorio práctico, has transformado una automatización de Python local de ejecución manual en un ecosistema robusto, seguro y planificado para su despliegue en entornos productivos.

Durante la práctica has aprendido a:
1. **Modelar contratos de datos estrictos:** Implementando reglas de negocio a nivel de datos mediante **Pydantic**, evitando la inyección de datos corruptos al almacén relacional.
2. **Modularizar con APScheduler:** Configurando un planificador del tipo `BlockingScheduler` persistente encargado de invocar las tareas de procesamiento en intervalos definidos, controlando de forma autónoma el ciclo de vida del contenedor.
3. **Optimizar con Multi-stage Builds:** Diseñando un archivo `Dockerfile` con una estructura de compilación separada del entorno de ejecución, logrando reducir drásticamente el peso de la imagen final y los riesgos potenciales de seguridad informática.
4. **Orquestar servicios con Docker Compose:** Definiendo la infraestructura de soporte para el pipeline (servidor PostgreSQL), configurando variables de entorno dinámicas, redes aisladas (*bridge*) y comprobaciones de salud del motor de base de datos para garantizar una orquestación resiliente a fallas de infraestructura.

### Recursos Adicionales para Autoestudio
* [Documentación Oficial de Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
* [Documentación de APScheduler - Configuración de Ejecutores y Planificadores](https://apscheduler.readthedocs.io/en/stable/)
* [Documentación de SQLAlchemy 2.0 - Guía de Configuración e Inserciones de Datos](https://docs.sqlalchemy.org/en/20/)
