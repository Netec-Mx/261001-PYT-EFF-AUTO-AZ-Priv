<p align="center">
  <img src="https://raw.githubusercontent.com/Netec-Mx/261001-PYT-EFF-AUTO-AZ-Priv/main/assets/LogoNetec.png" alt="NETEC" width="180" />
</p>

# Automatización de procesos con Python

Curso práctico y avanzado en el uso de Python para automatizar, estandarizar y escalar procesos internos relacionados con ingeniería y datos. El programa evoluciona desde scripts aislados hacia componentes de automatización mantenibles, comprobables mediante pruebas, seguros, observables y escalables, capaces de integrarse con pipelines de CI/CD, flujos de datos, herramientas operacionales, APIs y servicios programados. Los contenidos priorizan escenarios de integración de datos, orquestación de pipelines, operaciones sobre almacenes de datos en la nube y monitoreo y remediación de pipelines. Las prácticas se desarrollan con datos y escenarios ficticios en ambientes de laboratorio, utilizando como tecnologías de referencia Snowflake, GitHub, Amazon MWAA y Amazon CloudWatch. El curso incorpora el uso responsable de Claude Code y GitHub Copilot como apoyo al desarrollo con Python.


## Lista de laboratorios

----

### Capítulo 1

- [Práctica: De un proceso manual a una automatización mantenible](Capitulo01/README.md#práctica-de-un-proceso-manual-a-una-automatización-mantenible)
  - Duración estimada: 90 min
  - [Ver capítulo completo](Capitulo01/README.md)

Analizar un proceso repetitivo relacionado con ingeniería o datos y transformarlo en la base de una automatización Python estructurada y reutilizable.
Los participantes identificarán las actividades adecuadas para automatización, definirán el alcance, los posibles modos de fallo y los puntos que requieran aprobación humana. A partir de ese análisis, estructurarán una solución Python modular incorporando anotaciones de tipo, configuración externa, variables de entorno, gestión de dependencias y documentación.

**Criterios de evaluación**:

Corrección: correspondencia entre la automatización y el proceso y alcance definidos.
Mantenibilidad: estructura modular, legible, documentada y configurable.
Seguridad: separación adecuada de credenciales y configuraciones sensibles.
Valor operativo: capacidad de reducir una actividad repetitiva o propensa a errores.

----

### Capítulo 2

- [Práctica: Automatización de calidad y validación de contratos de datos](Capitulo02/README.md#práctica-automatización-de-calidad-y-validación-de-contratos-de-datos)
  - Duración estimada: 120 min
  - [Ver capítulo completo](Capitulo02/README.md)

Automatizar controles repetibles de estructura y calidad sobre conjuntos de datos ficticios destinados a un flujo de procesamiento empresarial.
Los participantes construirán una automatización configurable que ingiera información desde archivos, valide esquemas y contratos de datos, aplique reglas de calidad, detecte problemas de completitud, duplicados y anomalías, realice transformaciones y genere automáticamente resultados y excepciones.
La solución deberá permitir reutilizar las mismas reglas sobre diferentes conjuntos de datos.

**Criterios de evaluación**:

Corrección: identificación correcta de incumplimientos de calidad y esquema.
Resiliencia: tratamiento controlado de datos o estructuras inválidas.
Mantenibilidad: reglas y parámetros modificables sin alterar la lógica principal.
Valor operativo: sustitución de controles manuales por validaciones consistentes y repetibles.

----

### Capítulo 3

- [Práctica: Automatización de integración de datos con Snowflake](Capitulo03/README.md#práctica-automatización-de-integración-de-datos-con-snowflake)
  - Duración estimada: 120 min

Implementar un flujo automatizado y repetible de extracción, validación, transformación y persistencia de información entre diferentes sistemas.
Los participantes desarrollarán una solución Python que extraiga información ficticia mediante una API REST, gestione autenticación, paginación, límites de tasa, tiempos de espera y reintentos, valide las respuestas mediante Pydantic y transforme la información antes de su carga y persistencia en Snowflake.
La automatización incorporará controles de esquema, reconciliación e idempotencia para permitir ejecuciones repetidas de forma segura.

**Criterios de evaluación**

Corrección: preservación de la integridad de la información durante la extracción, validación y persistencia.
Resiliencia: gestión de errores de comunicación, límites de tasa, tiempos de espera y reintentos.
Mantenibilidad: separación entre integración, validación y persistencia.
Seguridad: administración segura de autenticación y credenciales.
Valor operativo: automatización de un flujo de integración ejecutable repetidamente.

----

### Capítulo 4

- [Práctica: Monitoreo de pipelines y remediación controlada de pipelines](Capitulo04/README.md#práctica-monitoreo-de-pipelines-y-remediación-controlada-de-pipelines)
  - Duración estimada: 90 min
  - [Ver capítulo completo](Capitulo04/README.md)

Implementar mecanismos que permitan detectar y diagnosticar fallos en un pipeline de datos y ejecutar acciones de recuperación de manera segura y auditable.
Los participantes trabajarán con un pipeline de laboratorio ejecutado mediante Amazon MWAA, identificando fallos y recopilando registros, métricas e información diagnóstica mediante Amazon CloudWatch. A partir de esta información determinarán si la operación puede recuperarse de forma segura y aplicarán políticas de reintento, alertamiento y remediación controlada, manteniendo un registro de las decisiones y acciones realizadas.

**Criterios de evaluación**:

Corrección: identificación adecuada de estados de ejecución y condiciones de fallo.
Resiliencia: aplicación apropiada de mecanismos de reintento, recuperación y remediación.
Mantenibilidad: separación y organización de los componentes de registro, monitoreo y remediación.
Seguridad: protección de información sensible en registros y alertas.
Valor operativo: reducción del esfuerzo necesario para diagnosticar y responder ante fallos.

----

### Capítulo 5

- [Práctica: Pruebas y escalamiento de una automatización](Capitulo05/README.md#práctica-pruebas-y-escalamiento-de-una-automatización)
  - Duración estimada: 120 min
  - [Ver capítulo completo](Capitulo05/README.md)

Comprobar funcionalmente una automatización y optimizar su ejecución sin introducir regresiones ni comprometer su mantenibilidad.
Los participantes construirán una suite de pruebas incorporando pruebas unitarias, pruebas de integración, objetos simulados, datos de prueba y pruebas de regresión.
Posteriormente, ejecutarán la automatización sobre volúmenes crecientes de información, aplicarán procesamiento por lotes y técnicas de programación asíncrona, concurrencia o multiprocesamiento según corresponda, realizarán análisis de rendimiento y compararán su comportamiento antes y después de la optimización.

**Criterios de evaluación**:

Corrección: capacidad de las pruebas para validar el comportamiento esperado y detectar regresiones.
Resiliencia: consideración de escenarios exitosos, errores y condiciones límite.
Mantenibilidad: pruebas reutilizables y correctamente separadas de la lógica productiva.
Valor operativo: mejora medible del procesamiento o utilización de recursos sin comprometer la funcionalidad.

---

### Capítulo 6

- [Práctica: De automatización local a solución desplegable y orquestada](Capitulo06/README.md#práctica-de-automatización-local-a-solución-desplegable-y-orquestada)
  - Duración estimada: 180 min
  - [Ver capítulo completo](Capitulo06/README.md)

Los participantes prepararán una automatización Python para su ejecución reproducible, trabajando su empaquetado y contenerización mediante Docker. Posteriormente, configurarán un flujo de CI/CD mediante GitHub Actions, incorporando pruebas y controles de calidad y seguridad.
Finalmente, trabajarán un escenario independiente de orquestación mediante Amazon MWAA, definiendo un DAG que coordine tareas Python, dependencias, estados de ejecución y manejo de fallos.

**Criterios de evaluación**:

Corrección: ejecución correcta del componente empaquetado en el entorno definido.
Resiliencia: incorporación de validaciones y mecanismos de recuperación ante fallos de entrega.
Mantenibilidad: configuración, dependencias, versionado y empaquetado reproducibles.
Seguridad: gestión adecuada de secretos, permisos, dependencias y datos sensibles.
Valor operativo: capacidad de entregar y operar la automatización mediante un proceso repetible y controlado. 

----

### Capítulo 7

- [Práctica: Productividad del desarrollo asistida por IA](Capitulo07/README.md#práctica-productividad-del-desarrollo-asistida-por-ia)
  - Duración estimada: 120 min
  - [Ver capítulo completo](Capitulo07/README.md)

Aplicar IA de manera controlada sobre una actividad concreta del ciclo de desarrollo Python y validar técnicamente el resultado generado.
Los participantes utilizarán Claude Code o GitHub Copilot para apoyar una actividad sobre código existente, como generación de pruebas, revisión de código, documentación, refactorización o depuración. El resultado deberá ser revisado, probado y validado antes de incorporarse a la solución.

**Criterios de evaluación**:

Corrección: el código generado o modificado conserva el comportamiento funcional esperado.
Mantenibilidad: el resultado mantiene o mejora la legibilidad y calidad del código.
Seguridad: el resultado generado es revisado antes de su incorporación.
Valor operativo: la IA reduce esfuerzo sin sustituir los controles de ingeniería.


## 📬 **Contacto y más información**

Si tienes alguna pregunta o necesitas más detalles, no dudes en [contactarnos](mailto:soporte@netec.com). También puedes encontrar más recursos en nuestra [página](https://netec.com).

---

¡Gracias por visitar nuestra plataforma! No olvides revisar todos los laboratorios y comenzar tu viaje de aprendizaje hoy mismo.
