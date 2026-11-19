# Entregable 5. Componente Blockchain

Código: `src/blockchain/blockchain.py` (cadena, bloques y verificación), `src/blockchain/demo_alteracion.py` (prueba de alteración), `tests/test_blockchain.py` (pruebas unitarias). Evidencia: `outputs/evidencias/blockchain_prueba_alteracion.txt`. Configuración: `config.CADENA_JSON` (`blockchain_data/cadena.json`) y `config.DIFICULTAD_POW = 4`. Solo usa la biblioteca estándar de Python.

## 1. Tipo de Blockchain

**Blockchain privada y permisionada, simulada, de un solo nodo.**

- **Privada / permisionada:** solo la institución operadora de SIAPE escribe bloques; cualquiera con acceso al archivo `cadena.json` puede verificar, pero no existe una red pública.
- **Simulada:** la cadena es un archivo JSON local, no una red de nodos.
- **Un solo nodo validador:** no hay consenso distribuido; el propio sistema valida y agrega bloques.
- **Prueba de trabajo (PoW) didáctica:** cada bloque se "mina" hasta que su hash empiece con 4 ceros hexadecimales. En un solo nodo la PoW no aporta seguridad adicional; se incluye para mostrar el concepto y encarecer la reescritura de la cadena.

La información del operador es privada y el objetivo es demostrar la integridad y trazabilidad con una implementación explicable de principio a fin.

## 2. Arquitectura

```
 Documento / resultado
 (reporte .txt, predicción dict, dataset .csv, modelo .joblib, métricas .json)
          |
          v
   SHA-256 (sha256_archivo / sha256_json / sha256_bytes)
          |  hash_documento (64 caracteres hexadecimales)
          v
   Blockchain.agregar_registro(tipo, nombre, hash_documento, metadatos)
          |  crea Bloque(indice, marca_tiempo, datos, hash_anterior)
          v
   Bloque.minar(dificultad)  -> busca nonce hasta hash con 4 ceros
          |
          v
   Cadena en memoria -> Blockchain.guardar() -> blockchain_data/cadena.json (escritura atómica)

 Verificación posterior:
   archivo / contenido actual -> nuevo SHA-256 -> comparar con hash_documento registrado
        -> ÍNTEGRO / MODIFICADO / NO REGISTRADO
   validar_cadena() -> recalcula hash, PoW, enlaces e índices de cada bloque
```

Componentes principales:

| Elemento | Responsabilidad |
|---|---|
| `Bloque` | Estructura de un bloque; `calcular_hash()`, `minar()`, `a_dict()`, `desde_dict()` |
| `Blockchain` | Cadena persistida; `agregar_registro`, `registrar_archivo`, `registrar_contenido`, `validar_cadena`, `buscar_por_hash`, `buscar_por_nombre`, `verificar_archivo`, `verificar_contenido`, `a_lista`, `guardar`, `cargar` |
| `sha256_bytes`, `sha256_archivo`, `sha256_json` | Funciones de hash (el archivo se lee por bloques de 1 MiB) |
| `demo_alteracion.ejecutar_demo()` | Prueba reproducible en un directorio temporal |

La escritura del JSON es **atómica** (se escribe a un `.tmp` y se renombra con `os.replace`) y está protegida con un candado de hilos (`threading`), de modo que el dashboard puede escribir concurrentemente sin dejar el archivo a medias. Si el JSON está dañado, `cargar()` lanza un error y no lo sobrescribe (puede ser evidencia de alteración).

## 3. Estructura del bloque

| Campo | Descripción |
|---|---|
| `indice` | Número consecutivo (0 = génesis) |
| `marca_tiempo` | Fecha y hora ISO 8601 en UTC |
| `datos` | `{"tipo", "nombre", "hash_documento", "metadatos"}` |
| `hash_anterior` | Hash del bloque previo (el génesis usa 64 ceros) |
| `nonce` | Número que se prueba hasta cumplir la PoW |
| `hash` | SHA-256 del bloque |

`datos.tipo` toma los valores `reporte`, `prediccion`, `modelo`, `dataset`, `metricas` (o `genesis` para el bloque 0).

**Ejemplo de bloque** (generado con `Blockchain` en un directorio temporal, dificultad 4; el `hash_documento` es el del documento de demostración de la evidencia):

