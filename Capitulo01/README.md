# Práctica: De un proceso manual a una automatización mantenible

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 90 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar (Apply) |

---

## Descripción General

En este laboratorio, transformarás un script de automatización inestable y monolítico ("código espagueti") en una arquitectura de automatización limpia, modular y orientada a objetos (OOP) en Python. Trabajarás sobre un escenario real: un script llamado `spaghetti_extractor.py` que descarga datos transaccionales mediante peticiones HTTP inestables, aplica reglas de negocio de forma procedural y guarda archivos sin control de errores ni registros de auditoría (logging). 

Aprenderás a descomponer este bloque monolítico en clases especializadas para cada responsabilidad del patrón ETL (Extracción, Transformación y Carga), a manejar la configuración mediante variables de entorno y archivos JSON, y a establecer un sistema de logging estructurado que garantice la observabilidad del proceso. Al finalizar, este framework modular servirá como la base sólida sobre la cual se construirán las siguientes etapas del pipeline de datos.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Identificar puntos de falla comunes, acoplamientos rígidos y cuellos de botella en scripts procedurales tradicionales.
- [ ] Refactorizar flujos secuenciales complejos aplicando principios de Programación Orientada a Objetos (OOP) en Python.
- [ ] Implementar un módulo de configuración dinámico y seguro que integre archivos JSON y variables de entorno (`os.getenv`).
- [ ] Configurar un sistema de logging jerárquico y robusto utilizando la librería estándar de Python para auditoría en tiempo de ejecución.
- [ ] Desarrollar mecanismos de reintento con retardo (backoff) para superar fallas intermitentes de red en descargas de archivos de datos.

---

## Prerrequisitos

Para realizar este laboratorio con éxito, necesitas contar con:
1. **Conocimientos conceptuales**:
   - Familiaridad con la Programación Orientada a Objetos (OOP) en Python (clases, métodos, herencia, encapsulamiento).
   - Manejo intermedio de tipos de datos en Python (`dict`, `list`, `str`) y estructuras de control.
   - Entendimiento básico del protocolo HTTP (códigos de estado como 200, 404, 500 y excepciones de red).
2. **Acceso y Herramientas**:
   - Una terminal de comandos (Bash o PowerShell).
   - Editor de código VS Code con la extensión oficial de Python instalada.
   - Licencia activa de GitHub Copilot (Individual, Business o Enterprise) configurada en VS Code mediante la extensión oficial (v1.173.0 o superior) para asistencia en la refactorización si es requerido.

---

## Entorno de Laboratorio

Este laboratorio se realiza en el directorio global asignado a la suite de automatizaciones. A continuación se detallan las especificaciones de hardware, software y comandos de inicialización:

### Especificaciones de Hardware y Entorno

| Componente | Requisito Mínimo | Requisito Recomendado |
| :--- | :--- | :--- |
| **Procesador** | Intel i5 o equivalente (4 núcleos físicos) | Intel i7 / AMD Ryzen 5 o superior |
| **Memoria RAM** | 8 GB RAM | 16 GB RAM |
| **Almacenamiento** | 20 GB de espacio libre (SSD preferiblemente) | 20 GB de espacio libre (SSD) |
| **Conectividad** | Acceso ilimitado a internet (HTTPS) | Acceso ilimitado a internet (HTTPS) |

### Especificaciones de Software y Versiones Exactas

