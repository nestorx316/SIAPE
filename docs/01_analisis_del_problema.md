# Entregable 1. Análisis del problema

## Título del proyecto

**SIAPE: Sistema Inteligente de Análisis, Predicción y Trazabilidad Energética.**
Tema asignado: *Sistema inteligente para análisis y optimización del consumo energético*.

## Integrantes

| # | Integrante | Rol principal |
|---|------------|---------------|
| 1 | Nestor Alexander Acosta Andrade | Procesamiento de datos y Machine Learning (generación, preparación, modelos de consumo y clustering) |
| 2 | Aaron Salas Chavez | Procesamiento de Lenguaje Natural (corpus, sentimiento, NER, generación de texto) |
| 3 | Juan Alberto Santos Espinoza | Blockchain, integración y dashboard (cadena de bloques, backend Flask, dashboard, pruebas) |

Aunque cada integrante tiene un rol principal, todos conocen el flujo completo del sistema (requisito de la sección 4 del documento del profesor). La documentación y las pruebas son responsabilidad compartida.

## Planteamiento del problema

Un operador de eficiencia energética del estado de Veracruz administra 10 instalaciones eléctricas de cuatro tipos: 3 residenciales, 3 comerciales, 2 industriales y 2 educativas (definidas en `config.INSTALACIONES`), ubicadas en Veracruz, Boca del Río, Xalapa, Córdoba, Coatzacoalcos y Orizaba. El operador enfrenta cuatro dificultades:

1. **No anticipa su demanda.** Sin un pronóstico del consumo horario (kWh) no puede planear la contratación de capacidad, programar mantenimientos ni detectar desviaciones.
2. **No distingue patrones de uso.** Es difícil identificar, entre miles de curvas horarias, qué días o instalaciones se comportan de forma distinta (cierres, paros, picos inusuales).
3. **Recibe información textual no estructurada.** Quejas, fallas y comentarios de los usuarios (por app, correo, llamada y portal) contienen información valiosa (equipos dañados, lugares, folios, montos), pero leerlos uno por uno no escala.
4. **No puede demostrar que sus resultados no fueron alterados.** Si un reporte o una predicción sustenta una auditoría o un ajuste tarifario, debe poder probarse que no se modificó después de generarse.

**Pregunta guía:** ¿Es posible construir un prototipo que prediga el consumo eléctrico horario con error bajo, agrupe perfiles de uso, interprete los reportes de los usuarios, genere un informe automático y garantice la integridad de ese informe y de las predicciones con una cadena de bloques?

## Antecedentes

> Esta sección es contextual y general. Cualquier dato externo específico se marca como **(verificar fuente)** y no se utiliza en los cálculos del proyecto.

- **Contexto energético en México.** El suministro eléctrico del país está a cargo principalmente de la Comisión Federal de Electricidad (CFE), y la operación del sistema eléctrico nacional es coordinada por el CENACE. Ambas organizaciones aparecen en el corpus de reportes simulados como entidades reconocibles (`config.ORGANIZACIONES`). Los usuarios mexicanos pueden recibir tarifas que dependen de la región y del tipo de servicio (doméstico, comercial, industrial) **(verificar fuente)**.
- **Veracruz.** El estado tiene clima cálido y húmedo en la costa (Veracruz, Boca del Río, Coatzacoalcos) y más fresco en zonas de altura (Xalapa, Orizaba, Córdoba). El uso de aire acondicionado en clima cálido y húmedo es un factor reconocido de demanda eléctrica en los meses de calor **(verificar fuente)**. Esta relación temperatura-consumo es la que reproduce el simulador del proyecto.
- **Machine Learning para pronóstico de demanda.** El pronóstico de carga eléctrica de corto plazo es un problema clásico de series temporales. En la literatura se emplean desde modelos estadísticos (promedios móviles, ARIMA) hasta modelos de aprendizaje automático (árboles, bosques aleatorios, máquinas de soporte vectorial, redes neuronales) que usan como variables la hora, el día, la temperatura y el consumo histórico **(verificar fuente para referencias bibliográficas concretas)**. Los modelos basados en árboles son populares por su buen desempeño con datos tabulares y su relativa facilidad de interpretación.
- **Medidores inteligentes y analítica de texto.** La disponibilidad de lecturas horarias y de canales digitales de atención a usuarios hace posible combinar series de consumo con análisis de texto de reportes **(verificar fuente)**.
- **Trazabilidad con Blockchain.** El encadenamiento de bloques con funciones hash se ha propuesto para registrar de forma verificable datos y resultados sensibles a manipulación. En este proyecto se usa una versión didáctica, privada y de un solo nodo (ver `05_blockchain.md`).

