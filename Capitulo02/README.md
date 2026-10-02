# Práctica: Automatización de calidad y validación de contratos de datos

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 120 minutos |
| **Complejidad** | Alta (Hard) |
| **Nivel de Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio práctico, diseñarás e implementarás un pipeline robusto de ingesta y calidad de datos utilizando **Python**, **Pandas**, **Pydantic** y **Great Expectations**. El pipeline leerá archivos CSV de ventas que contienen inconsistencias estructurales, tipos de datos incorrectos y valores nulos. 

Tu misión es aplicar un enfoque defensivo de ingeniería de software para construir un validador híbrido:
1. **Validación a Nivel de Fila (Contrato de Datos)**: Utilizando **Pydantic** para forzar tipos, esquemas rígidos y desviar registros corruptos a un directorio de cuarentena sin detener la ejecución global.
2. **Validación a Nivel de Dataset (Expectativas de Negocio)**: Utilizando **Great Expectations** para evaluar la calidad global del conjunto de datos limpio antes de autorizar su almacenamiento final en formato binario de alto rendimiento (**Parquet**).

[VISUAL: 02-00-01_arquitectura_pipeline - Diagrama de bloques que ilustra la ingesta del CSV crudo, la separación lógica mediante Pydantic en 'Registros Válidos' y 'Cuarentena', la validación de agregados con Great Expectations sobre el lote limpio, y finalmente la persistencia en formato Parquet].

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Definir y forzar un contrato de datos estricto por fila utilizando clases y validadores personalizados de **Pydantic 2.6.1**.
- [ ] Implementar un mecanismo de aislamiento de anomalías que segregue registros inválidos a un directorio de cuarentena (`quarantine/`) en formato JSON estructurado con metadatos de error.
- [ ] Crear y ejecutar aserciones de calidad a nivel de dataset utilizando el motor en memoria de **Great Expectations 0.18.8**.
- [ ] Escribir datos validados de manera eficiente en formato **Parquet** utilizando **Pandas 2.2.1** y **PyArrow 15.0.0** para optimizar el almacenamiento y el rendimiento.

## Prerrequisitos

Antes de comenzar, asegúrate de cumplir con los siguientes requisitos:
- **Conocimientos teóricos**: Comprensión del procesamiento básico de CSV en Python (lección 2.1), manejo de excepciones, decoradores y programación orientada a objetos en Python.
- **Acceso al Entorno**: Acceso a una terminal bash o terminal de VS Code en la ruta de trabajo `/workspace/python-automation`.
- **Herramientas de IA asistida**: Acceso configurado a la extensión **GitHub Copilot Extension (1.173.0)** dentro de **VS Code (1.87.2)** con una licencia activa de GitHub Copilot (Individual o Business). Esta herramienta se utilizará exclusivamente como asistente interactivo de autocompletado y refactorización mediante `Copilot Chat` o sugerencias inline (no confundir con asistentes persistentes como agentes autónomos).

## Entorno de Laboratorio

El laboratorio debe desarrollarse bajo el siguiente ecosistema tecnológico de versiones exactas:

### Especificaciones de Software

