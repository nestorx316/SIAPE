# Conclusiones, limitaciones y trabajo futuro

Fuentes de las cifras: `models/metricas_modelos.json`, `models/clusters_resumen.json`, `models/metricas_pnl.json` y `outputs/evidencias/`.

## 1. Resultados

### Datos
- Se generaron 43,680 registros horarios verdaderos de 10 instalaciones (44,117 con 437 duplicados inyectados) y 4,000 reportes de usuarios. La preparación detectó y corrigió los problemas inyectados: 437 duplicados, 24 variantes de texto de `tipo_instalacion` normalizadas a 4, 257 atípicos y los nulos (~2 % por columna), para llegar a 42,000 filas sin nulos y 27 columnas.

### Machine Learning
- **Random Forest** fue el modelo seleccionado (menor RMSE en validación cruzada temporal: 2.497). En el conjunto de prueba (8,740 horas) obtuvo **MAE 1.220 kWh, RMSE 2.493 kWh, R² 0.9948, MAPE 6.29 %**.
- Frente a la línea base "consumo de hace 24 h" (MAE 6.170, RMSE 14.572, R² 0.8213), reduce el MAE en 80.2 % y el RMSE en 82.9 %.
- Por tipo de instalación el R² va de 0.9545 (residencial) a 0.9881 (comercial); el MAPE, de 4.10 % (industrial) a 7.74 % (residencial).
- El modelo depende casi totalmente de `consumo_lag_1h` (importancia 0.9585): es un pronóstico de corto plazo.
- El intervalo de predicción (percentiles 5-95 de los árboles) cubrió 77.7 % de los valores de prueba, por debajo del 90 % nominal.
- **K-Means**: k = 6 con silhouette 0.662; los clusters separan por completo los tipos de instalación (1,750 perfiles diarios) y detectan días de baja actividad de las escuelas y días residenciales atípicos.

### PNL
- **Sentimiento:** SVM lineal con TF-IDF (unigramas y bigramas): exactitud 0.936 y F1-macro 0.938 en prueba (800 reportes), con confusiones concentradas entre positivo y negativo (32 de los 51 errores) por los reportes mixtos y el ruido de etiqueta.
- **NER híbrido:** 10 tipos de entidad, F1 global 0.999 (precisión 1.000, recall 0.998), sobre texto simulado.
- **NLG:** reportes en español con reglas condicionadas, deterministas (hash estable).

### Blockchain
- Cadena privada simulada con SHA-256 y PoW de 4 ceros. En la prueba de alteración, 10 de 10 comprobaciones fueron correctas: la modificación de un solo carácter de un documento, la alteración de una predicción y la alteración de un bloque (incluso con re-minado, por el enlace roto con el siguiente) fueron detectadas.

### Integración
- El flujo completo conecta datos, ML, PNL, reporte, hash, cadena y dashboard (ver `06_arquitectura.md`), cumpliendo los nueve objetivos específicos (tabla en `01_analisis_del_problema.md`).

## 2. Limitaciones

1. **Datos simulados.** Las métricas altas reflejan la regularidad del simulador; no pueden extrapolarse a medidores reales, que tienen más variabilidad, fallas de comunicación y comportamientos no previstos.
2. **Periodo de 182 días.** No se cubre un año completo; no se evalúa estacionalidad de invierno ni de temporada de calor completa. Las variables `mes_sin`, `mes_cos` y `mes` casi no aportan (< 0.0001).
3. **Modelo dependiente de la hora anterior.** Con 95.85 % de importancia en `consumo_lag_1h`, el pronóstico a 24 h (donde el modelo reutiliza sus predicciones) acumula error. **No se evaluó** el modelo sin `consumo_lag_1h` ni un horizonte multi-paso formal.
4. **Intervalos subcalibrados** (cobertura 77.7 % frente a 90 %).
5. **Un único conjunto de prueba temporal;** sin repeticiones con distintas ventanas más allá de la validación cruzada temporal de 3 cortes.
6. **PNL.**
   - Stemmer y lematizador propios y simplificados (el lematizador es incompleto).
   - El sentimiento se entrenó con textos de plantillas y etiquetas del generador; no se validó con anotadores humanos.
   - NER con diccionarios y regex: no reconoce alias ("refri", "clima"), ciudades nuevas ni fechas informales; su evaluación es parcialmente circular.