## Justificación

- **Técnica.** El problema integra en un solo flujo las cuatro áreas de la materia: datos y preparación, aprendizaje automático supervisado y no supervisado, procesamiento de lenguaje natural y Blockchain.
- **Operativa.** Un pronóstico horario con error bajo permite planear; la agrupación de perfiles señala días atípicos; el análisis de reportes prioriza fallas y equipos problemáticos; el reporte automático ahorra tiempo.
- **Confianza.** Registrar el hash del reporte y de las predicciones permite una auditoría posterior: cualquier modificación de un solo carácter se detecta.
- **Académica.** Se cumplen los nueve objetivos específicos del profesor con un prototipo explicable y reproducible (semilla fija `config.SEMILLA = 42`).
- **Ética y legal.** Al usar datos simulados no se maneja información personal ni confidencial (sección 13 del documento del profesor).

## Objetivo general

Diseñar y desarrollar un prototipo de sistema inteligente que procese datos de consumo eléctrico y reportes de usuarios, prediga el consumo horario y agrupe perfiles mediante Machine Learning, analice el texto mediante PNL (sentimiento, entidades y generación de reportes) y garantice la integridad y trazabilidad de los resultados mediante una Blockchain privada, para apoyar la toma de decisiones de un operador de eficiencia energética.

## Objetivos específicos

Se alinean uno a uno con nueve objetivos específicos.

| # | Objetivo específico del proyecto | Dónde se cumple |
|---|---|---|
| 1 | Identificar y analizar un problema simulado de gestión del consumo eléctrico en 10 instalaciones de Veracruz. | Este documento |
| 2 | Generar, organizar y preparar un dataset de consumo horario (44,117 filas crudas) y un corpus de 4,000 reportes de usuarios. | `02_dataset.md`, `src/data/` |
| 3 | Diseñar una arquitectura que muestre el flujo desde las fuentes hasta el dashboard. | `06_arquitectura.md` |
| 4 | Implementar modelos de ML: regresión supervisada del consumo (Random Forest y alternativas) y agrupamiento K-Means de perfiles diarios. | `03_modelo_ml.md`, `src/ml/` |
| 5 | Implementar PNL: representación (tokenización, stopwords, stemming, lematización, TF-IDF), sentimiento, NER y generación de lenguaje. | `04_pnl.md`, `src/nlp/` |
| 6 | Implementar una Blockchain privada simulada con SHA-256 y prueba de trabajo que registre y verifique la integridad de reportes, predicciones, dataset, modelo y métricas. | `05_blockchain.md`, `src/blockchain/` |
| 7 | Desarrollar un dashboard web (Flask) para visualizar resultados, predecir y verificar registros. | `app/`, `06_arquitectura.md` |
| 8 | Integrar los componentes en un prototipo con un pipeline reproducible. | `pipeline.py`, `src/reporte/generar_reporte.py` |
| 9 | Evaluar resultados y establecer limitaciones, conclusiones y mejoras. | `03_modelo_ml.md`, `07_conclusiones.md` |

## Alcance

**Incluye:**

