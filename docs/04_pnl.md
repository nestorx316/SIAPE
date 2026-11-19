# Entregable 4. Componente de Procesamiento de Lenguaje Natural (PNL)

Fuentes de las cifras: `models/metricas_pnl.json`, `outputs/evidencias/pnl_entrenamiento.txt`, `pnl_ejemplos.txt`, `generacion_reportes.txt`. Código: `src/nlp/` (`procesamiento_texto.py`, `sentimiento.py`, `entidades.py`, `generacion.py`, `analizador.py`).

Restricción técnica: el entorno solo permite pandas, numpy, scikit-learn, scipy, matplotlib, seaborn, joblib, Flask y la biblioteca estándar; no hay nltk ni spaCy. Por eso la tokenización, las stopwords, el stemmer, el lematizador y el NER se implementaron a mano, y solo el TF-IDF y los clasificadores usan scikit-learn. Se implementaron las **cuatro** funcionalidades de PNL: representación, sentimiento, NER y generación de lenguaje.

## 1. Conjunto de textos (corpus)

- **4,000 reportes de usuarios en español** (`data/raw/reportes_usuarios_raw.csv`), simulados con plantillas combinatorias (procedimiento detallado en `02_dataset.md`, sección 1.3).
- Columnas: `id_reporte, id_instalacion, fecha_hora, canal, texto, sentimiento_etiqueta, entidades_esperadas`.
- Etiquetas: negativo 1,531, positivo 1,343, neutral 1,126. Canales: app, portal, correo, llamada.
- Dificultad deliberada: 10.1 % de casos mixtos o ambiguos, 2.9 % de ruido de etiqueta, errores ortográficos, negaciones, emojis contradictorios, alias coloquiales.
- Textos únicos: 100 %.
- Procesado: `data/processed/reportes_procesados.csv` (incluye `texto_limpio`, `sentimiento_predicho`, `confianza`, `entidades`, `n_entidades`).

## 2. Preparación del texto (`procesamiento_texto.preprocesar`)

La función `preprocesar(texto)` devuelve un diccionario con `normalizado, tokens, sin_stopwords, stems, lemas`.

1. **Normalización** (`normalizar`). Minúsculas; eliminación de URLs y correos; emojis convertidos en marcadores (`emojipositivo`, `emojinegativo`); enmascaramiento de folios, códigos de instalación, montos, kWh, horas, fechas numéricas y números (son entidades, no aportan al sentimiento); espacios limpios.
2. **Tokenización** (`tokenizar`). Expresión regular de palabras con acentos, ñ y ü.
3. **Eliminación de stopwords** (`quitar_stopwords`). Lista propia de ~298 formas (incluye variantes sin acento). **Conserva las negaciones** (`no, nunca, sin, ni, tampoco, nada, nadie, ningún...`) porque cambian el sentido ("no funciona" frente a "funciona"), y también "pero", "muy", "más", "menos", "poco", "mucho", "solo" porque aportan al sentimiento.
4. **Stemming** (`stem`). Stemmer ligero por sufijos inspirado en Snowball español, simplificado: quita acentos, un sufijo derivativo, un sufijo verbal y la terminación de género y número, dejando raíces de al menos 3 letras. Ejemplos del corpus: *quejar* a `quej`, *instalación* a `instal`, *excelente* a `excelent`.
5. **Lematización** (`lematizar`). Diccionario de verbos frecuentes con conjugaciones generadas por reglas regulares más reglas de plural. Ejemplos: *necesito* a `necesitar`, *pérdidas* a `pérdida`, *cobren* a `cobrar`, *luminarias* a `luminaria`. Es incompleta: *logramos*, *exijo* o *causan* quedan sin lematizar (ver ejemplos 1, 2 y 3 más abajo).

**Diferencia entre stemming y lematización:** el stemming recorta sufijos y produce raíces que pueden no ser palabras (`excelent`, `grac`); la lematización busca la forma canónica real (`gracias`, `ahorrar`). Aquí el modelo usa los **stems** porque son robustos y no requieren un diccionario completo.