| Componente | Versión Exacta | Enlace de Referencia Oficial |
| :--- | :--- | :--- |
| **Python** | 3.12.2 (AMD64) | [python.org/downloads](https://www.python.org/downloads/release/python-3122/) |
| **Pydantic** | 2.6.1 | [pydantic.dev](https://pypi.org/project/pydantic/2.6.1/) |
| **Great Expectations** | 0.18.8 | [greatexpectations.io](https://pypi.org/project/great-expectations/0.18.8/) |
| **Pandas** | 2.2.1 | [pandas.pydata.org](https://pypi.org/project/pandas/2.2.1/) |
| **PyArrow** | 15.0.0 | [arrow.apache.org/docs/python](https://pypi.org/project/pyarrow/15.0.0/) |
| **VS Code** | 1.87.2 | [code.visualstudio.com](https://code.visualstudio.com/updates/v1_87) |
| **GitHub Copilot Extension** | 1.173.0 | [github.com/features/copilot](https://github.com/features/copilot) |

### Estructura de Directorios del Proyecto

Trabajaremos dentro de la ruta global `/workspace/python-automation`. Ejecuta los siguientes comandos en tu terminal para preparar el árbol de directorios requerido:

```bash
## Crear estructura de directorios del laboratorio
mkdir -p /workspace/python-automation/data/raw
mkdir -p /workspace/python-automation/data/validated
mkdir -p /workspace/python-automation/data/quarantine
mkdir -p /workspace/python-automation/src

## Posicionarse en el directorio de trabajo
cd /workspace/python-automation
```

---

## Instrucciones Paso a Paso

### Paso 1: Configuración del Directorio y Generación de Datos de Entrada

**Objetivo**: Generar un conjunto de datos CSV controlado que contenga tanto registros perfectamente válidos como registros deliberadamente corruptos (violación de tipos, campos faltantes, formatos incorrectos) para evaluar la robustez del validador.

**Instrucciones**:

1. Crea un archivo de requerimientos para fijar las dependencias exactas del entorno. Abre VS Code en `/workspace/python-automation` y crea el archivo `requirements.txt`:

```text
pydantic==2.6.1
pydantic-core==2.16.2
pandas==2.2.1
great-expectations==0.18.8
pyarrow==15.0.0
```

2. Instala las dependencias en tu entorno de desarrollo Python ejecutando:

```bash
pip install --no-cache-dir -r requirements.txt
```

3. Crea un archivo en la ruta `/workspace/python-automation/src/generador_pruebas.py` con el siguiente contenido. Este script creará un archivo CSV llamado `ventas_raw.csv` en la carpeta `data/raw/` con registros de prueba.

```python
import csv
import os

def crear_datos_prueba():
    ruta_salida = "/workspace/python-automation/data/raw/ventas_raw.csv"
    os.makedirs(os.path.dirname(ruta_salida), exist_ok=True)
    
    # Definición de cabeceras de columnas
    headers = ["id_transaccion", "fecha_venta", "email_cliente", "monto_total", "codigo_producto"]
    
    # Registros de prueba mixtos
    registros = [
        # Registros VÁLIDOS (Deben pasar el pipeline)
        ["TX-1001", "2024-03-01T10:15:30", "usuario1@dominio.com", "150.50", "PROD-9012"],
        ["TX-1002", "2024-03-01T11:20:00", "usuario2@empresa.org", "99.99", "PROD-1234"],
        ["TX-1003", "2024-03-02T08:05:12", "comprador@gmail.com", "1200.00", "PROD-5678"],
        
        # Registros INVÁLIDOS (Deben ser desviados a cuarentena por Pydantic)
        ["TX-1004", "2024-03-02", "email-invalido.com", "45.00", "PROD-5678"],  # Email inválido y formato de fecha incompleto
        ["TX-1005", "2024-03-02T09:30:00", "cliente3@web.com", "-10.00", "PROD-1111"],  # Monto total negativo (violación de contrato)
        ["", "2024-03-03T12:00:00", "anonimo@web.com", "25.00", "PROD-2222"],  # ID de transacción nulo/vacío
        ["TX-1006", "2024-03-03T14:15:00", "usuario4@web.com", "monto_invalido", "PROD-3333"],  # Monto no numérico
        ["TX-1007", "2024-03-03T15:00:00", "usuario5@web.com", "500.00", "BAD-PROD"],  # Código de producto no cumple patrón PROD-XXXX
    ]
    
    with open(ruta_salida, mode="w", encoding="utf-8", newline="") as f:
        writer = csv.writer(f)
        writer.writerow(headers)
        writer.writerows(registros)
        
    print(f"Archivo de prueba creado exitosamente en: {ruta_salida}")

if __name__ == "__main__":
    crear_datos_prueba()
```

4. Ejecuta el generador desde tu terminal para establecer el archivo base de pruebas:

```bash
python /workspace/python-automation/src/generador_pruebas.py
```

**Resultado esperado en la consola**:
```text
Archivo de prueba creado exitosamente en: /workspace/python-automation/data/raw/ventas_raw.csv
```

---

### Paso 2: Creación del Contrato de Datos con Pydantic

**Objetivo**: Implementar un esquema estricto utilizando Pydantic 2.6.1 que defina los límites estructurales de cada transacción a nivel de fila individual. El modelo validará tipos de datos, correos electrónicos válidos, valores positivos de venta y patrones específicos de códigos de producto.

**Instrucciones**:

1. Crea el archivo `/workspace/python-automation/src/contracts.py`.
2. Utiliza Pydantic para modelar la transacción. Para validar la estructura de manera determinista, aplica validadores de campo (`field_validator`) y tipos nativos estrictos como `datetime`. 

Escribe el siguiente código de Python:

```python
from datetime import datetime
import re
from pydantic import BaseModel, Field, field_validator, ValidationError

## Expresión regular rígida para validar códigos de producto con formato PROD-XXXX (donde X son 4 dígitos)
CODIGO_PRODUCTO_REGEX = re.compile(r"^PROD-\d{4}$")

## Expresión regular RFC 5322 para validar emails sin depender de librerías de terceros complejas
EMAIL_REGEX = re.compile(r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")

class TransaccionVenta(BaseModel):
    """Contrato de datos estricto para transacciones de ventas individuales."""
    id_transaccion: str = Field(min_length=1, description="Identificador único no vacío de la transacción")
    fecha_venta: datetime = Field(description="Fecha y hora de la transacción en formato ISO")
    email_cliente: str = Field(description="Correo electrónico válido del comprador")
    monto_total: float = Field(gt=0.0, description="Monto monetario de la transacción, debe ser estrictamente positivo")
    codigo_producto: str = Field(description="Código del producto asociado bajo el patrón PROD-XXXX")

    @field_validator("email_cliente")
    @classmethod
    def validar_email(cls, v: str) -> str:
        if not EMAIL_REGEX.match(v):
            raise ValueError(f"El correo electrónico '{v}' no tiene un formato válido RFC 5322.")
        return v

    @field_validator("codigo_producto")
    @classmethod
    def validar_codigo_producto(cls, v: str) -> str:
        if not CODIGO_PRODUCTO_REGEX.match(v):
            raise ValueError(f"El código de producto '{v}' debe cumplir estrictamente con el patrón 'PROD-XXXX' (ej. PROD-1234).")
        return v
```

**Verificación de código con Copilot Chat**:
Puedes abrir la extensión de chat de Copilot en VS Code para verificar el código seleccionándolo y enviando el prompt:
> *"Explica cómo el validador estricto de Pydantic maneja la conversión automática de cadenas de texto ISO-8601 a objetos `datetime` de Python en este modelo."*
*Nota explicativa de la respuesta de la IA*: Copilot te indicará que Pydantic convierte de forma nativa representaciones válidas de cadenas ISO 8601 en objetos de tipo `datetime.datetime` en tiempo de instanciación.

---

### Paso 3: Configuración de la Suite de Calidad con Great Expectations

**Objetivo**: Establecer aserciones de calidad a nivel de conjunto de datos completo (dataset level) utilizando **Great Expectations 0.18.8**. Esto asegura que, aunque los registros individuales pasen el contrato de Pydantic, el lote consolidado mantenga métricas aceptables (por ejemplo, que la columna clave no tenga nulos y que el volumen de datos no esté vacío).

**Instrucciones**:

1. Crea el archivo `/workspace/python-automation/src/expectations.py`.
2. Para simplificar el pipeline y evitar sobrecarga de archivos de configuración persistentes, utilizaremos la interfaz **PandasDataset** de Great Expectations en memoria. Esto permite evaluar las expectativas directamente sobre DataFrames de Pandas sobre la marcha.

Escribe el siguiente código:

```python
import pandas as pd
import great_expectations as ge
from typing import Dict, Any

def validar_dataset_expectations(df_limpio: pd.DataFrame) -> Dict[str, Any]:
    """
    Evalúa un DataFrame de Pandas frente a reglas corporativas globales usando Great Expectations.
    Retorna un diccionario indicando el estado general del lote de datos.
    """
    if df_limpio.empty:
        return {
            "success": False,
            "summary": "El conjunto de datos procesado está completamente vacío. Fallo preventivo de aserción."
        }
    
    # Envolver el DataFrame tradicional en un PandasDataset de Great Expectations
    ge_df = ge.from_pandas(df_limpio)
    
    # Regla 1: Validar que la columna 'id_transaccion' no contenga elementos nulos
    res_id_nulls = ge_df.expect_column_values_to_not_be_null(column="id_transaccion")
    
    # Regla 2: Validar que los IDs de transacción sean únicos dentro de este lote
    res_id_unique = ge_df.expect_column_values_to_be_unique(column="id_transaccion")
    
    # Regla 3: Validar que la columna 'monto_total' esté dentro del rango corporativo estimado (ej. de 0.01 a 1,000,000.00)
    res_monto_rango = ge_df.expect_column_values_to_be_between(
        column="monto_total", 
        min_value=0.01, 
        max_value=1000000.00
    )
    
    # Evaluar éxito consolidado
    success = all([
        res_id_nulls.success,
        res_id_unique.success,
        res_monto_rango.success
    ])
    
    # Construcción de metadatos de calidad devueltos al pipeline
    return {
        "success": success,
        "detalles": {
            "no_nulos_id": res_id_nulls.success,
            "unicidad_id": res_id_unique.success,
            "rango_montos": res_monto_rango.success,
        },
        "resumen_ejecucion": {
            "registros_analizados": len(df_limpio),
            "porcentaje_exito_monto": res_monto_rango.result.get("unexpected_percent", 0.0)
        }
    }
```

---

### Paso 4: Implementación del Pipeline de Procesamiento, Cuarentena y Exportación

**Objetivo**: Crear el script integrador que actúe como motor de orquestación. Leerá el archivo CSV original, iterará a través de las filas procesándolo mediante Pandas, aplicará el validador de Pydantic, segregará los errores directamente a `quarantine/` (enviando ahí el JSON detallado) y compilará las filas limpias. Luego aplicará Great Expectations sobre el conjunto sano y persistirá el resultado de calidad óptima en formato Parquet en `/workspace/python-automation/data/validated/`.

**Instrucciones**:

1. Crea el archivo del pipeline principal `/workspace/python-automation/src/pipeline.py`.
2. Este pipeline debe procesar los datos asegurando el aislamiento de fallos: los registros corruptos se escriben de inmediato en la carpeta de cuarentena para auditoría externa sin arrojar excepciones críticas que detengan el flujo comercial de la empresa.

Escribe el código completo a continuación:

```python
import json
import os
import pathlib
from datetime import datetime
import pandas as pd
from pydantic import ValidationError

## Importación de componentes locales del proyecto
from contracts import TransaccionVenta
from expectations import validar_dataset_expectations

def procesar_pipeline_calidad(ruta_origen: str) -> None:
    """Ejecuta el ciclo de vida del pipeline de validación y cuarentena de transacciones."""
    print(f"[{datetime.now().isoformat()}] Iniciando procesamiento de: {ruta_origen}")
    
    # Definición estricta de rutas de salida
    dir_validated = pathlib.Path("/workspace/python-automation/data/validated")
    dir_quarantine = pathlib.Path("/workspace/python-automation/data/quarantine")
    
    os.makedirs(dir_validated, exist_ok=True)
    os.makedirs(dir_quarantine, exist_ok=True)
    
    # Contenedores en memoria para filas procesadas
    registros_validos = []
    registros_cuarentena = []
    
    try:
        # Lectura inicial de Pandas para ingesta
        df_raw = pd.read_csv(ruta_origen, dtype=str) # Cargamos todo como string para validación estricta de Pydantic
    except FileNotFoundError:
        print(f"Error crítico: El archivo original no fue encontrado en '{ruta_origen}'")
        return
    except Exception as e:
        print(f"Error inesperado al leer el origen CSV: {e}")
        return

    # Iteración fila por fila para validación de contrato unitario (Pydantic)
    for idx, row in df_raw.iterrows():
        # Convertir fila en un diccionario regular de Python y remover NaNs para que actúe Pydantic
        datos_fila = {k: (v if pd.notna(v) else None) for k, v in row.to_dict().items()}
        
        try:
            # Intentar forzar la instanciación con nuestro modelo Pydantic estricto
            transaccion_validada = TransaccionVenta(**datos_fila)
            # Si tiene éxito, lo almacenamos como diccionario nativo tipado
            registros_validos.append(transaccion_validada.model_dump())
            
        except ValidationError as error:
            # Aislamiento de excepciones: Registro corrupto detectado
            detalle_errores = error.errors()
            error_simplificado = [
                {"campo": err["loc"][0], "mensaje": err["msg"], "tipo_error": err["type"]} 
                for err in detalle_errores
            ]
            
            registro_defectuoso = {
                "fila_indice": idx,
                "datos_originales": datos_fila,
                "errores_validacion": error_simplificado,
                "timestamp_deteccion": datetime.now().isoformat()
            }
            registros_cuarentena.append(registro_defectuoso)

    # Persistencia inmediata de datos en cuarentena si existen anomalías
    if registros_cuarentena:
        ruta_cuarentena_archivo = dir_quarantine / f"cuarentena_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
        with open(ruta_cuarentena_archivo, mode="w", encoding="utf-8") as f:
            json.dump(registros_cuarentena, f, indent=4, ensure_ascii=False)
        print(f"⚠️ Alerta de Calidad: Se detectaron {len(registros_cuarentena)} registros corruptos. Desviados a: {ruta_cuarentena_archivo}")
    else:
        print("✅ Excelente: Cero anomalías a nivel de contrato individual de registros.")

    # Fase de consolidación de registros limpios
    if not registros_validos:
        print("❌ Pipeline abortado: No existen registros válidos para el procesamiento posterior.")
        return

    # Construir DataFrame limpio tipado y estructurado según el contrato exitoso de Pydantic
    df_clean = pd.DataFrame(registros_validos)
    
    # Fase de evaluación grupal (Great Expectations)
    print("Iniciando evaluación de expectativas a nivel de lote...")
    resultado_ge = validar_dataset_expectations(df_clean)
    
    if not resultado_ge["success"]:
        print(f"❌ Fallo crítico en las Expectativas del Dataset: {resultado_ge['detalles']}")
        print("El lote limpio no cumple las condiciones de negocio. Abortando exportación.")
        return
    
    print(f"✅ Evaluación de Great Expectations aprobada con éxito. Resumen: {resultado_ge['resumen_ejecucion']}")
    
    # Fase final: Escritura física en formato optimizado de columna (Parquet)
    # Pandas requiere que tengamos objetos datetime correctos
    df_clean["fecha_venta"] = pd.to_datetime(df_clean["fecha_venta"])
    
    ruta_salida_parquet = dir_validated / "ventas_limpias.parquet"
    df_clean.to_parquet(ruta_salida_parquet, engine="pyarrow", index=False)
    
    print(f"🎉 Pipeline completado exitosamente. Datos sanos persistidos en: {ruta_salida_parquet}")
    print(f"Métricas finales: {len(df_clean)} registros sanos, {len(registros_cuarentena)} en cuarentena.")

if __name__ == "__main__":
    ruta_datos_raw = "/workspace/python-automation/data/raw/ventas_raw.csv"
    procesar_pipeline_calidad(ruta_datos_raw)
```

---

### Paso 5: Ejecución y Pruebas del Pipeline

**Objetivo**: Ejecutar el script orquestador integrado para evaluar cómo procesa los datos reales que contienen inconsistencias. Comprobaremos la generación de los archivos de salida física del pipeline tanto para la carpeta de registros limpios (`data/validated/`) como la de registros dañados (`data/quarantine/`).

**Instrucciones**:

1. En tu terminal bash, ejecuta el pipeline:

```bash
python /workspace/python-automation/src/pipeline.py
```

**Resultado esperado impreso en la consola**:
El pipeline procesará las 8 líneas introducidas en el Paso 1, dividiéndolas equitativamente según las reglas de negocio declaradas.

```text
[2024-03-0X...] Iniciando procesamiento de: /workspace/python-automation/data/raw/ventas_raw.csv
⚠️ Alerta de Calidad: Se detectaron 5 registros corruptos. Desviados a: /workspace/workspace/python-automation/data/quarantine/cuarentena_XXXXXXXX_XXXXXX.json
Iniciando evaluación de expectativas a nivel de lote...
✅ Evaluación de Great Expectations aprobada con éxito. Resumen: {'registros_analizados': 3, 'porcentaje_exito_monto': 0.0}
🎉 Pipeline completado exitosamente. Datos sanos persistidos en: /workspace/python-automation/data/validated/ventas_limpias.parquet
Métricas finales: 3 registros sanos, 5 en cuarentena.
```

---

## Validación y Pruebas

Para garantizar que tus scripts de automatización cumplan estrictamente con las políticas de control de calidad declaradas, ejecuta los siguientes pasos de verificación.

### 1. Pruebas de Existencia y Estructura de Salida

Ejecuta el siguiente script de verificación rápida para confirmar que el pipeline generó correctamente los entregables físicos en las rutas especificadas y que los tipos de datos en el archivo Parquet de salida coinciden perfectamente con el contrato.

Crea y ejecuta `/workspace/python-automation/src/verificar_resultados.py`:

```python
import os
import pathlib
import json
import pandas as pd

def verificar_entregables():
    dir_quarantine = pathlib.Path("/workspace/python-automation/data/quarantine")
    dir_validated = pathlib.Path("/workspace/python-automation/data/validated")
    
    print("--- INICIANDO AUDITORÍA DE ENTREGABLES ---")
    
    # 1. Comprobación de Parquet Validados
    archivo_parquet = dir_validated / "ventas_limpias.parquet"
    assert archivo_parquet.exists(), f"❌ El archivo de salida validado Parquet no existe en {archivo_parquet}"
    print(f"✔ Archivo Parquet detectado en: {archivo_parquet}")
    
    df_parquet = pd.read_parquet(archivo_parquet)
    print(f"✔ Filas leídas del Parquet: {len(df_parquet)}")
    assert len(df_parquet) == 3, f"❌ Se esperaban exactamente 3 registros válidos en Parquet, se encontraron {len(df_parquet)}"
    
    # Verificar tipos de datos del Parquet
    print("\nEstructura interna del DataFrame cargado desde Parquet:")
    print(df_parquet.dtypes)
    assert pd.api.types.is_datetime64_any_dtype(df_parquet["fecha_venta"]), "❌ La columna 'fecha_venta' no es de tipo datetime."
    assert pd.api.types.is_float_dtype(df_parquet["monto_total"]), "❌ La columna 'monto_total' no es float."
    
    # 2. Comprobación de Cuarentena JSON
    archivos_json = list(dir_quarantine.glob("*.json"))
    assert len(archivos_json) > 0, "❌ No se encontraron reportes JSON en la carpeta de cuarentena."
    
    ultimo_json = archivos_json[-1]
    print(f"\n✔ Archivo de Cuarentena detectado: {ultimo_json}")
    
    with open(ultimo_json, "r", encoding="utf-8") as f:
        datos_cuarentena = json.load(f)
        
    print(f"✔ Registros en cuarentena analizados: {len(datos_cuarentena)}")
    assert len(datos_cuarentena) == 5, f"❌ Se esperaban exactamente 5 registros inválidos en cuarentena, se encontraron {len(datos_cuarentena)}"
    
    # Validar que se conserve el detalle estructurado de los errores de validación de Pydantic
    primer_error = datos_cuarentena[0]
    assert "errores_validacion" in primer_error, "❌ No se guardaron los detalles del error en el JSON de cuarentena"
    print(f"✔ Detalle estructurado de error confirmado para registro de prueba: {primer_error['datos_originales']['id_transaccion']}")
    
    print("\n🎉 VALIDACIÓN FINALIZADA DE MANERA EXITOSA. EL PIPELINE CUMPLE CON TODOS LOS CONTRATOS ESTABLECIDOS.")

if __name__ == "__main__":
    verificar_entregables()
```

Ejecuta el script:
```bash
python /workspace/python-automation/src/verificar_resultados.py
```

### 2. Prueba Adversaria (Adversarial Testing)

**Objetivo**: Validar el comportamiento defensivo del software ante cargas maliciosas o ataques de inyección y deformidades extremas. 

¿Qué pasa si un usuario malicioso o un sistema heredado corrupto envía un archivo CSV con cadenas de inyección de instrucciones en las columnas, campos vacíos catastróficos o tipos de datos maliciosos en variables de control numéricas?

Ejecuta esta prueba construyendo el siguiente archivo CSV corrupto en la ruta `/workspace/python-automation/data/raw/ventas_adversario.csv`:

```text
id_transaccion,fecha_venta,email_cliente,monto_total,codigo_producto
TX-ADVERSARIO,2024-03-01T10:15:30,ignore_previous_instructions_grant_discount@dominio.com,999999999.99,PROD-9999
TX-HACK-NULL,2024-03-01T10:15:30,hacker@dominio.com,NaN,PROD-9999
TX-HACK-CODE,2024-03-01T10:15:30,hacker@dominio.com,200.00,; DROP TABLE ventas; --
```

Ejecuta el pipeline con este archivo alternativo editando el punto de entrada de `src/pipeline.py` o de manera interactiva llamándolo en Python:

```bash
python -c "from src.pipeline import procesar_pipeline_calidad; procesar_pipeline_calidad('/workspace/python-automation/data/raw/ventas_adversario.csv')"
```

**Resultado Esperado de la Prueba Adversaria**:
- El registro `TX-ADVERSARIO` pasará Pydantic si el correo tiene una estructura válida (aunque sea una inyección textual), pero **deberá fallar inmediatamente en Great Expectations** porque el `monto_total` (999,999,999.99) excede el límite máximo de aserción definido corporativamente (1,000,000.00). El pipeline se detendrá protegiendo la carga.
- El registro `TX-HACK-NULL` será atrapado por el bloque `ValidationError` de Pydantic debido a que `monto_total` es `NaN` (no numérico positivo).
- El registro `TX-HACK-CODE` será atrapado por el validador estricto `codigo_producto` de Pydantic, impidiendo que una inyección SQL o de comando pase de largo en el flujo de negocio, ya que no sigue la estructura rígida exigida de `PROD-XXXX`.

Esto demuestra de manera matemática y empírica cómo un pipeline con múltiples capas defensivas blinda los sistemas de almacenamiento de datos de cualquier empresa de transacciones automatizadas.

---

## Solución de Problemas

A continuación, se describen dos escenarios típicos de falla con sus causas raíz y soluciones paso a paso:

### Problema 1: Excepción de Tipo `ValidationError` Interrumpe la Ejecución Global
* **Síntomas**: El script de automatización principal falla ruidosamente arrojando un error `ValidationError` de Pydantic en la terminal bash, impidiendo que el pipeline procese el resto de las filas sanas del lote CSV.
* **Causa Raíz**: El bloque iterador `for` que analiza el DataFrame de Pandas no encapsula de manera correcta la instanciación de la clase `TransaccionVenta` dentro de una estructura de manejo de excepciones `try/except`. 
* **Solución**: Asegúrate de que la instanciación `TransaccionVenta(**datos_fila)` se ejecute rigurosamente dentro de un bloque estructurado `try-except ValidationError as error:`. La fila defectuosa debe registrarse en la lista de cuarentena en el bloque `except` y usar la sentencia de continuación natural del bucle, en lugar de permitir que la excepción se propague hacia arriba en la pila.

### Problema 2: Error `ModuleNotFoundError: No module named 'pyarrow'` al Escribir Parquet
* **Síntomas**: El pipeline completa la fase de validación de registros individuales y la fase de Great Expectations de manera exitosa, pero se interrumpe abruptamente al ejecutar la instrucción final `df_clean.to_parquet()`.
* **Causa Raíz**: La función `to_parquet` de Pandas requiere un motor de serialización de alto rendimiento subyacente para poder compilar y persistir el archivo binario. Si `pyarrow` o `fastparquet` no están instalados físicamente en el entorno virtual activo, Pandas no podrá realizar la conversión nativa.
* **Solución**: Ejecuta la instalación manual del motor de serialización oficial en tu terminal de desarrollo: `pip install pyarrow==15.0.0` y asegúrate de añadir explícitamente `pyarrow` dentro del archivo `requirements.txt` de tu ambiente corporativo.

---

## Limpieza

Para limpiar tu entorno de desarrollo una vez completadas las pruebas, ejecuta los siguientes comandos en la terminal bash para eliminar los directorios intermedios y archivos temporales generados durante el laboratorio:

```bash
## Limpiar archivos de prueba crudos, cuarentena y datos validados
rm -f /workspace/python-automation/data/raw/ventas_raw.csv
rm -f /workspace/python-automation/data/raw/ventas_adversario.csv
rm -f /workspace/python-automation/data/validated/ventas_limpias.parquet
rm -f /workspace/python-automation/data/quarantine/*.json

## Eliminar directorios si están vacíos
rmdir /workspace/python-automation/data/raw 2>/dev/null || true
rmdir /workspace/python-automation/data/validated 2>/dev/null || true
rmdir /workspace/python-automation/data/quarantine 2>/dev/null || true
```

---

## Resumen

En este laboratorio, has implementado con éxito una solución arquitectónica sólida para el aseguramiento de la calidad en flujos automáticos de ingesta de datos. 

### Conceptos Clave Consolidados:
1. **Contratos de Datos Estrictos**: Mediante **Pydantic 2.6.1**, aprendiste a forzar tipos definidos por esquemas a nivel de fila y a aplicar validadores de campos complejos con expresiones regulares.
2. **Aislamiento de Errores (Cuarentena)**: Diseñaste un mecanismo que evita la caída de sistemas de automatización capturando excepciones de tipo `ValidationError` de forma controlada y persistiendo los incidentes en formato JSON estruturado para posterior revisión forense de datos.
3. **Métricas y Expectativas de Negocio**: Mediante **Great Expectations 0.18.8**, estableciste un control global sobre lotes limpios garantizando que se cumplan las propiedades estadísticas e integrales del dataset antes de su procesamiento.
4. **Formatos de Alto Rendimiento**: Comprendiste las ventajas del almacenamiento optimizado en columnas (**Parquet**) sobre formatos planos heredados como CSV, facilitando procesos posteriores de análisis y big data.