| Software / Dependencia | Versión Exacta | Arquitectura / Distribución | Enlace de Descarga / Fuente Oficial |
| :--- | :--- | :--- | :--- |
| **Python** | 3.12.2 | CPython x86_64 / ARM64 | [Python Release 3.12.2](https://www.python.org/downloads/release/python-3122/) |
| **pip** | 24.0 | Integrado con Python | [Pip Installation Guide](https://pip.pypa.io/en/stable/installation/) |
| **requests** | 2.31.0 | Librería de terceros (PyPI) | [Requests PyPI](https://pypi.org/project/requests/2.31.0/) |
| **VS Code** | 1.87.2 | IDE de desarrollo multiplataforma | [VS Code February 2024](https://code.visualstudio.com/updates/v1_87) |
| **GitHub Copilot Ext.** | 1.173.0 | Extensión de VS Code | [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) |

### Inicialización de Estructura de Directorios

Ejecuta los siguientes comandos en tu terminal para preparar el espacio de trabajo global `/workspace/python-automation` y asegurar el entorno virtual aislado:

```bash
## 1. Crear la estructura de directorios requerida
mkdir -p /workspace/python-automation/config
mkdir -p /workspace/python-automation/data/validated
mkdir -p /workspace/python-automation/logs
mkdir -p /workspace/python-automation/src/utils

## 2. Situarse en el directorio de trabajo
cd /workspace/python-automation

## 3. Crear y activar el entorno virtual aislado de Python 3.12.2
python3.12 -m venv venv
source venv/bin/activate  # En Windows usa: .\venv\Scripts\activate

## 4. Actualizar pip e instalar la versión exacta de las dependencias
pip install --upgrade pip==24.0
pip install requests==2.31.0
```

---

## Instrucciones Paso a Paso

### Paso 1: Configurar el Servidor Inestable de Prueba y el Script Monolítico Base

**Objetivo**: Crear un simulador de servidor HTTP que devuelva errores intermitentes de red y un script procedural "espagueti" que intente descargar y procesar los datos para visualizar de manera clara sus puntos de falla.

**Instrucciones**:

1. Crea el archivo del servidor de prueba en `/workspace/python-automation/unstable_server.py`. Este script ejecutará un servidor HTTP local básico que falla intencionalmente de forma intermitente (retornando errores 500 el 50% de las veces) para simular inestabilidad de red:

```python
## File: /workspace/python-automation/unstable_server.py
import http.server
import socketserver
import random

PORT = 8080
FAIL_RATE = 0.5  # 50% de probabilidad de fallo de red simulado

class UnstableHandler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/ventas.csv":
            if random.random() < FAIL_RATE:
                self.send_response(500)
                self.send_header("Content-type", "text/plain")
                self.end_headers()
                self.wfile.write(b"Internal Server Error (Simulado)")
                print("[-] Servidor Simuló: Error 500.")
            else:
                self.send_response(200)
                self.send_header("Content-type", "text/csv")
                self.end_headers()
                csv_data = (
                    "id,fecha,monto,cliente\n"
                    "1,2023-10-01,150.50,Cliente_A\n"
                    "2,2023-10-02,300.00,Cliente_B\n"
                    "3,2023-10-02,INVALID_AMOUNT,Cliente_C\n"
                    "4,2023-10-03,450.75,Cliente_D\n"
                )
                self.wfile.write(csv_data.encode("utf-8"))
                print("[+] Servidor Simuló: Respuesta 200 OK.")
        else:
            self.send_response(404)
            self.end_headers()

if __name__ == "__main__":
    socketserver.TCPServer.allow_reuse_address = True
    with socketserver.TCPServer(("", PORT), UnstableHandler) as httpd:
        print(f"[*] Servidor de simulación inestable corriendo en el puerto {PORT}...")
        try:
            httpd.serve_forever()
        except KeyboardInterrupt:
            print("\n[*] Deteniendo servidor.")
```

2. Ejecuta el servidor en segundo plano en una terminal separada:
```bash
python /workspace/python-automation/unstable_server.py
```

3. Crea el script monolítico heredado en `/workspace/python-automation/spaghetti_extractor.py`. Este script representa las malas prácticas comunes: rutas estáticas incrustadas ("hardcoded"), nulo manejo de excepciones, uso de impresiones en pantalla para depurar en lugar de logging estructurado, y parseo manual frágil de cadenas sin manejo de tipos corruptos:

```python
## File: /workspace/python-automation/spaghetti_extractor.py
import requests
import os

## CONFIGURACIÓN MANUAL RÍGIDA
URL = "http://localhost:8080/ventas.csv"
OUTPUT_FILE = "/workspace/python-automation/data/validated/datos_manuales.csv"

print("Iniciando proceso...")

## Extracción frágil sin reintentos ni control de timeouts
response = requests.get(URL)

if response.status_code == 200:
    data = response.text
    print("Datos descargados correctamente.")
    
    # Procesamiento / Transformación procedural frágil
    lines = data.strip().split("\n")
    headers = lines[0].split(",")
    rows = []
    
    for line in lines[1:]:
        cols = line.split(",")
        # Conversión de tipo directa sin validación de nulos o datos corruptos
        transaction_id = int(cols[0])
        fecha = cols[1]
        monto = float(cols[2]) # Esto fallará en el registro 3
        cliente = cols[3]
        rows.append(f"{transaction_id},{fecha},{monto},{cliente}")
        
    # Escritura sin validar estructura de directorios
    with open(OUTPUT_FILE, "w") as f:
        f.write(",".join(headers) + "\n")
        for r in rows:
            f.write(r + "\n")
            
    print("Proceso completado exitosamente.")
else:
    print("Error catastrófico en la descarga.")
```

4. Ejecuta el script espagueti para experimentar sus fallas:
```bash
python /workspace/python-automation/spaghetti_extractor.py
```

**Resultado esperado de la ejecución**: El script fallará de dos formas alternantes según la inestabilidad simulada:
- Si el servidor devuelve error 500, el script imprime un texto básico `"Error catastrófico en la descarga."` pero no arroja código de salida adecuado, ni reintenta la petición.
- Si el servidor devuelve 200 OK, el script se interrumpe abruptamente arrojando un error de tipo `ValueError: could not convert string to float: 'INVALID_AMOUNT'` debido a que el registro 3 contiene una cadena inválida en lugar de un flotante, deteniendo el flujo completo de forma destructiva sin salvar los registros correctos anteriores.

---

### Paso 2: Diseñar el Módulo de Configuración Dinámica y Logger Estructurado

**Objetivo**: Implementar un administrador de configuración jerárquico que lea archivos de configuración JSON de forma segura y que permita la sobreescritura de parámetros mediante variables de entorno del sistema operativo, junto con una infraestructura de logging robusta que guarde trazas formateadas en consola y en archivo de forma asíncrona.

**Instrucciones**:

1. Crea el archivo de configuración base JSON en `/workspace/python-automation/config/settings.json`:

```json
{
  "ENVIRONMENT": "development",
  "API_URL": "http://localhost:8080/ventas.csv",
  "MAX_RETRIES": 3,
  "BACKOFF_FACTOR": 2,
  "TIMEOUT_SECONDS": 5,
  "OUTPUT_PATH": "/workspace/python-automation/data/validated/ventas_procesadas.csv"
}
```

2. Crea el módulo de gestión de configuración segura en `/workspace/python-automation/src/utils/config.py`. Este módulo utiliza variables de entorno mediante la librería estándar `os` para sobreescribir valores estáticos del archivo JSON, ideal para pipelines CI/CD y despliegues en contenedores:

```python
## File: /workspace/python-automation/src/utils/config.py
import os
import json
from pathlib import Path

class ConfigManager:
    """Administrador de configuración jerárquica para la suite de automatización."""
    
    def __init__(self, config_path: str = "/workspace/python-automation/config/settings.json"):
        self.config_path = Path(config_path)
        self.settings = {}
        self._load_defaults()
        self._load_from_json()
        self._override_with_env()

    def _load_defaults(self):
        """Carga de configuraciones por defecto en memoria."""
        self.settings = {
            "ENVIRONMENT": "production",
            "API_URL": "",
            "MAX_RETRIES": 3,
            "BACKOFF_FACTOR": 1.5,
            "TIMEOUT_SECONDS": 10,
            "OUTPUT_PATH": "/workspace/python-automation/data/validated/output.csv"
        }

    def _load_from_json(self):
        """Carga y parsea la configuración JSON si el archivo existe."""
        if self.config_path.exists():
            try:
                with open(self.config_path, "r", encoding="utf-8") as f:
                    json_data = json.load(f)
                    self.settings.update(json_data)
            except json.JSONDecodeError as e:
                # Si está corrupto, se mantiene con valores por defecto y se reporta
                print(f"[ALERTA CONFIG] Error parseando JSON de configuración: {e}. Usando valores por defecto.")

    def _override_with_env(self):
        """Sobreescribe las configuraciones con variables de entorno del S.O. si existen."""
        for key in self.settings.keys():
            env_val = os.getenv(key)
            if env_val is not None:
                # Convertir tipos según el valor original definido por defecto
                orig_type = type(self.settings[key])
                try:
                    if orig_type == bool:
                        self.settings[key] = env_val.lower() in ("true", "1", "yes")
                    elif orig_type == int:
                        self.settings[key] = int(env_val)
                    elif orig_type == float:
                        self.settings[key] = float(env_val)
                    else:
                        self.settings[key] = env_val
                except ValueError:
                    print(f"[ALERTA CONFIG] No se pudo convertir la variable de entorno {key} al tipo {orig_type}.")

    def get(self, key: str, default=None):
        """Retorna un parámetro específico de configuración."""
        return self.settings.get(key, default)
```

3. Crea el módulo de logging unificado en `/workspace/python-automation/src/utils/custom_logger.py`. Este módulo establece dos manejadores (handlers) de registros: uno para salida estándar de consola con colores lógicos e información de depuración, y un manejador rotativo de archivos de texto en disco para auditorías posteriores:

```python
## File: /workspace/python-automation/src/utils/custom_logger.py
import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path

def setup_logger(name: str = "automation_pipeline", log_file: str = "/workspace/python-automation/logs/pipeline.log") -> logging.Logger:
    """Configura un sistema estructurado de logs de alta disponibilidad."""
    logger = logging.getLogger(name)
    logger.setLevel(logging.DEBUG)
    
    # Evitar duplicidad de handlers si se llama varias veces
    if logger.handlers:
        return logger

    # Crear directorios de logs si no existen
    Path(log_file).parent.mkdir(parents=True, exist_ok=True)

    # Formato homogéneo para auditoría técnica
    formatter = logging.Formatter(
        fmt="%(asctime)s | %(levelname)-8s | [%(filename)s:%(lineno)d] | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )

    # Handler para Consola (Salida estándar)
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)

    # Handler para Archivos en disco con Rotación Automática (Max 5MB por archivo, guarda 3 respaldos)
    file_handler = RotatingFileHandler(
        filename=log_file,
        maxBytes=5 * 1024 * 1024,
        backupCount=3,
        encoding="utf-8"
    )
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)

    logger.addHandler(console_handler)
    logger.addHandler(file_handler)

    return logger
```

**Verificación de este paso**: Puedes verificar la carga dinámica de variables ejecutando en la consola:
```bash
python -c "from src.utils.config import ConfigManager; print(ConfigManager().get('MAX_RETRIES'))"
## Debería retornar 3
```

---

### Paso 3: Implementar la Arquitectura Orientada a Objetos para el ETL

**Objetivo**: Crear clases independientes y desacopladas para cada etapa del flujo (Extracción, Transformación, y Carga), asegurando un control estricto de excepciones, resiliencia ante cortes de red mediante reintentos, y consistencia transaccional al guardar los datos finales.

**Instrucciones**:

1. Crea el componente de extracción resiliente en `/workspace/python-automation/src/extractor.py`. Este módulo implementa un ciclo robusto de reintentos con retrasos basados en exponencial backoff:

```python
## File: /workspace/python-automation/src/extractor.py
import time
import requests
from src.utils.custom_logger import setup_logger

logger = setup_logger()

class DataExtractor:
    """Clase responsable de la ingesta de datos remotos mediante HTTP."""

    def __init__(self, url: str, max_retries: int = 3, backoff_factor: float = 2.0, timeout: int = 5):
        self.url = url
        self.max_retries = max_retries
        self.backoff_factor = backoff_factor
        self.timeout = timeout
        self.session = requests.Session()

    def download_data(self) -> str:
        """Descarga el contenido CSV aplicando reintentos exponenciales ante errores de red transitorios (5xx, timeouts)."""
        retries = 0
        current_delay = self.backoff_factor

        while retries < self.max_retries:
            try:
                logger.info(f"Intentando descargar datos (Intento {retries + 1}/{self.max_retries})...")
                response = self.session.get(self.url, timeout=self.timeout)
                
                # Manejar códigos HTTP
                if response.status_code == 200:
                    logger.info("Inundación de datos completada con éxito.")
                    return response.text
                
                # Si es un error del servidor (5xx), amerita reintento
                elif 500 <= response.status_code < 600:
                    logger.warning(f"Error del servidor HTTP {response.status_code} recibido.")
                else:
                    logger.error(f"Error HTTP no recuperable {response.status_code} recibido.")
                    response.raise_for_status()

            except requests.RequestException as e:
                logger.warning(f"Excepción de conexión capturada: {str(e)}")

            retries += 1
            if retries < self.max_retries:
                logger.info(f"Esperando {current_delay} segundos antes de reintentar...")
                time.sleep(current_delay)
                current_delay *= 2  # Exponencial Backoff

        logger.critical("Se agotaron todos los intentos de descarga de red.")
        raise ConnectionError("No se pudo obtener datos del servidor de manera estable.")
```

2. Crea el módulo de transformación de datos con manejo tolerante a fallos de tipado en `/workspace/python-automation/src/transformer.py`:

```python
## File: /workspace/python-automation/src/transformer.py
import csv
import io
from typing import List, Dict, Any
from src.utils.custom_logger import setup_logger

logger = setup_logger()

class DataTransformer:
    """Clase encargada de parsear, limpiar, validar tipos y normalizar los datos."""

    def clean_and_validate(self, raw_csv_data: str) -> List[Dict[str, Any]]:
        """Limpia los encabezados y convierte tipos de datos, descartando filas inválidas."""
        cleaned_records = []
        
        # Usamos modulo csv para evitar cortes manuales frágiles por comas
        reader = csv.DictReader(io.StringIO(raw_csv_data.strip()))
        
        # Validar estructura básica del encabezado
        expected_fields = {"id", "fecha", "monto", "cliente"}
        actual_fields = set(reader.fieldnames or [])
        if not expected_fields.issubset(actual_fields):
            logger.critical(f"El esquema recibido no es válido. Falta alguno de: {expected_fields}")
            raise ValueError("Incompatibilidad del esquema de datos entrante.")

        for row_index, row in enumerate(reader, start=2):
            try:
                # Validar y castear tipos rigurosamente
                record_id = int(row["id"].strip())
                fecha = row["fecha"].strip()
                monto = float(row["monto"].strip()) # Aquí fallará localmente el registro corrupto
                cliente = row["cliente"].strip()

                if not fecha or not cliente:
                    raise ValueError("Campos obligatorios vacíos encontrados.")

                cleaned_records.append({
                    "id": record_id,
                    "fecha": fecha,
                    "monto": monto,
                    "cliente": cliente
                })
            except (ValueError, TypeError, KeyError) as e:
                # Tolerancia a fallas: se descarta el registro corrupto pero se continúa el procesamiento
                logger.error(f"Fila {row_index} corrupta omitida. Error: {str(e)} | Datos de fila: {row}")
                continue

        logger.info(f"Transformación finalizada. Filas procesadas exitosamente: {len(cleaned_records)}")
        return cleaned_records
```

3. Crea el cargador de archivos consistente en `/workspace/python-automation/src/loader.py`. Este módulo emplea una técnica transaccional básica de escritura de sistemas de archivos: escribe primero un archivo temporal y luego lo renombra al destino final, evitando la pérdida de información en caso de interrupción a medio proceso de escritura:

```python
## File: /workspace/python-automation/src/loader.py
import csv
import os
from pathlib import Path
from typing import List, Dict, Any
from src.utils.custom_logger import setup_logger

logger = setup_logger()

class DataLoader:
    """Clase responsable de la persistencia atómica de la información procesada."""

    def __init__(self, output_path: str):
        self.output_path = Path(output_path)

    def save_to_csv(self, data: List[Dict[str, Any]]) -> bool:
        """Guarda la lista de diccionarios en un archivo CSV de forma atómica."""
        if not data:
            logger.warning("No hay datos válidos disponibles para escribir en disco.")
            return False

        # Asegurar que el directorio de salida exista
        self.output_path.parent.mkdir(parents=True, exist_ok=True)
        
        # Escritura atómica mediante archivo temporal
        temp_file = self.output_path.with_suffix(".tmp")
        headers = list(data[0].keys())

        try:
            logger.info(f"Iniciando escritura física temporal en {temp_file}...")
            with open(temp_file, "w", newline="", encoding="utf-8") as f:
                writer = csv.DictWriter(f, fieldnames=headers)
                writer.writeheader()
                writer.writerows(data)
            
            # Renombrado atómico (mueve del temporal al definitivo en un único paso de S.O.)
            if os.path.exists(self.output_path):
                os.remove(self.output_path)
            os.rename(temp_file, self.output_path)
            logger.info(f"Persistencia exitosa. Archivo listo en {self.output_path}")
            return True
            
        except Exception as e:
            logger.critical(f"Error de E/S fatal escribiendo archivo físico de datos: {str(e)}")
            if temp_file.exists():
                os.remove(temp_file)
            raise e
```

---

### Paso 4: Construir el Orquestador Central y Ejecutar la Automatización

**Objetivo**: Integrar las piezas modulares en un único orquestador estructurado e interactivo controlado por la clase `ConfigManager`, y validar su correcta ejecución.

**Instrucciones**:

1. Crea el orquestador principal del pipeline en `/workspace/python-automation/src/pipeline.py`:

```python
## File: /workspace/python-automation/src/pipeline.py
import sys
from src.utils.config import ConfigManager
from src.utils.custom_logger import setup_logger
from src.extractor import DataExtractor
from src.transformer import DataTransformer
from src.loader import DataLoader

logger = setup_logger()

class AutomationPipeline:
    """Orquestador de automatización de datos (ETL)."""

    def __init__(self, config: ConfigManager):
        self.config = config
        # Instanciar submódulos pasando la configuración dinámica
        self.extractor = DataExtractor(
            url=self.config.get("API_URL"),
            max_retries=self.config.get("MAX_RETRIES"),
            backoff_factor=self.config.get("BACKOFF_FACTOR"),
            timeout=self.config.get("TIMEOUT_SECONDS")
        )
        self.transformer = DataTransformer()
        self.loader = DataLoader(
            output_path=self.config.get("OUTPUT_PATH")
        )

    def run(self) -> bool:
        """Ejecuta de manera coordinada el ciclo completo de la automatización."""
        logger.info("==================================================================")
        logger.info(f"Iniciando ciclo de automatización en entorno: {self.config.get('ENVIRONMENT')}")
        logger.info("==================================================================")

        try:
            # 1. Extracción con reintentos
            raw_data = self.extractor.download_data()

            # 2. Transformación tolerante a errores de fila
            cleaned_data = self.transformer.clean_and_validate(raw_data)

            # 3. Carga atómica de archivos
            success = self.loader.save_to_csv(cleaned_data)

            if success:
                logger.info("Pipeline de automatización completado con éxito absoluto.")
                return True
            else:
                logger.error("La automatización culminó pero no se escribieron datos.")
                return False

        except Exception as e:
            logger.critical(f"Falla crítica irrecuperable en el pipeline: {str(e)}", exc_info=True)
            return False

if __name__ == "__main__":
    # Inicializar manager de configuración
    config_manager = ConfigManager()
    
    # Instanciar y arrancar el pipeline centralizado
    pipeline = AutomationPipeline(config_manager)
    exito = pipeline.run()
    
    # Asegurar códigos de retorno correctos al sistema operativo
    if not exito:
        sys.exit(1)
    sys.exit(0)
```

2. Ejecuta el pipeline final:
```bash
python /workspace/python-automation/src/pipeline.py
```

**Resultado esperado de la ejecución**:
Verás registros estructurados en la consola que indican el flujo paso a paso:
- Un reintento inmediato si el servidor inestable simula una caída 500 (gracias al Exponential Backoff implementado).
- Un aviso de error al encontrarse con el registro `INVALID_AMOUNT`, omitiéndolo y continuando el procesamiento del resto de las filas en lugar de bloquearse catastróficamente.
- Un aviso de persistencia atómica exitosa del archivo final de salida.

---

## Validación y Pruebas

Para garantizar que el código modificado cumple con los requisitos de robustez requeridos de una automatización empresarial, llevaremos a cabo una fase de pruebas que incluye una validación medible y pruebas frente a casos adversos (adversarial/edge cases).

### Validación Medible de Resultados

Ejecuta el siguiente script de validación física en tu terminal para confirmar que la infraestructura ha funcionado de manera correcta:

```bash
## 1. Verificar existencia del archivo físico procesado
test -f /workspace/python-automation/data/validated/ventas_procesadas.csv && echo "[OK] El archivo de salida existe." || echo "[ERROR] Archivo no encontrado."

## 2. Contar filas del CSV generado (Debe tener los encabezados + 3 filas de datos procesadas, habiendo ignorado la corrupta)
## Total de líneas esperadas: 4 (encabezados, registro 1, registro 2, registro 4)
total_lineas=$(wc -l < /workspace/python-automation/data/validated/ventas_procesadas.csv)
if [ "$total_lineas" -eq 4 ]; then
    echo "[OK] Cantidad de líneas válidas esperadas: 4."
else
    echo "[ERROR] Cantidad incorrecta de líneas generadas: $total_lineas."
fi

## 3. Comprobar registros en el archivo físico de logs
grep -i "omitida" /workspace/python-automation/logs/pipeline.log && echo "[OK] Log de omisión de fila registrado." || echo "[ERROR] El log de alerta no existe."
```

### Pruebas frente a Casos Adversos (Inyección de Datos y Formato Incompatible)

Como parte de los principios de diseño seguro y robusto en automatización, probaremos la resiliencia del sistema ante un caso adverso: una inyección de encabezado corrupto y datos nulos manipulados desde un archivo externo.

1. Detén el servidor HTTP de simulación (`unstable_server.py`) presionando `Ctrl + C` en su terminal.
2. Vamos a engañar al pipeline ejecutando una simulación de variables de entorno donde forzamos la lectura de un endpoint inexistente y simulamos una alteración de configuración en vivo usando variables de entorno de Linux:

```bash
## Definimos valores de configuración mediante variables de entorno para anular el settings.json
export MAX_RETRIES=2
export API_URL="http://localhost:8080/no_existe.csv"

## Ejecutamos el pipeline
python /workspace/python-automation/src/pipeline.py
```

**Resultado esperado**: El pipeline de automatización abortará de forma segura y controlada arrojando un código de error de salida `1` al sistema operativo. En el log `/workspace/python-automation/logs/pipeline.log` verás documentado el historial de reintentos rápidos, la advertencia de conexión y la traza de excepción crítica limpia, evitando de esta forma un bucle infinito o un comportamiento silencioso.

Asegúrate de limpiar las variables de entorno de prueba antes de continuar:
```bash
unset MAX_RETRIES
unset API_URL
```

---

## Solución de Problemas

A continuación se exponen las dos fallas más frecuentes identificadas al ejecutar este laboratorio con sus respectivas resoluciones paso a paso.

### Caso 1: Error `ModuleNotFoundError: No module named 'requests'` al inicializar el pipeline

- **Síntoma**: Al ejecutar `python /workspace/python-automation/src/pipeline.py`, la terminal arroja una traza de error que finaliza con: `ModuleNotFoundError: No module named 'requests'`.
- **Causa**: Estás ejecutando el pipeline fuera del entorno virtual de Python (`venv`) en donde instalaste los paquetes, o el entorno virtual no fue inicializado correctamente.
- **Solución**: Asegúrate de estar situado en la raíz del espacio de trabajo y activa tu entorno virtual antes de proceder con el script:
  ```bash
  cd /workspace/python-automation
  source venv/bin/activate
  # Vuelve a verificar las dependencias instaladas en el entorno actual
  pip show requests
  ```

### Caso 2: El archivo `pipeline.log` no registra eventos en tiempo de ejecución o no se crea

- **Síntoma**: La carpeta `/workspace/python-automation/logs` está vacía o el archivo `pipeline.log` no incrementa su peso de bytes durante ejecuciones repetidas.
- **Causa**: Fallo en los permisos de escritura del sistema operativo sobre la ruta absoluta asignada en `/workspace/python-automation/logs/` o colisión de bloqueos de archivo por parte de procesos zombies de Python que no finalizaron correctamente en segundo plano.
- **Solución**: 
  1. Brinda permisos completos de escritura al espacio de trabajo (en entornos Linux/macOS):
     ```bash
     chmod -R 755 /workspace/python-automation
     ```
  2. Verifica que no existan procesos Python persistentes en segundo plano que tengan abierto el descriptor del archivo de registros:
     ```bash
     ps aux | grep python
     # O finaliza los hilos colgados de forma selectiva
     pkill -f unstable_server.py
     ```

---

## Limpieza

Para finalizar, limpia de forma segura el entorno del laboratorio eliminando las variables temporales del sistema y apagando los hilos de simulación activa:

```bash
## 1. Terminar todos los procesos de simulación de servidor local en segundo plano
pkill -f unstable_server.py || echo "Servidor inestable ya se encontraba detenido."

## 2. Remover de forma segura los archivos temporales y de persistencia de salida generados
rm -f /workspace/python-automation/data/validated/datos_manuales.csv
rm -f /workspace/python-automation/data/validated/ventas_procesadas.csv.tmp

## 3. Limpiar los caches generados de python
find /workspace/python-automation -type d -name "__pycache__" -exec rm -r {} + 2>/dev/null || true

## 4. Desactivar el entorno de desarrollo virtual
deactivate 2>/dev/null || true

echo "[+] Limpieza del entorno de automatización finalizada."
```

---

## Resumen

En este laboratorio, has completado de manera exitosa la transición de un script monolítico ("espagueti") inestable a un framework de automatización modular de nivel empresarial:

1. **Diseño modular desacoplado**: Separaste los componentes en clases dedicadas (`DataExtractor`, `DataTransformer`, `DataLoader`), permitiendo que cada etapa sea mantenida de forma independiente bajo principios SOLID.
2. **Resiliencia robusta**: Implementaste técnicas de tolerancia de red mediante algoritmos de **Retraso Exponencial (Exponential Backoff)** y persistencia transaccional mediante buffers de archivos temporales.
3. **Manejo controlado de excepciones**: El sistema aprendió a tolerar errores de campos específicos (filas mal formadas) sin la necesidad de interrumpir la descarga completa de registros saludables de transacciones.
4. **Configuración dinámica de variables**: Diseñaste un administrador dinámico capaz de adaptarse a entornos variables (Desarrollo, Pruebas o Producción) a través del motor de herencia dinámico entre variables de entorno y archivos de configuración estructurados en formato JSON.

Este framework de automatización modular servirá como la base sólida para los próximos laboratorios, donde agregaremos validación de contratos de datos rígidos, contenedores Docker y orquestadores programados de procesos automatizados.