## 3. Representación (TF-IDF)

- El texto preprocesado (cadena de stems sin stopwords) se transforma con `TfidfVectorizer(ngram_range=(1, 2), min_df=2, sublinear_tf=True)`.
- **TF-IDF**: pondera cada término por su frecuencia en el documento (TF, con escala logarítmica por `sublinear_tf`) multiplicada por la rareza del término en el corpus (IDF). Los términos que aparecen en casi todos los textos pesan poco y los distintivos pesan más.
- **Unigramas + bigramas**: los bigramas capturan expresiones como `no qued` o `sin cort` (en los términos más influyentes de las clases negativa y positiva).
- Términos más influyentes por clase (`terminos_top_por_clase`, calculados con Regresión Logística sobre entrenamiento; el peso es el coeficiente):

| Clase | Términos (raíz y peso) |
|---|---|
| negativo | `no` (5.221), `emojinegat` (3.378), `no qued` (2.767), `nadie` (2.56), `rat` (2.509), `lleg` (2.357), `mal` (2.317), `apagon` (2.237) |
| neutral | `requier` (4.309), `inform` (4.032), `program` (3.654), `solicit` (3.634), `pregunt` (3.022) |
| positivo | `excelent` (3.769), `muy` (3.378), `funcion` (3.045), `gran` (2.823), `grac` (2.619), `felicit` (2.597), `maravill` (2.386) |

La más influyente en la clase negativa es la negación `no`, lo que justifica haberla conservado.

![Términos TF-IDF](../outputs/figuras/pnl_terminos_tfidf.png)

## 4. Análisis de sentimiento (`src/nlp/sentimiento.py`)

**Clases:** positivo, negativo, neutral. **División:** 80 % entrenamiento (3,200) y 20 % prueba (800), estratificada por clase, semilla 42. **Selección:** mayor F1-macro promedio en validación cruzada estratificada de 5 pliegues sobre entrenamiento (`f1_macro_cv`), de modo que el conjunto de prueba no interviene en la elección.

### Comparación de los tres clasificadores

| Clasificador | Exactitud | Precisión macro | Recall macro | F1-macro (prueba) | F1-macro CV(5) |
|---|---|---|---|---|---|
| Naive Bayes multinomial (alpha = 0.3) | 0.9337 | 0.9357 | 0.9344 | 0.9350 | 0.9289 |
| Regresión Logística (C = 5.0) | 0.9275 | 0.9305 | 0.9284 | 0.9294 | 0.9294 |
| **SVM lineal** (`LinearSVC`, C = 0.5, calibrado con `CalibratedClassifierCV`, cv = 3) | **0.9363** | **0.9384** | **0.9370** | **0.9377** | **0.9311** |

Los tres son muy parecidos (diferencias menores a 1 punto). Se eligió el **SVM lineal**, que además es el mejor en prueba. Se calibra para obtener probabilidades (que alimentan la columna `confianza`). El modelo guardado (`models/modelo_sentimiento.joblib`) se reentrena con los 4,000 reportes.

![Comparación de clasificadores](../outputs/figuras/pnl_comparacion_clasificadores.png)

### Reporte por clase (SVM lineal, prueba, n = 800)

| Clase | Precisión | Recall | F1 | Soporte |
|---|---|---|---|---|
| negativo | 0.934 | 0.931 | 0.933 | 306 |
| neutral | 0.968 | 0.947 | 0.957 | 225 |
| positivo | 0.913 | 0.933 | 0.923 | 269 |
| exactitud global | | | 0.936 | 800 |

### Matriz de confusión (filas = real, columnas = predicho)

| Real \ Predicho | negativo | neutral | positivo |
|---|---|---|---|
| **negativo** | 285 | 3 | 18 |
| **neutral** | 6 | 213 | 6 |
| **positivo** | 14 | 4 | 251 |

![Matriz de confusión](../outputs/figuras/pnl_matriz_confusion.png)