7. **Blockchain didáctica.** Un nodo, sin consenso, sin firmas digitales, marca de tiempo local; protege contra alteraciones parciales pero no contra la reescritura total de la cadena por quien controle el archivo. Prueba integridad, no veracidad.
8. **Dashboard local,** sin autenticación, usuarios ni despliegue.
9. **Clustering.** El tipo de instalación explica por completo los clusters, de modo que aporta poca información nueva sobre comportamientos dentro de un mismo tipo; la interpretación del cluster 4 residencial (¿fin de semana?) no se verificó.

## 3. Conclusiones

1. Es viable resolver el problema del operador con un flujo integrado de datos, ML, PNL y Blockchain usando únicamente librerías estándar de Python, con resultados reproducibles (semilla 42).
2. Un **modelo de bosque aleatorio** supera ampliamente a la línea base y a los demás algoritmos comparados (regresión lineal, árbol, KNN, SVR) para el pronóstico de consumo horario a corto plazo; la validación cruzada temporal y la división cronológica son esenciales para no sobrestimar el desempeño.
3. El resultado más informativo no es el R² alto sino la **interpretación**: el consumo reciente domina la predicción, la línea base de "ayer" no es suficiente, y la cobertura de los intervalos es menor a la nominal, de modo que el modelo debe usarse con criterio.
4. El **PNL** permite convertir reportes no estructurados en indicadores (sentimiento, equipos y lugares más mencionados) y en recomendaciones automáticas, aunque sus métricas deben leerse con cautela por ser datos sintéticos.
5. El registro de **hashes en una cadena de bloques** es sencillo y efectivo para detectar modificaciones de reportes, predicciones, datasets y modelos; ofrece trazabilidad cronológica aunque, con un solo nodo, no sustituye una red descentralizada.
6. Se logró un prototipo coherente, no una colección de programas aislados: cada resultado del análisis termina en un reporte verificable y visible en el dashboard.

## 4. Trabajo futuro

### Datos
- Usar **datos reales** de consumo de CFE o de **medidores inteligentes**, y de clima de estaciones meteorológicas (por ejemplo, de la CONAGUA o el SMN).
- Ampliar el periodo a uno o más años, incluir tarifas, horarios de punta y calendario laboral real.
- Procesar flujos en tiempo casi real (ingesta por API o MQTT) y almacenar en una base de datos (PostgreSQL o series de tiempo).

### Machine Learning
- Hacer una **ablación** sin `consumo_lag_1h` y evaluar horizontes de 24 a 168 h (pronóstico multi-paso, modelos directos por horizonte).
- Probar modelos de gradiente (XGBoost, LightGBM), redes recurrentes o Transformers para series de tiempo, y modelos probabilísticos o *conformal prediction* para intervalos calibrados.
- Detección de anomalías en línea (por ejemplo, con Isolation Forest o con el error de predicción) y alertas.
- Explicabilidad (SHAP) por instalación y optimización (recomendaciones de ajuste de carga).

### PNL
- Sustituir las reglas por **spaCy** (NER estadístico en español) y **transformers** en español (por ejemplo, BETO o RoBERTa multilingüe) afinados para sentimiento; usar embeddings en lugar de TF-IDF.
- Anotación con humanos para evaluar sobre texto real; incluir ironía, abreviaturas y errores ortográficos reales.
- Generación de reportes con un modelo de lenguaje grande con restricciones para evitar alucinaciones, conservando la verificabilidad.

### Blockchain
- **Red multi-nodo** con consenso (por ejemplo, PBFT o Raft) o plataformas empresariales como **Hyperledger Fabric**; o **testnet de Ethereum** con un contrato inteligente que guarde los hashes.
- Firmas digitales (ECDSA/Ed25519) para identificar quién registra, sellado de tiempo confiable y árbol de Merkle para registrar lotes.
- Anclaje periódico del hash de la cadena en una red pública.

### Despliegue y producto
- Contenedores (Docker), servidor WSGI (gunicorn) y despliegue en la nube; base de datos, autenticación y roles (operador, auditor).
- Pruebas automatizadas ampliadas (CI), monitoreo del modelo (deriva de datos) y reentrenamiento programado.
- Alertas por correo o mensajería y panel móvil.
- Seguridad y privacidad: cifrado, bitácoras de acceso y cumplimiento de la normativa de datos personales.