- Simulación de 182 días de consumo horario para 10 instalaciones (43,680 registros reales más 437 duplicados inyectados) y de 4,000 reportes de texto en español.
- Preparación de datos documentada: nulos, duplicados, atípicos, codificación, escalado, creación de variables y división temporal.
- Pronóstico de consumo horario por regresión supervisada (1 hora hacia adelante, apoyado en el consumo reciente) y un modo de pronóstico de 24 horas.
- Agrupamiento de perfiles diarios con K-Means y lista de días atípicos.
- Análisis de sentimiento (3 clases), reconocimiento de 10 tipos de entidades y generación de reportes por plantillas.
- Blockchain privada de un solo nodo con SHA-256, PoW (4 ceros) y verificación de archivos y contenidos.
- Dashboard web local.

**No incluye:** datos reales de CFE ni de medidores, despliegue en la nube, autenticación de usuarios, red blockchain distribuida, modelos de lenguaje neuronales ni predicción de precios o tarifas.

## Limitaciones

1. **Datos simulados.** Las métricas reflejan la regularidad del simulador y no deben extrapolarse a datos reales.
2. **Periodo corto.** Solo 182 días (enero a principios de julio de 2026): no se observa un ciclo anual completo ni invierno-verano completo.
3. **Dependencia del consumo reciente.** El modelo depende en gran medida de `consumo_lag_1h` (importancia 0.958). Es un pronóstico de corto plazo, no de largo plazo.
4. **Intervalo de predicción optimista.** Los percentiles 5-95 de los árboles cubren solo 77.7 % de los valores de prueba (nominal 90 %).
5. **PNL sobre texto sintético.** El NER y el clasificador de sentimiento tienen métricas altas porque los textos se generaron con plantillas (ver `04_pnl.md`).
6. **Blockchain didáctica.** Un solo nodo, sin consenso distribuido ni identidades criptográficas.
7. **Librerías restringidas.** Solo pandas, numpy, scikit-learn, scipy, matplotlib, seaborn, joblib, Flask y la biblioteca estándar; los stemmers, lematizador y NER son implementaciones propias y simplificadas.

## Usuarios potenciales

- Gestor o analista de eficiencia energética de una organización con varias instalaciones.
- Administradores de edificios, escuelas, plazas comerciales y plantas industriales.
- Personal de atención a usuarios que recibe quejas y reportes.
- Auditores internos o externos que necesiten verificar la integridad de un reporte o de una predicción.
- Estudiantes y docentes como material didáctico de integración de ML, PNL y Blockchain.

## Descripción general de la solución

SIAPE es un sistema de extremo a extremo con siete etapas:

1. **Fuentes y generación.** `src/data/generar_datos.py` simula el consumo horario con clima de Veracruz, perfiles por tipo y problemas de calidad inyectados; `src/data/generar_reportes.py` simula 4,000 reportes.
2. **Preparación.** `src/data/preparar_datos.py` limpia, imputa, crea variables y divide temporalmente.
3. **Machine Learning.** `src/ml/entrenar_modelo.py` compara cinco modelos y una línea base y selecciona Random Forest (R² = 0.9948, MAE = 1.220 kWh en prueba); `src/ml/clustering.py` agrupa perfiles diarios en k = 6 (silhouette 0.662); `src/ml/predecir.py` ofrece la API de predicción.
4. **PNL.** `src/nlp/` preprocesa, clasifica sentimiento (SVM lineal, F1-macro 0.938), extrae entidades (F1 global 0.999) y redacta resúmenes.
5. **Reporte.** `src/reporte/generar_reporte.py` integra los resultados en un reporte textual y calcula su SHA-256.
6. **Blockchain.** `src/blockchain/blockchain.py` registra el hash del reporte, de las predicciones, del dataset, del modelo y de las métricas.
7. **Dashboard.** `app/app.py` (Flask) presenta KPIs, gráficas, predicción, análisis de texto y verificación de integridad.

Los detalles de cada componente están en los documentos `02` a `07`. Las cifras de este documento provienen de `models/metricas_modelos.json`, `models/metricas_pnl.json`, `models/clusters_resumen.json` y `outputs/evidencias/`.