La confusión principal ocurre entre positivo y negativo (18 + 14 casos), que corresponde a los reportes mixtos con "pero" o "aunque" y al ruido de etiqueta. La exactitud no llega a 1.0 porque ~10 % son mixtos y ~3 % de las etiquetas tienen ruido deliberado: ese techo es esperable. (El acierto de 0.968 sobre todo el corpus, reportado en `pnl_ejemplos.txt`, incluye datos de entrenamiento y por eso es más alto; la cifra de referencia es 0.936.)

![Distribución del sentimiento](../outputs/figuras/pnl_distribucion_sentimiento.png)

## 5. Reconocimiento de entidades nombradas (NER, `src/nlp/entidades.py`)

### Tipos de entidad (10)

`DISPOSITIVO, LUGAR, ORGANIZACION, FECHA, HORA, CODIGO_INSTALACION, FOLIO, MONTO, ENERGIA, PERSONA`.

### Método híbrido (4 pasos)

1. Se crea una copia **plegada** del texto (minúsculas y sin acentos, de la misma longitud) para que los diccionarios coincidan con "cordoba" o "CFE" y las posiciones sirvan para el texto original.
2. **Diccionarios (gazetteers)**: `DISPOSITIVO` (`config.DISPOSITIVOS`, admite plurales), `LUGAR` (`config.CIUDADES`) y `ORGANIZACION` (`config.ORGANIZACIONES`).
3. **Expresiones regulares**: `FECHA` ("12 de marzo", "15/03/2026", "ayer"), `HORA` ("a las 18:30"), `CODIGO_INSTALACION` (`RES-001`), `FOLIO` (`FOL-123456`), `MONTO` ($1,250), `ENERGIA` ("350 kWh") y `PERSONA` (título "Sr./Sra./Ing./Lic." más nombre con mayúscula).
4. **Resolución de traslapes**: ante coincidencias solapadas se conserva la más larga.

`extraer_entidades(texto)` devuelve `[{"texto","tipo","inicio","fin"}]`.

### Métricas (`ner` en `metricas_pnl.json`)

Evaluación contra las `entidades_esperadas` del generador: un acierto es el mismo texto (en minúsculas) y el mismo tipo dentro del reporte.

| Tipo | Precisión | Recall | F1 | Soporte |
|---|---|---|---|---|
| DISPOSITIVO | 1.000 | 0.995 | 0.998 | 1,998 |
| LUGAR | 1.000 | 0.992 | 0.996 | 1,729 |
| ORGANIZACION | 1.000 | 1.000 | 1.000 | 1,031 |
| FECHA | 1.000 | 1.000 | 1.000 | 1,351 |
| HORA | 1.000 | 1.000 | 1.000 | 825 |
| CODIGO_INSTALACION | 1.000 | 1.000 | 1.000 | 763 |
| FOLIO | 1.000 | 1.000 | 1.000 | 1,112 |
| MONTO | 1.000 | 1.000 | 1.000 | 779 |
| ENERGIA | 1.000 | 1.000 | 1.000 | 846 |
| PERSONA | 1.000 | 1.000 | 1.000 | 912 |
| **Global (micro)** | **1.000** | **0.998** | **0.999** | 11,346 |

### Por qué las métricas son tan altas (y por qué no deben tomarse como rendimiento real)

- El texto es **simulado con plantillas**: las entidades se insertan con formatos regulares que las mismas expresiones regulares y diccionarios cubren. La evaluación es en parte circular: el sistema reconoce los patrones con los que se generó el corpus.
- Las únicas fallas (recall menor a 1) provienen de los casos adversos que se inyectaron a propósito: alias coloquiales ("refri", "clima") y ciudades con error de dedo que no están en los diccionarios.
- Con reportes reales (abreviaturas, nombres propios nuevos, formatos libres de fecha) el desempeño sería menor; un modelo estadístico (spaCy, transformers) sería el siguiente paso (ver `07_conclusiones.md`).

## 6. Generación de lenguaje natural (`src/nlp/generacion.py`)