```json
{
  "indice": 1,
  "marca_tiempo": "2026-11-14T21:10:45+00:00",
  "datos": {
    "tipo": "reporte",
    "nombre": "reporte_20260701_120000.txt",
    "hash_documento": "3126fee3c7520dcb57198c1e4120eb57bc07a7039570d527128c840baea445e7",
    "metadatos": { "tamano_bytes": 2048 }
  },
  "hash_anterior": "0000728e2e3edf8a770e3c762b6a37c5d6e834165a83ac5b784ba65f7f2566be",
  "nonce": 31528,
  "hash": "0000b1c5cc8de1cb65eec9536df4ac6d1ec691c5ecbcbdc0e9918fadd82ee04a"
}
```

Es un ejemplo ilustrativo: el nombre del archivo y los metadatos son de muestra. Los bloques reales (reporte, predicciones, dataset, modelo, métricas) se crean al ejecutar el pipeline y se guardan en `blockchain_data/cadena.json`.

## 4. Generación del hash

**Hash del documento.** SHA-256 sobre los bytes del contenido:

- Archivo: `sha256_archivo(ruta)`, lectura por bloques de 1 MiB (sirve para datasets grandes y modelos).
- Texto: UTF-8. Diccionario (p. ej. una predicción): `sha256_json(obj)`, que usa JSON **canónico**: `sort_keys=True`, `ensure_ascii=False`, separadores compactos `(",", ":")`. Así dos diccionarios iguales dan el mismo hash sin importar el orden de las claves.

**Hash del bloque.**

```
hash = SHA-256( JSON canónico de { indice, marca_tiempo, datos, hash_anterior, nonce } )
```

Es decir, todos los campos excepto `hash`. Como el hash incluye `hash_anterior`, cambiar cualquier bloque cambia su hash y rompe el enlace del siguiente. Esa es la propiedad de "encadenamiento".

**Prueba de trabajo (`Bloque.minar`).** Se fija `nonce = 0` y se incrementa hasta que `hash` comience con `config.DIFICULTAD_POW = 4` ceros ("0000..."). En promedio se requieren ~16^4 = 65,536 intentos (en la evidencia, el génesis requirió nonce = 21,293). Modificar un bloque obliga a volver a minarlo y también los siguientes.

## 5. Información registrada y justificación

| Tipo | Qué se registra | Por qué |
|---|---|---|
| `reporte` | Hash del archivo del reporte (`outputs/reportes/reporte_<AAAAMMDD_HHMMSS>.json`, que contiene el texto NLG y los resultados; se genera también un `.txt` legible) | Es el documento que sustenta decisiones; si alguien lo edita, se detecta |
| `prediccion` | Hash de cada predicción (entradas, valor, intervalo) cuando el usuario la registra desde el dashboard | Garantiza que la predicción usada en una decisión (p. ej. ajuste de contratación) no se modificó |
| `dataset` | Hash del dataset procesado | Permite probar con qué datos se entrenó |
| `modelo` | Hash del archivo `modelo_consumo.joblib` | Permite probar que el modelo desplegado es el mismo que se evaluó |
| `metricas` | Hash de `metricas_modelos.json` | Evita maquillar los resultados de evaluación |

**No se guarda el documento, solo su hash**, por tres razones:

1. **Privacidad:** el hash no permite reconstruir el documento, así que la cadena no expone información sensible.
2. **Tamaño:** 64 caracteres sin importar si el archivo pesa megabytes.
3. **Verificación suficiente:** para auditar se recalcula el hash del documento actual y se compara.

**¿Por qué Blockchain y no solo una base de datos con hashes?** Una base de datos puede editarse (documento y hash) sin dejar rastro por quien tenga acceso de escritura. En una cadena de bloques cada registro depende del anterior: alterar un registro pasado obliga a recalcular y re-minar todos los posteriores, y cualquier copia previa de la cadena evidencia la manipulación. Aporta **trazabilidad** (orden cronológico y encadenado), **integridad verificable** (cualquiera puede recalcular) y **no repudio** limitado (el registro tiene marca de tiempo). En este prototipo de un solo nodo la protección es parcial (ver limitaciones).

## 6. Procedimiento de validación

### 6.1 Verificar un documento (`verificar_archivo`, `verificar_contenido`)

1. Se calcula el SHA-256 del documento actual.
2. Se busca en la cadena un bloque con ese `hash_documento` (excluyendo el génesis).
3. Resultados (`{"integro", "hash_calculado", "hash_registrado", "bloque", "mensaje"}`):
   - **ÍNTEGRO:** el hash coincide con el registrado (e indica si corresponde a una versión anterior del documento).
   - **MODIFICADO:** no coincide con ningún bloque pero existe un bloque con el mismo nombre; se muestra el hash registrado.
   - **NO REGISTRADO:** no hay bloque con ese hash ni con ese nombre.