**Método:** plantillas con **selección condicionada** por reglas si-entonces. No usa ningún modelo de lenguaje; es determinista (mismos datos generan el mismo texto), lo que permite calcular un hash estable del reporte para la Blockchain.

- `generar_resumen(datos: dict) -> str` redacta un reporte de varios párrafos (`_p_alcance`, `_p_modelo`, `_p_prediccion`, `_p_clusters`, `_p_pnl`, `_p_recomendaciones`, `_p_cierre`). Todas las claves de entrada son opcionales; si falta un dato, se omite su párrafo.
- **Reglas condicionadas:** se califica el R² (`_calificar_r2`) y el silhouette (`_calificar_silhouette`) con palabras; si el porcentaje de reportes negativos supera 40 % se emite una **alerta**; los dispositivos más mencionados activan **recomendaciones de optimización** (diccionario `RECOMENDACIONES`, p. ej. aire acondicionado: ajustar termostatos a 24-25 °C y dar mantenimiento a filtros).
- `resumir_reportes(df) -> str` resume un DataFrame de reportes: total, distribución de sentimiento, instalación con más reportes negativos, canal más usado y equipos más mencionados.
- El reporte final lo ensambla `src/reporte/generar_reporte.py`, que llama a `generar_resumen` con las métricas del modelo, la predicción, los clusters y los resultados de PNL.

## 7. Analizador integral (`src/nlp/analizador.py`)

- `analizar_texto(texto)` devuelve `{"preprocesado": {...}, "sentimiento": {...}, "entidades": [...]}` y se usa en el dashboard para analizar texto libre.
- `procesar_reportes()` procesa los 4,000 reportes, escribe `data/processed/reportes_procesados.csv` (11,322 entidades extraídas en total), las figuras `pnl_*.png` y `outputs/evidencias/pnl_ejemplos.txt`.
- Distribución de sentimiento predicho (todo el corpus): negativo 1,539, positivo 1,349, neutral 1,112.

![Entidades más frecuentes](../outputs/figuras/pnl_entidades_frecuentes.png)

## 8. Ejemplos de entrada y salida

Tomados de `outputs/evidencias/pnl_ejemplos.txt` (A: reportes del corpus; B: frases nuevas no vistas en el corpus). Se muestran los campos principales; el archivo contiene también tokens, stems y lemas de cada ejemplo.

| # | Entrada | Sentimiento (probabilidad) | Entidades |
|---|---|---|---|
| 1 | "Queja/sugerencia: Logramos ahorrar 32,456 kWh gracias al ajuste. Mi código de instalación es EDU-002. Sigan así." (real: positivo) | positivo (0.9744) | `[ENERGIA] 32,456 kWh`; `[CODIGO_INSTALACION] EDU-002` |
| 3 | "Hola, Los cortes de luz en Orizaba son interminables y nos causan pérdidas. Es inaceptable que cobren $71,960 por un consumo que no corresponde." (real: negativo) | negativo (0.992) | `[LUGAR] Orizaba`; `[MONTO] $71,960` |
| 4 | "cliente indica que se botó el ventilador y hubo riesgo de incendio. el problema empezó el 6 de enero. 😡" (real: negativo) | negativo (0.9755) | `[DISPOSITIVO] ventilador`; `[FECHA] 6 de enero` |
| 5 | "Buen día, Necesito saber si CFE ofrece auditorías energéticas. Registro la lectura del refrigerador: 2,458 kWh. Les escribo desde Córdoba. Quedo al pendiente." (real: neutral) | neutral (0.9753) | `[ORGANIZACION] CFE`; `[DISPOSITIVO] refrigerador`; `[ENERGIA] 2,458 kWh`; `[LUGAR] Córdoba` |
| 6 | "Estimado personal: Queda registrado el mantenimiento del minisplit como parte del programa. Se solicita informar el estatus del folio FOL-843119. Lectura actual: 14,842 kwh. 😠" (real: neutral) | neutral (0.983) | `[DISPOSITIVO] minisplit`; `[FOLIO] FOL-843119`; `[ENERGIA] 14,842 kwh` |
| 9 | "El transformador de IND-001 hizo un ruido horrible ayer a las 22:15 y CFE no responde, folio FOL-445566. Pésimo servicio!!" | negativo (0.9819) | `[DISPOSITIVO] transformador`; `[CODIGO_INSTALACION] IND-001`; `[FECHA] ayer`; `[HORA] a las 22:15`; `[ORGANIZACION] CFE`; `[FOLIO] FOL-445566` |
| 10 | "Gracias al Ing. Carlos Ruiz por reparar el minisplit en Xalapa, ahora funciona de maravilla 👍" | positivo (0.9638) | `[PERSONA] Ing. Carlos Ruiz`; `[DISPOSITIVO] minisplit`; `[LUGAR] Xalapa` |
| 11 | "Solicito el duplicado del recibo de $1,250 correspondiente a 350 kWh del 15/03/2026." | neutral (0.9698) | `[MONTO] $1,250`; `[ENERGIA] 350 kWh`; `[FECHA] 15/03/2026` |
| 12 | "El servicio fue rápido, pero el aire acondicionado sigue sin enfriar en Boca del Río" | negativo (0.5545; positivo 0.4322) | `[DISPOSITIVO] aire acondicionado`; `[LUGAR] Boca del Río` |
| 13 | "No tengo ninguna queja, el compresor opera sin fallas desde el mantenimiento" | positivo (0.8989) | `[DISPOSITIVO] compresor` |