```
Reporte original -> Procesamiento -> SHA-256 -> Hash -> Registro en bloque
Posteriormente: Documento actual -> nuevo hash -> ¿coincide con el registrado?
                                       Sí -> Íntegro      No -> Modificado
```

### 6.2 Validar la cadena (`validar_cadena`)

Para cada bloque se recalcula y comprueba:

1. El **índice** es consecutivo.
2. El **hash guardado** coincide con el recalculado (detecta alteración de contenido).
3. El hash cumple la **prueba de trabajo** (4 ceros).
4. El **enlace**: `hash_anterior` igual al hash del bloque previo (el génesis, 64 ceros).

Devuelve `{"valida": bool, "errores": [mensajes], "n_bloques": int}`.

## 7. Evidencia del registro y prueba de alteración

Ejecución del 2026-11-14 20:58:56 (`python -m src.blockchain.demo_alteracion`, cadena temporal; la cadena real no se toca). **Resultado: 10/10 comprobaciones correctas; todas las alteraciones fueron detectadas.**

| Paso | Acción | Resultado |
|---|---|---|
| 1 | Crear cadena con génesis | Hash `0000b6a4...4b29` (nonce 21293); 1 bloque |
| 2 | Registrar un documento de ejemplo (`reporte_demo.txt`, SHA-256 `3126fee3...45e7`); también predicción (bloque 2) y métricas (bloque 3) | Bloque 1 hash `00005048...2654`, enlazado con el génesis |
| 3 | Verificar documento sin modificar | **ÍNTEGRO** (hash calculado = registrado) |
| 4 | Modificar **un solo carácter** (1250 a 1260) y verificar | Hash cambia a `c709ffb5...ab7`; **MODIFICADO** |
| 5 | Verificar en memoria una predicción (dict): original y alterada (52.3 a 52.4) | Original íntegro; alterada **MODIFICADA** |
| 6 | Validar la cadena sin alteraciones | Válida: 4 bloques, sin errores |
| 7 | Alterar `hash_documento` del bloque 1 en una **copia** de la cadena | Detectado: "Bloque 1: el hash guardado no coincide con su contenido ... el bloque fue alterado" |
| 8 | El atacante recalcula y re-mina el bloque 1 para ocultar el cambio | Detectado: "Bloque 2: enlace roto, hash_anterior no coincide con el hash del bloque 1" |
| 9 | Comprobar que la cadena de la demo sigue válida y que la real no se tocó | Correcto |

Lecciones de la prueba: (a) un carácter cambia por completo el hash (efecto avalancha); (b) alterar un bloque se detecta por el hash propio; (c) un atacante que re-mina el bloque alterado rompe el enlace con el siguiente, por lo que tendría que re-minar toda la cadena posterior.

Registro de la cadena real: al ejecutar `pipeline.py` (ver `06_arquitectura.md`) se registran bloques de tipo `reporte`, `dataset`, `modelo`, `metricas` y, desde el dashboard, `prediccion`. Estos bloques se pueden ver en la página Blockchain del dashboard (explorador de bloques) o con `Blockchain().a_lista()`.

Pruebas unitarias adicionales: `python -m unittest tests.test_blockchain` (dificultad 2 para rapidez).

## 8. Limitaciones

1. **Un solo nodo, no descentralizada:** no hay consenso ni réplicas. Quien tenga acceso de escritura al archivo podría reescribir **toda** la cadena (re-minando todos los bloques) sin que una copia aislada lo delate.
2. **PoW didáctica:** con un solo nodo y dificultad 4 (~65 mil intentos), reescribir la cadena es barato para un atacante; la PoW no equivale a seguridad real.
3. **Sin identidades ni firmas digitales:** no se firma cada bloque con una clave privada; no hay autenticación de quién registró.
4. **Marca de tiempo local:** proviene del reloj del sistema, no de una fuente confiable.
5. **Solo prueba integridad, no veracidad:** el hash demuestra que un documento no cambió desde que se registró, no que su contenido sea correcto.
6. **Escalabilidad:** el JSON se reescribe completo en cada registro; adecuado para un prototipo, no para volúmenes altos.
7. **El documento debe conservarse aparte:** como solo se guarda el hash, si se pierde el documento no puede recuperarse.
8. **Mejoras:** firmas digitales, varios nodos validadores con consenso, anclar periódicamente el hash raíz en una red pública o testnet (ver `07_conclusiones.md`).