### Ejemplo completo de preprocesamiento (ejemplo 9)

- Entrada: *El transformador de IND-001 hizo un ruido horrible ayer a las 22:15 y CFE no responde, folio FOL-445566. Pésimo servicio!!*
- Normalizado: `el transformador de hizo un ruido horrible ayer a las y cfe no responde, folio . pésimo servicio!!`
- Sin stopwords: `transformador, ruido, horrible, cfe, no, responde, pésimo, servicio`
- Stems: `transform, ruid, horr, cfe, no, respond, pesim, servic`
- Lemas: `transformador, ruido, horrible, cfe, no, responder, pésimo, servicio`

### Casos que destacan

- **Ejemplo 12 (mixto):** la confianza es baja (0.55 frente a 0.43), señal correcta de ambigüedad. **Ejemplo 13:** con negaciones ("no tengo ninguna queja", "sin fallas") el modelo acierta gracias a conservar las negaciones y a los bigramas.
- **Ejemplo 2** (mixto: queja de fuga de corriente más ahorro) se clasifica como positivo con probabilidad 0.7281 y 0.2716 para negativo, siguiendo la regla del generador de etiquetar según la cláusula posterior al conector. **Ejemplos 7 y 8** (en `pnl_ejemplos.txt`) son los dos errores del clasificador frente a la etiqueta simulada. En el 7 la etiqueta real es "positivo" pero el texto describe ruido y olor a quemado y cortes de luz, así que la predicción "negativo" es razonable: es un caso de ruido de etiqueta deliberado. En el 8 ("Encantado con la rapidez... Muchas gracias") la etiqueta es "neutral" y se predijo "positivo" (probabilidad 0.8165).

## 9. Evidencia de funcionamiento

- `outputs/evidencias/pnl_entrenamiento.txt`: comparación de clasificadores, reporte de clasificación, matriz de confusión, términos influyentes y evaluación del NER.
- `outputs/evidencias/pnl_ejemplos.txt`: ejemplos entrada a salida completos.
- `outputs/evidencias/generacion_reportes.txt`: estadísticas del corpus.
- Figuras: `pnl_matriz_confusion.png`, `pnl_comparacion_clasificadores.png`, `pnl_distribucion_sentimiento.png`, `pnl_entidades_frecuentes.png`, `pnl_terminos_tfidf.png`.
- Reproducción: `python -m src.data.generar_reportes`, `python -m src.nlp.sentimiento` y `python -m src.nlp.analizador`.
