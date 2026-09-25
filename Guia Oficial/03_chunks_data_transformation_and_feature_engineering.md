---
tema: "Capítulo 3 — Transformación de datos e ingeniería de características (chunks de repaso)"
fuente: Guia Oficial/03_data_transformation_and_feature_engineering.md
guia-mla-c01: [Dominio 1, 1.2 Transform data and perform feature engineering, 1.3 Ensure data integrity and prepare data for modeling]
verificado: 2026-09-25 (cifras heredadas de la versión explicada)
tags: [aws, mla-c01, mla-c02, repaso, feature-engineering, data-lake, s3, lake-formation, glue, glue-databrew, athena, feature-store, data-wrangler, jumpstart, rekognition, comprehend, textract, bedrock, ground-truth, clarify]
---

> [!warning] Estado de servicios (verificado 2026-09-25)
> - SageMaker → **SageMaker AI**. **Data Wrangler** se usa hoy desde **SageMaker Canvas**; la versión de Studio Classic sigue documentada.
> - **Ground Truth** y **Clarify**: no aceptan clientes nuevos. Los clientes existentes pueden seguir usándolos, pero no recibirán funciones nuevas.
> - **Mechanical Turk**: cierra el **2026-09-30**; ese día deja de estar disponible como fuerza de trabajo de Ground Truth.
> - **Nova Canvas** (Bedrock): sin clientes nuevos desde 2026-03-30; deja de funcionar el 2026-09-30.
> - **SDK de SageMaker v3**: `sagemaker.clarify` → `sagemaker.core.clarify`; `sagemaker.feature_store` → recurso `FeatureGroup` en `sagemaker.mlops.feature_store`. Para ejecutar el código del libro: `pip install "sagemaker>=2,<3"`.
> - **MLA-C02** (beta desde 2026-09-29): Mechanical Turk sale del temario, y las habilidades evaluadas ya no nombran Ground Truth ni Clarify, aunque el etiquetado y el sesgo siguen dentro. Esas partes van marcadas **[C01]**. El C02 pide más IA generativa de la que cubre este capítulo.

---

## 1 · Data lake en AWS

**Qué recordar**
- Data lake: repositorio central de datos crudos en su formato original, con **schema-on-read**. Data warehouse: el esquema se define antes de cargar (**schema-on-write**).
- Pila típica: **S3** (almacenamiento) + **Glue** (Data Catalog y crawlers) + **Athena** (SQL) + **Lake Formation** (IAM centralizado, seguridad y gobernanza).
- Características clave:
  - Almacenamiento a escala: payloads pequeños y lotes grandes, a intervalos o en tiempo real.
  - Movimiento de datos: entrada y salida.
  - Datos crudos: el formato no importa en la ingesta ni en el almacenamiento.
  - Seguridad.
  - Catálogo: datos relacionales y no relacionales, con rastreo, catalogación e indexación.

**Reglas/condiciones**
- Seguridad = IAM + cifrado:
  - En **reposo**: S3 por bucket, con claves de S3 o de KMS.
  - En **tránsito**: TLS; una política de bucket puede rechazar las peticiones que no lleguen por HTTPS.
  - En **uso**: Nitro Enclaves.
  - Según el libro, el cifrado en uso y en tránsito importa tanto como en reposo.
- Guardrails (límites que aplican a todos, tengan los permisos que tengan): S3 Block Public Access, SCP de Organizations, reglas de AWS Config (estas detectan y reportan; no bloquean).
- Crawler de Glue → infiere esquema, tipos y particiones (`anio=2025/mes=01/`) → registra la tabla en el Data Catalog.
- Archivos con esquemas distintos en la misma carpeta → el crawler crea varias tablas o particiones incompatibles → Athena devuelve errores o columnas vacías. Por eso los esquemas se homogeneizan antes que nada.
- Athena es serverless y cobra por datos escaneados → guardar en Parquet (columnar y comprimido) abarata las consultas.

**Diferencias o trampas**
- No borres la zona cruda: permite reprocesar sin volver a pedir los datos a los sistemas de origen.

---

## 2 · Amazon S3 como almacenamiento del data lake

**Qué recordar**
- Opción preferida para el data lake: durabilidad de 11 nueves, alta disponibilidad, seguridad e integración con los servicios de datos y de ML.
- **Fuente única de verdad**: Athena, Glue, el entrenamiento de SageMaker y el almacén offline de Feature Store leen la misma copia.

**Reglas/condiciones**
- Durabilidad (no perder datos) ≠ disponibilidad (poder acceder a ellos). Los 11 nueves son un **objetivo de diseño**: el **SLA cubre la disponibilidad**, no la durabilidad. Escala: con 10 M de objetos, en promedio se perdería 1 cada 10 000 años.
- Replica en varias AZ. Las clases de una sola zona (One Zone-IA) no resisten la pérdida de esa AZ.
- Clases:
  - Standard: acceso frecuente, sin cargo por lectura.
  - Standard-IA: más barata por GB, pero **cobra cada GB recuperado**.
  - Glacier: archivo.
- **Intelligent-Tiering**: 30 días sin acceso → nivel infrecuente; 90 días → archivo con acceso instantáneo. Al leerse, el objeto vuelve al nivel frecuente. Cobra una tarifa de monitoreo por objeto, y los objetos de **menos de 128 KB** no se monitorean ni se mueven.
- Reglas de ciclo de vida → mueven objetos a Glacier por antigüedad.
- **Object Lock en modo compliance** → nadie, ni el usuario raíz, puede borrar ni modificar un objeto antes de que venza su retención.
- **Versioning** → conserva las versiones anteriores. Cada versión se cobra, así que se combina con una regla de ciclo de vida que borre las antiguas.

**Diferencias o trampas**
- Millones de archivos diminutos → Intelligent-Tiering no ahorra nada.
- Separar datos por bucket → solo cuando cambian los permisos o el cifrado (p. ej., un cliente con su propia clave de KMS). Para organizar basta con prefijos.

---

## 3 · AWS Lake Formation

**Qué recordar**
- Capa de gobernanza sobre el Glue Data Catalog. Centraliza los permisos por base de datos, tabla, columna, fila y celda, con concesiones tipo `GRANT`. También monitorea el acceso y ayuda con el compliance.

**Reglas/condiciones**
- **Filtros de datos**:
  - Fila: una expresión, como `hospital_id = 'H3'`.
  - Columna: incluir o excluir columnas.
  - Celda: fila y columna a la vez.
  - Se asignan al conceder `SELECT` sobre una tabla.
- Los filtros se aplican **al leer** sobre la misma tabla → una sola copia de los datos.
- Los usuarios no tienen permisos directos sobre el bucket. Consultan con motores integrados (**Athena, EMR, Redshift Spectrum**), a los que Lake Formation da **credenciales temporales** limitadas a lo autorizado. Cada acceso queda en **CloudTrail**.
- Permite compartir tablas, o las partes que definen los filtros, con **otras cuentas** de AWS.

**Diferencias o trampas**

| Alternativa | Por qué no sirve para filtrar filas o columnas del lago |
|---|---|
| Políticas de IAM o de bucket | Llegan a bucket, prefijo u objeto; un archivo Parquet trae todas sus filas y columnas |
| S3 Access Points / Access Grants | La misma granularidad: objeto o prefijo |
| DataBrew o Glue job con versiones enmascaradas | Generan copias |
| Macie | Descubre y clasifica datos sensibles; no controla el acceso |
| Redshift (seguridad por fila y columna) | Hay que cargar los datos, lo que crea otra copia, y el lago en S3 queda fuera |

---

## 4 · Tipos de datos y áreas de la ingeniería de características

**Qué recordar**
- Tipos de datos:
  - **Categóricos**: ordinales (con orden, como las tallas) o nominales (sin orden, como los estados).
  - **Numéricos**: discretos (contables) o continuos (incluyen fecha y hora).
  - **Texto**, **imagen** y **series temporales** (en estas, el orden cronológico es crucial).
  - El tipo de dato determina la técnica.
- La ingeniería de características saca más información de los datos existentes. **No agrega datos nuevos** y se apoya en el conocimiento del dominio.

| Área | ¿Reduce dimensionalidad? | Qué hace |
|---|---|---|
| Extracción | Sí | Crea automáticamente características nuevas a partir de las existentes. Típica en imagen, audio y texto (p. ej., `windshield_present`, o las palabras más frecuentes sin artículos) |
| Selección | Sí | Ordena por importancia predictiva y conserva las más relevantes. Elimina las irrelevantes y las redundantes (correlacionadas, como ingresos y ventas); los filtros estadísticos miden la relación con la variable objetivo |
| Creación/transformación | **No** | Genera características nuevas (fecha → día, mes y año; día de la semana) |

**Diferencias o trampas**
- La extracción y la selección suelen usarse juntas. La creación/transformación no reduce la dimensionalidad.
- Cada dimensión extra dificulta el entrenamiento. También cuesta en infraestructura: más espacio en S3, más escaneo en Athena, más tiempo de lectura y más memoria.

---

## 5 · Amazon SageMaker Feature Store

**Qué recordar**
- Repositorio fully managed para almacenar, compartir y gestionar características. Mantiene sincronizados el historial para entrenar y el valor actual para inferir en tiempo real.
- **Feature group**: definición de columnas (entero, fraccionario, cadena) + **record identifier** (p. ej., `customer_id`) + **event time**.

| | Online store | Offline store |
|---|---|---|
| Contenido | Solo el **último** registro de cada ID | **Todos** los registros históricos (agrega, nunca sobrescribe) |
| Acceso | `GetRecord`, milisegundos | Athena, segundos |
| Almacenamiento | `Standard`, `Standard_V2` o `InMemory` (ElastiCache) | **Tu** bucket de S3, en Parquet, registrado en el Glue Data Catalog (formato Glue o **Iceberg**; se recomienda Iceberg porque compacta los archivos pequeños) |
| Uso | Inferencia en tiempo real | Entrenamiento, inferencia por lotes, consultas point-in-time |

**Reglas/condiciones**
- **Ingesta única** (`PutRecord`, o exportación desde Data Wrangler) → con los dos almacenes activos, Feature Store los mantiene sincronizados.
- `Standard` es el tipo predeterminado. `Standard_V2` actualiza una sola característica sin reescribir el registro y rechaza escrituras fuera de orden según el event time.
- `InMemory` → **no admite almacén offline**, no admite claves de KMS del cliente y tiene un máximo predeterminado de **50 GiB**. Si necesitas historial y tiempo real sincronizados → `Standard`/`Standard_V2` con offline.
- **Point-in-time**: arma el dataset con el valor vigente en la fecha de cada ejemplo, no con el valor actual.
- Los grupos se descubren en Studio por nombre, descripción y etiquetas. Acceso con IAM por grupo, cifrado en reposo con KMS y acceso privado desde la VPC por **PrivateLink**.

**Diferencias o trampas**
- DynamoDB: latencia suficiente, pero solo guarda el valor actual; el historial habría que construirlo aparte. ElastiCache: no guarda historia. S3 + Athena: tiene la historia, pero tarda segundos. Redshift: es analítico, no sirve miles de lecturas de una fila por segundo.
- Data Wrangler y DataBrew **calculan** características; no las almacenan ni las sirven.
- Guardar los vectores ya extraídos (p. ej., de imágenes procesadas con GPU) evita extraerlos otra vez para cada modelo.
- Resumen del libro: Data Wrangler → ingeniería de características; Feature Store → almacenar, compartir y gestionar las características resultantes.

---

## 6 · Valores faltantes

**Reglas/condiciones**
- Muchos faltantes en **varias** características → **recolectar** de nuevo. Sopesa el costo de la reingesta frente al de un modelo pobre y frente a imputar o eliminar. Si el dato crudo sigue en el lago, basta con reprocesarlo.
- Pocos faltantes y **aleatorios** (probable falla de ingesta) → **imputar**: con la media si la distribución es normal; si no, con la mediana o el valor más frecuente.
- Muchos faltantes concentrados en **una** característica → **eliminar esa característica**. Riesgo: quedarse sin datos suficientes para el modelo.
- La mayoría de los algoritmos no manejan los faltantes por sí solos.

**Diferencias o trampas**
- `SimpleImputer(strategy='mean')` es de **scikit-learn**, no de AWS. «Con SageMaker» significa que corre en un notebook de SageMaker (JupyterLab en Studio o una notebook instance), que se cobra por hora mientras está encendido.
- pandas y sklearn cargan todo en la RAM de la instancia. Si no cabe → instancia más grande o Glue (Spark).
- Pasos de DataBrew: `FILL_WITH_AVERAGE`, `FILL_WITH_MEDIAN`, `FILL_WITH_MODE`, `FILL_WITH_CUSTOM`, `FILL_WITH_LAST_VALID`, `REMOVE_MISSING`.

---

## 7 · AWS Glue DataBrew

**Qué recordar**
- Preparación de datos **visual y sin código**, de la familia Glue, **serverless**. Cobra por sesión interactiva y por el tiempo de cómputo de los jobs.
- Flujo:
  1. **Proyecto**: se conecta a S3, al Glue Data Catalog, a Redshift o a bases de datos por red. Carga una **muestra** en una cuadrícula con la distribución de cada columna.
  2. **Receta**: los pasos elegidos quedan guardados y se reutilizan con otros datasets (hay más de 250 transformaciones).
  3. **Job**: aplica la receta al dataset completo, se puede programar y escribe en S3.

| Tarea | Pasos de receta |
|---|---|
| Faltantes | `FILL_WITH_AVERAGE`/`MEDIAN`/`MODE`/`CUSTOM`/`LAST_VALID`, `REMOVE_MISSING` |
| Atípicos | `REMOVE_OUTLIERS`, `REPLACE_OUTLIERS`, `FLAG_OUTLIERS`, `RESCALE_OUTLIERS_WITH_Z_SCORE`, `RESCALE_OUTLIERS_WITH_SKEW`. Detección: `Z_SCORE`, `MODIFIED_Z_SCORE` (basado en la mediana), `IQR` |
| Duplicados | `DELETE_DUPLICATE_ROWS`, `REMOVE_DUPLICATES`, `FLAG_DUPLICATE_ROWS` (solo marca) |
| Escalado | `SCALE` con `Z_SCORE`, `MIN_MAX`, `MEAN_NORMALIZATION` |
| Sesgo | `SKEWNESS` con `LOG`, `ROOT`, `SQUARE` |
| Binning | `BUCKETIZATION` |
| Categóricas | `CATEGORICAL_MAPPING` (label/ordinal), `ONE_HOT_ENCODING`; no tiene paso para alta cardinalidad |
| Texto | `TOKENIZATION`: tokeniza, quita stop words (lista predeterminada o propia), hace stemming (`PORTER`/`LANCASTER`) y expande contracciones. **No lematiza** |

**Diferencias o trampas**
- **DataBrew o Data Wrangler**: DataBrew también está pensado para analistas que no trabajan en SageMaker, y cobra por sesión y por job. Data Wrangler vive en SageMaker (Canvas), exige un dominio de SageMaker y se integra con Feature Store y Pipelines.
- «Eliminar duplicados de la forma más simple y sin código» → DataBrew.

---

## 8 · Amazon SageMaker Data Wrangler

**Qué recordar**
- Preparación de datos **low-code** dentro de SageMaker, hoy desde **Canvas**: importar, explorar (EDA), transformar y analizar.
- **Data flow**: los pasos se diseñan sobre una **muestra** y después se aplican al dataset completo. Salidas: **S3**, **Feature Store**, un paso de **Pipelines**, un **script de Python** o un **serial inference pipeline**.
- Corre sobre **Apache Spark** → procesa datasets que no caben en la memoria de una sola máquina.

| Grupo | Contenido |
|---|---|
| Handle outliers | Detección: desviación estándar, desviación estándar robusta, cuantiles, umbrales mín./máx. Corrección: recortar al límite, eliminar la fila, marcar como inválido |
| Process numeric | Standard, Robust, Min Max y Max Absolute Scaler |
| Encode categorical | Ordinal encode, One-hot encode, Similarity encode (alta cardinalidad) |
| Time series | Agrupar por serie, remuestrear, rellenar huecos, validar timestamps, igualar longitudes, featurize datetime, lags, rangos de fechas, rolling window, extract features |
| Featurize text | Vectorize, con tokenizador configurable; sin lematización integrada |
| Balance data | Random oversampling, random undersampling, SMOTE |
| Split data | Random, ordered, stratified, split by key |
| Reduce dimensionality | PCA |
| Imágenes | Importa desde un prefijo de S3. Con OpenCV e imgaug: redimensionar, brillo, escala de grises, rotar, quitar imágenes corruptas o duplicadas |
| Custom transform | pandas, PySpark o fórmula (logaritmos, raíces, binning, lematización, deduplicación) |

**Reglas/condiciones**
- Los estadísticos de *Handle outliers* (media, σ, cuantiles) y la lista de categorías de los encoders se fijan **al definir el paso**, con la muestra cargada, y se reutilizan en cada job. Una muestra poco representativa da límites erróneos.
- Una categoría que no estaba al definir el paso → se trata según la **estrategia de inválidos**: omitir la fila, conservarla como categoría extra, dar error o reemplazarla por NaN.

**Diferencias o trampas**
- Servicios para atípicos que pide el examen: Data Wrangler y DataBrew.

---

## 9 · Valores atípicos: detección y tratamiento

**Qué recordar**
- Atípico: punto que se desvía significativamente de la media. Regla general: a más de **3σ**. En una normal, ±1σ cubre ≈ 68 %, ±2σ ≈ 95 % y ±3σ ≈ 99.7 %.
- **Ruido** = un grupo de puntos erróneos. **Atípico** = un punto muy alejado de la media. Un atípico puede ser **natural** (refleja algo real) o **artificial** (un error).
- El método de detección se elige mirando el **histograma**:

| Distribución | Método | Regla |
|---|---|---|
| Sesgada | **IQR** (robusto al sesgo) | Atípico si `< Q1 − 1.5·IQR` o `> Q3 + 1.5·IQR`, con `IQR = Q3 − Q1` |
| Normal | **Z-score** | `z = (x − μ)/σ`; atípico si `abs(z) > 3` |

**Reglas/condiciones (tratamiento)**
- Ruido o error artificial → **eliminar** (no afecta la calidad del modelo) o **imputar** con la media o la mediana.
- Atípico que sesga la distribución → **transformación logarítmica**: comprime el rango, acerca el atípico al resto sin eliminarlo y vuelve la distribución más simétrica (log₁₀ 1000 = 3; log₁₀ 23 ≈ 1.36).
- Entrenar con datos ruidosos → el modelo aprende peculiaridades → **overfitting**.

**Diferencias o trampas**
- En el ejemplo del libro (10 puntos sesgados a la derecha), IQR detecta [25, 23] y Z-score no. El libro lo atribuye al sesgo, pero hay otra razón: con `ddof=0`, el `abs(z)` máximo posible es `(n−1)/√n`, que para n = 10 vale 2.85. Ningún punto podía superar 3.
- `DataFrame.quantile` es de **pandas**; NumPy no tiene `DataFrame` (su función es `np.quantile`).

---

## 10 · Deduplicación

**Qué recordar**
- Los duplicados distorsionan los resultados, favorecen el **overfitting** y meten **sesgo**: las entradas repetidas desbalancean los datos.
- Orígenes típicos:
  - Entrega **at-least-once**: ante un fallo de red, el servicio reenvía en vez de perder el mensaje.
  - Jobs de carga relanzados completos después de fallar a la mitad.
  - Dos sistemas de origen que exportan la misma entidad.

**Reglas/condiciones**
- Lo más simple, sin código → **DataBrew**: `DELETE_DUPLICATE_ROWS`, `REMOVE_DUPLICATES`, o `FLAG_DUPLICATE_ROWS` para revisar antes de borrar.
- Alternativas: un snippet en la custom transform de Data Wrangler, o un Glue job con Spark (`dropDuplicates`).

---

## 11 · Estandarización y reformateo a escala: notebook de SageMaker + Glue

**Qué recordar**
- Estandarizar evita que las características con valores grandes dominen. Importa en los algoritmos sensibles a la escala, como la regresión lineal y las SVM.
- Reformatear = tipos, formatos y unidades uniformes (p. ej., fechas y unidades de medida). En AWS, lo típico es pasar CSV o JSON a **Parquet**.
- Pasos del libro: cargar en un notebook → `StandardScaler` → exportar a S3 → **Glue job** que reformatea → entrenar.

**Reglas/condiciones**
- `pd.read_csv('s3://bucket/ruta.csv')` necesita **`s3fs`** y un **rol de ejecución** con `s3:GetObject`. Sin ese permiso falla con **`AccessDenied`**, que no es un error de pandas.
- Escribir en S3 exige `s3:PutObject` (con `to_csv('s3://…')` o con boto3). La ruta sin `s3://` del libro (`'your_dataset.csv'`) lee el disco local de la instancia.
- **Glue job**: script (normalmente PySpark) en infraestructura serverless. La capacidad se mide en **DPU** (1 DPU = 4 vCPU + 16 GB) y se cobra por DPU-hora. Glue levanta las máquinas y las apaga al terminar.
- Reparto habitual: el notebook sirve para explorar y probar sobre una muestra; el Glue job repite esa lógica sobre todo el volumen (p. ej., cada noche) y necesita **su propio rol de IAM** con permisos de S3.

**Diferencias o trampas**
- Un dataset más grande que la RAM (200 GB frente a 16 GB) → Glue (Spark), no pandas.

---

## 12 · Replicar las transformaciones en producción

**Qué recordar**
- El endpoint recibe datos crudos, así que hay que aplicarles **las mismas transformaciones, con los mismos parámetros**, que en el entrenamiento. Si la transformación solo existe en un notebook, no se puede replicar.

**Reglas/condiciones**
- Tres formas:
  1. Exportar el flujo de Data Wrangler como **serial inference pipeline**: un endpoint que encadena el contenedor de preprocesamiento y el del modelo.
  2. Leer las características ya calculadas desde **Feature Store**, al entrenar y al inferir.
  3. Empaquetar la transformación en el **contenedor del modelo**.
- Los encoders con vocabulario (one-hot, ordinal) fijan las categorías, y una categoría nueva rompe el pipeline o se trata como inválida. El **hashing** no guarda estado: el mismo código funciona en el notebook, en Spark y en el endpoint.

---

## 13 · Normalización vs estandarización

| | Normalización | Estandarización |
|---|---|---|
| Resultado | Rango fijo, normalmente **[0, 1]** | Media **0** y desviación estándar **1** |
| Implementación | **MinMax** (`MinMaxScaler`) | **Z-score**, `z = (x − μ)/σ` (`StandardScaler`) |
| Úsala cuando | Importa la **escala** y no hace falta centrar en 0 | Importa la **distribución** (datos aproximadamente normales) |
| Algoritmos | Basados en distancias (**k-NN**, **redes neuronales**); acelera el descenso de gradiente | Los que suponen normalidad: **regresión lineal**, **logística**, algoritmos con descenso de gradiente |
| ¿Corrige el sesgo? | **No** | **No** |

**Reglas/condiciones**
- Beneficios de normalizar: ponderación equitativa, convergencia más rápida (menos segundos facturados de entrenamiento) y coeficientes comparables entre sí.
- MinMax conserva las relaciones entre los valores, pero no toca el sesgo. Con Z-score, unos datos sesgados siguen igual de sesgados.
- «Escalado» es el término general que abarca a los dos.
- Normaliza o estandariza **después** de dividir en train/test, para evitar la fuga de datos.

**Detalles examinables**
- En AWS: *Process numeric* de Data Wrangler (Standard, Min Max Scaler) y `SCALE` de DataBrew (`Z_SCORE`, `MIN_MAX`, `MEAN_NORMALIZATION`).

---

## 14 · Escaladores para datos no normales: robusto, MaxAbs y de potencia

| Técnica | Cómo | Cuándo | Clase de sklearn |
|---|---|---|---|
| Robust scaling | Resta la **mediana** y divide entre el **IQR** | Atípicos y datos sesgados | `RobustScaler` |
| MaxAbs | Divide entre el valor absoluto máximo → **[−1, 1]** | Datos **dispersos** (muchos ceros): los ceros se quedan en 0 | `MaxAbsScaler` |
| Box-Cox | Transformación de potencia | Estabiliza la varianza y acerca los datos a la normal; **solo valores positivos** | `PowerTransformer` |
| Yeo-Johnson | Transformación de potencia | Lo mismo, con **positivos y negativos** | `PowerTransformer(method='yeo-johnson')` |

**Diferencias o trampas**
- El formato disperso guarda solo los valores distintos de cero. Restar la media (Z-score) convierte los ceros en otros números, y el dataset en memoria puede multiplicar su tamaño. MaxAbs deja los ceros intactos.
- Box-Cox y Yeo-Johnson **sí** reducen el sesgo. El escalado robusto es menos sensible a los atípicos.
- En AWS: *Process numeric* de Data Wrangler (Robust, Max Absolute Scaler).

---

## 15 · Transformaciones contra el sesgo: logaritmo y raíces

**Qué recordar**
- **Logaritmo**:
  - Reduce el sesgo a la derecha (colas largas: ingresos, notas de un examen difícil).
  - Limita el peso de los atípicos sin eliminarlos.
  - **Linealiza** relaciones multiplicativas o exponenciales: `y = 1, 10, 100, 1000` → `log₁₀ y = 0, 1, 2, 3`.
- **Raíz cuadrada o cúbica**: bajan una varianza alta, con menos efecto que el logaritmo.

**Reglas/condiciones**

| Transformación | Dominio |
|---|---|
| Logaritmo (cualquier base) | Solo valores > 0. Con ceros o negativos: desplaza los datos o usa raíz cúbica |
| Raíz cuadrada | Valores ≥ 0 |
| Raíz cúbica | Cualquier valor, incluidos los negativos y el 0 |

**Detalles examinables**
- **Corrigen** el sesgo: logaritmo, raíz cuadrada, Box-Cox, Yeo-Johnson. **No lo corrigen**: Z-score, MinMax.
- En AWS: `SKEWNESS` de DataBrew (`LOG`, `ROOT`, `SQUARE`); en Data Wrangler, una custom transform.

---

## 16 · Binning (bucketing)

- Divide un valor continuo en intervalos discretos: el numérico pasa a categórico. Hace el modelo más robusto frente a atípicos y ruido, y lo simplifica.
- Ejemplo: el peso de un vehículo → `is_minicompact`, `is_subcompact`, `is_compact`, `is_midsize`, `is_large`.
- En AWS: `BUCKETIZATION` de DataBrew; en Data Wrangler, custom transform o fórmula.

---

## 17 · Codificación de variables categóricas

| Técnica | Cómo | Úsala para | En contra |
|---|---|---|---|
| **Label encoding** | Un entero por categoría (`White, Black, Red` → `0, 1, 2`) | Datos **ordinales** y modelos **basados en árboles** | Impone un orden; problema para algoritmos que suponen relaciones ordenadas |
| **One-hot** | Una columna binaria (0/1) por categoría | **Pocas** categorías **nominales** con modelos **que no son árboles** (lineales, k-NN, redes) | La dimensionalidad crece con la cardinalidad |
| **Binary encoding** | Categoría → entero → cada bit en una columna | Frenar el crecimiento de dimensiones de one-hot | Pierde información: las diferencias entre categorías se difuminan |
| **Feature hashing** | Función hash (+ módulo) → índice en un vector de tamaño fijo `n_features` | **Alta cardinalidad**, de forma eficiente y económica | **Colisiones** |

**Reglas/condiciones**
- One-hot con alta cardinalidad: ~40 000 códigos postales × 10 M de filas × 8 bytes, en formato denso ≈ **3 TB**, frente a ~50 MB de la columna original. Salidas: formato disperso, binary encoding o **PCA** después del one-hot.
- Binary: `category_encoders.BinaryEncoder` numera desde **1**, así que 8 categorías dan **4** columnas y no 3 (Brown = 8 = `1000`). El número de filas no cambia.
- Hashing:
  - No guarda vocabulario, y el número de columnas lo fijas tú. Las categorías nuevas no cambian el esquema.
  - El valor hash **no es único**: en el ejemplo del libro, White y Blue colisionan.
  - `FeatureHasher` asigna un signo ±1 a cada valor.

**Detalles examinables**
- En AWS: *Encode categorical* de Data Wrangler (Ordinal, One-hot, **Similarity encode** para alta cardinalidad); DataBrew: `CATEGORICAL_MAPPING` y `ONE_HOT_ENCODING`.
- Los encoders de Data Wrangler fijan las categorías al definir el paso. Las que no vio se tratan según la estrategia de inválidos: omitir, categoría extra, error o NaN.

---

## 18 · Series temporales en Data Wrangler

**Qué recordar**
- Cada punto depende de sus valores pasados, así que el orden importa. Data Wrangler agrupa estas transformaciones en *Time series* (low-code, respetando el formato de entrada del modelo de pronóstico) y ofrece visualizaciones para ver los patrones antes de crear características.

**Reglas/condiciones**
- **Featurize datetime** (el primer paso recomendado) → `date_month`, `date_day`, `date_week_of_year`, `date_day_of_year`, `date_quarter`.
  - Acepta texto o un **timestamp Unix** (s, ms, µs o ns).
  - Formato: inferido o dado con códigos `strftime`. Darlo es lo más rápido. No darlo ni inferirlo es lo más robusto, pero hasta ~10× más lento.
  - Salida: un solo vector o una columna por componente.
  - **Embedding mode**: `cyclic` → modelos lineales y redes profundas; `ordinal` → árboles.
- **Encode categorical** sobre los componentes: `date_quarter` → one-hot de 4 columnas. El libro numera los trimestres de 0 a 3; no está confirmado (pandas usa 1–4).
- **Lag features**: valores pasados de la **variable objetivo**; capturan la **autocorrelación**.
  - Parámetros: columna, timestamp y duración del rezago.
  - Opciones: incluir toda la ventana, aplanar en columnas o descartar las filas sin historia suficiente.
- **Extract features**: *Minimal subset* (8 características, la más rápida), *Efficient subset* (solo las que no son costosas), *All features* y *Manual subset*.
- **Rolling window features**: las mismas estrategias, calculadas sobre una ventana. Con ventana 3, la fila *t* recibe características de *t−3*, *t−2* y *t−1*.

**Diferencias o trampas**
- *All features* sobre millones de filas multiplica el tiempo del job, y con él el costo.
- El libro atribuye la extracción a `tsfresh`; la documentación vigente no lo menciona.
- Dividir una serie temporal → **ordered split**, no aleatorio.

---

## 19 · Características de imágenes: JumpStart vs Rekognition

| | SageMaker JumpStart | Amazon Rekognition |
|---|---|---|
| Qué es | Catálogo de modelos preentrenados (de AWS y de terceros) que despliegas o ajustas | API de visión ya entrenada |
| Qué devuelve | El vector de características completo (p. ej., de una CNN) | JSON con etiquetas, confianza, `BoundingBox`, texto y rostros |
| Infraestructura | Tú operas el endpoint o el batch transform | Ninguna; se paga por imagen |
| Control | Máximo | Solo las etiquetas y atributos que definió AWS |

**Reglas/condiciones**
- JumpStart en un **endpoint en tiempo real** → las instancias (con GPU si hace falta) se cobran por hora **mientras el endpoint exista**, aunque no reciba tráfico. Bórralo al terminar.
- JumpStart en **batch transform** → lee un prefijo de S3, escribe los vectores en otro y apaga las instancias. Es lo **más barato** para procesar una sola vez un archivo histórico.
- Permisos: el rol de ejecución de SageMaker necesita leer el bucket de entrada y escribir en el de salida.
- `DetectLabels` de Rekognition devuelve etiqueta, confianza y `BoundingBox` (`Left`, `Top`, `Width`, `Height`). Son **fracciones** del tamaño de la imagen, así que no dependen de la resolución. La imagen va en la petición o como referencia a S3.
- Después de extraer: **Data Wrangler** para normalizar píxeles, reducir con **PCA** (*Reduce dimensionality*) y transformar imágenes; **Feature Store** para guardar los vectores.
- Patrón de datos no estructurados: un objeto de S3 por archivo, más una tabla aparte con la ruta y los metadatos. Los servicios reciben rutas de S3 y devuelven JSON.

**Detalles examinables**
- «Rediseñar las características de un dataset de imágenes» → **JumpStart**. Análisis básico sin esfuerzo → Rekognition.

---

## 20 · Características de texto: Comprehend, Textract y algoritmos integrados

| Servicio | Qué hace | Modos | Cobro |
|---|---|---|---|
| **Comprehend** | NLP preentrenado: `DetectEntities`, `DetectKeyPhrases`, `DetectSentiment`, `DetectDominantLanguage` | Síncrono (un documento) o asíncrono (job que lee un prefijo de S3 y escribe en otro) | Por volumen de texto |
| **Textract** | OCR que además reconoce estructura: formularios (clave-valor), tablas y *queries*, con confianza y posición | Síncrono (imagen de una página) o asíncrono (PDF de varias páginas en S3; avisa al terminar) | Por página |

**Reglas/condiciones**
- Orden habitual: **Textract → Comprehend**. El documento escaneado pasa a texto, y del texto salen entidades y sentimiento.
- Archivo histórico de reseñas → Comprehend en modo asíncrono.
- Extracción personalizada → algoritmos integrados: **BlazingText** (Word2Vec) y **LDA** (modelado de temas). AWS ya los empaquetó en un contenedor: tú defines el training job (algoritmo, ruta de S3, tipo y número de instancias, rol de IAM) y el modelo queda en S3.

**Diferencias o trampas**
- Comprehend y Rekognition **producen predicciones**; no sirven para que personas etiqueten datos.
- Tokenización, stemming y lematización **no son funciones de SageMaker**: se hacen con bibliotecas (NLTK, spaCy) en un notebook o un processing job. Sin código: `TOKENIZATION` de DataBrew.

---

## 21 · Preprocesamiento de texto

| Técnica | Qué hace | Ejemplo |
|---|---|---|
| Tokenización | Divide el texto en tokens (palabras o frases); es el primer paso | «Machine learning is fascinating» → `["Machine", "learning", "is", "fascinating"]` |
| Stop words | Quita palabras comunes que no aportan información | «The cat is on the mat» → `["cat", "on", "mat"]` |
| Stemming | Recorta prefijos y sufijos | running → run |
| Lematización | Lleva a la forma base con reglas lingüísticas | running → run |
| N-gramas | Secuencias contiguas de *n* elementos; capturan el contexto | Bigramas de «Machine learning is fun»: `("Machine", "learning")`, `("learning", "is")`, `("is", "fun")` |
| Word embeddings | Convierten palabras en vectores continuos con significado semántico (Word2Vec, GloVe) | King y Queen quedan cerca |

**Detalles examinables**
- `TOKENIZATION` de DataBrew: tokeniza, quita stop words, hace stemming (`PORTER`, `LANCASTER`) y expande contracciones («don't» → «do not»). **No lematiza.**
- *Featurize text → Vectorize* de Data Wrangler: tokenizador configurable. **Tampoco lematiza**; para eso hace falta una custom transform.
- «Palabras → vectores con significado semántico» → **word embeddings**, no one-hot ni tokenización.

---

## 22 · Amazon Bedrock: tokens y cobro

**Qué recordar**
- Bedrock da acceso fully managed por API a **modelos fundacionales** (FM) de varios proveedores, sin servidores. La petición va firmada con credenciales de IAM a un endpoint regional. El prompt es el texto de entrada, y el modelo lo tokeniza para interpretarlo.

**Reglas/condiciones**
- Modelos de **texto** bajo demanda → se cobran los **tokens de entrada + los de salida**. Un prompt más largo cuesta más.
- La **salida** suele costar más por token. Limitar el máximo de tokens de la respuesta es la forma más directa de acotar el costo.
- Cada modelo tiene su **propio tokenizador**: el mismo prompt da un número de tokens distinto en cada modelo. `CountTokens` (sin costo) calcula los tokens antes de enviar la petición, y cada respuesta informa los tokens facturados.
- Orden de magnitud: 1 token ≈ ¾ de palabra en inglés; en español, algo más de 1 token por palabra.
- Modelos de **imagen** → se cobran por **imagen generada** (según resolución y calidad), no por token.
- Otras modalidades: inferencia por lotes y **provisioned throughput** (capacidad reservada que se cobra por hora).

---

## 23 · Amazon SageMaker Ground Truth [C01]

> El C02 ya no nombra Ground Truth, pero el etiquetado sigue en el temario. El servicio no acepta clientes nuevos.

**Qué recordar**
- Etiquetar = añadir la verdad de referencia (ground truth) que necesita el aprendizaje supervisado. Ground Truth combina automatización con **human-in-the-loop**.
- **Manifiesto de entrada**: JSON Lines en S3 con la lista de objetos que se van a etiquetar. **Manifiesto de salida**: JSON Lines en S3, con la ubicación y la etiqueta de cada objeto.
- El **labeling job** reúne: ubicación en S3, tipo de tarea, plantilla de la interfaz, fuerza de trabajo, precio por tarea (si aplica) y rol de IAM.

| Fuerza de trabajo | Cuándo |
|---|---|
| **Mechanical Turk** (crowdsourcing público) | Solo si declaras que los datos no tienen **PII**. Cierra el 2026-09-30 |
| **Privada** (empleados o contratistas, en un portal con usuarios que administras) | Datos confidenciales que no pueden salir de la organización |
| **Proveedores** (contratados en **AWS Marketplace**) | Etiquetado especializado |

**Reglas/condiciones: etiquetado automatizado (active learning)**
1. Una muestra aleatoria va a personas.
2. Con esas etiquetas se lanza un **training job**, y después un **batch transform** puntúa el resto de los datos.
3. Se etiquetan automáticamente solo los objetos cuya confianza supera un umbral que calcula Ground Truth; los de baja confianza vuelven a las personas.
4. El ciclo se repite hasta etiquetar todo o agotar el presupuesto de etiquetado humano.
- Costo aparte: se factura como entrenamiento e inferencia de SageMaker (p. ej., `ml.p3.2xlarge`), en instancias que no aparecen en tu consola de EC2.
- Solo 4 tareas integradas: clasificación de imágenes y de texto (una etiqueta), detección de objetos (bounding boxes) y segmentación semántica. Exige un mínimo de **1250** objetos; se recomiendan **5000**.

**Diferencias o trampas**
- El libro invierte el orden y dice que primero etiquetan modelos preentrenados. En realidad, las personas etiquetan primero.
- El envío «automático» de los resultados a Feature Store no está documentado. Es un paso que haces tú: leer el manifiesto de salida y llamar a `PutRecord`.
- Control de calidad: **consenso** (varios anotadores por objeto) y **muestreo de auditoría** (expertos revisan un subconjunto).
- Versionar los datos etiquetados → S3 Versioning, con una regla de ciclo de vida para las versiones antiguas.

---

## 24 · Desbalance de clases: mitigación

**Qué recordar**
- Con desbalance, el modelo se sesga hacia la clase mayoritaria. Es grave cuando la minoritaria es la crítica (una enfermedad rara, el fraude).
- **Data augmentation** es la buena práctica general: generar más ejemplos de la minoritaria. En imagen: rotaciones, volteos, cambios de color. En texto: reemplazo de sinónimos, inserción aleatoria.

| Técnica | Cómo | En contra |
|---|---|---|
| Oversampling | Duplica ejemplos minoritarios o crea sintéticos; **SMOTE** interpola entre minoritarios existentes | — |
| Undersampling | Elimina al azar ejemplos de la mayoritaria | Pierde información |
| Class weighting | Da más peso en la función de pérdida a los errores en la minoritaria | — |

**Reglas/condiciones**
- Si augmentation no es viable (costo, cómputo o tiempo) → oversampling, undersampling o class weighting. Operan sobre la tabla y son casi gratis, frente a las copias de imágenes en S3 y las épocas más largas en GPU.
- Augmentation ≠ datos sintéticos: la primera parte del dataset existente; los sintéticos no usan el dataset original.
- En AWS: *Balance data* de Data Wrangler (random oversampling, random undersampling, **SMOTE**), y sus transformaciones de imagen para augmentation.
- **Class weighting no es una transformación de datos**: es un hiperparámetro del training job (p. ej., `scale_pos_weight` en el XGBoost integrado).

**Detalles examinables**
- «Técnica más común para el desbalance de clases» → **data augmentation**.

---

## 25 · SageMaker Clarify: sesgo previo al entrenamiento [C01]

> No acepta clientes nuevos. El C02 ya no nombra Clarify, pero las métricas de sesgo siguen siendo materia de examen.

**Qué recordar**
- Clarify **mide** el sesgo, **no lo mitiga**: no modifica el dataset. La mitigación la aplicas tú (p. ej., con *Balance data*).
- Infraestructura: un **processing job** con el contenedor de Clarify. Lee de S3 el dataset y un JSON de configuración, y escribe en S3 `analysis.json` y un informe.
- Las métricas previas al entrenamiento (pre-training) **no necesitan modelo**. Las posteriores (post-training) necesitan predicciones: si no das un endpoint, Clarify crea un **shadow endpoint** temporal y lo borra al terminar.
- **Faceta**: la columna del grupo que se examina (edad, género). Faceta **a** = grupo favorecido; faceta **d** = grupo desfavorecido. **Etiqueta** y **faceta** son parámetros distintos.

| Métrica | Rango | Interpretación |
|---|---|---|
| **CI** (Class Imbalance) | [−1, +1] | 0 = sin desbalance; +1 = solo hay faceta *a*; −1 = solo hay faceta *d*. El libro dice «clase mayoritaria/minoritaria» |
| **DPL** (Difference in Proportions of Labels) | [−1, +1] para etiquetas binarias o multicategoría; (−∞, +∞) para continuas | 0 = misma proporción de positivos; +1 = la faceta *a* tiene más positivos; −1 = la *d* tiene más |

**Reglas/condiciones**
- En todas las métricas, **0 (o un valor cercano) = sin desbalance**.
- Las 8 métricas pre-training: **CI, DPL, KL, JS, LP, TVD, KS, CDD**.
- **EOD** y **PPD** requieren predicciones, así que son **post-training**. La Tabla 3.1 del libro las pone entre las pre-training, lo cual es un error; las filas de sesgo en imágenes también sobran.
- SDK v2: `BiasConfig(label_values_or_threshold, facet_name, facet_values_or_threshold)` + `DataConfig` (ubicación en S3) → `SageMakerClarifyProcessor` (tipo y número de instancias, rol de IAM). En el SDK v3: `sagemaker.core.clarify.BiasConfig`.
- Reemplazo que indica AWS: calcular las métricas con pandas y scikit-learn en tus propios jobs y registrarlas en **MLflow administrado** de SageMaker AI; **SHAP** para la explicabilidad; **Bedrock Evaluations** solo para modelos fundacionales.

**Diferencias o trampas**
- CI ≈ 1 → el libro concluye «hay que submuestrear», pero es solo una opción; también valen el oversampling y la ponderación de clases.
- Si usas la propia etiqueta como faceta (el ejemplo `is_fraudulent` del libro), CI se reduce a contar el desbalance de la etiqueta.

---

## 26 · División de datos

| Conjunto | Para qué | Proporción |
|---|---|---|
| Train | Aprender patrones; ajusta los parámetros | 60–80 % |
| Validation | Ajustar **hiperparámetros** y prevenir el overfitting | 10–20 % |
| Test | Evaluación final imparcial con datos nunca vistos | 10–20 % |

**Reglas/condiciones**
- Para qué se divide → para **prevenir el overfitting y evaluar el rendimiento**.
- **Data leakage**: información del test llega al train, las métricas se inflan y el modelo generaliza mal. Prevención: separación clara, validación cruzada, cuidado con el preprocesamiento y **escalar después de dividir**. Si se detecta, hay que reevaluar con datos bien separados.
- En SageMaker: prefijos `train/`, `validation/` y `test/`. Train y validation entran al training job como **canales**; test **no** entra al entrenamiento y se usa después (evaluación o batch transform). Los prefijos separados, con permisos distintos si hace falta, evitan la fuga.

**Split data de Data Wrangler**

| Técnica | Comportamiento | Para |
|---|---|---|
| Random | Muestra aleatoria sin solapamiento. **No garantiza** las proporciones de clase (el libro dice que sí). En datasets grandes es más lenta que la ordered | Datos cuyo orden no importa |
| Ordered | El primer X % de las filas, en su orden actual, va a train. Ordena antes por la columna de tiempo | **Series temporales**; evita la fuga temporal |
| Stratified | Mantiene la proporción de clases en cada subconjunto | Clasificación **desbalanceada** |
| Split by key | Ninguna combinación de valores de las columnas clave aparece en más de una partición (p. ej., `customer_id`) | Evitar la fuga en datos **no ordenados**; mantener juntos los registros relacionados |

- Los porcentajes se configuran (p. ej., 70/20/10). La **semilla es fija por defecto**, así que la división se reproduce igual en cada ejecución; puedes cambiarla.
- **Umbral de error** entre 0 y 1, para ganar velocidad. Con 10 000 filas, una división 80/20 y un error de 0.001, train tendrá entre 7990 y 8010 filas. Con 0 la división es exacta, pero más lenta.
- Cada subconjunto es una rama del flujo que se exporta por separado, normalmente a su propio prefijo de S3.

---

## 27 · Escenarios de decisión: restricción → servicio

**Feature Store** (fraude en tiempo real, miles de autorizaciones por segundo)
- Leer las características en decenas de ms → online store + `GetRecord`, con PrivateLink en la VPC.
- Mismos valores en entrenamiento y en producción, sin programar la sincronización → online + offline en el mismo feature group, con ingesta única. `Standard_V2` actualiza solo el contador que cambió y rechaza escrituras desordenadas.
- Valor exacto en el instante de cada transacción, por auditoría → offline store (agrega, con event time, consultable con Athena) + consultas point-in-time.
- Que otros equipos reutilicen las características → búsqueda por nombre, descripción y etiquetas + IAM por grupo.
- Por qué no las alternativas:
  - DynamoDB: solo guarda el valor actual.
  - ElastiCache: sin historia (por eso `InMemory` no admite offline).
  - S3 + Athena: tarda segundos.
  - Redshift: es analítico.
  - Data Wrangler / DataBrew: calculan, pero no sirven.

**Lake Formation** (consorcio de hospitales; 40 investigadores en varias cuentas)
- Cada investigador ve solo a los pacientes de su hospital → filtro de fila. Identificadores indirectos ocultos → filtro de columna (o de celda).
- Prohibido crear copias → filtros aplicados al leer la única tabla.
- Permisos en un solo lugar y accesos auditados → concesiones centralizadas + CloudTrail.
- Varias cuentas → compartir tablas filtradas entre cuentas.
- Por qué no las alternativas:
  - IAM, políticas de bucket, Access Points y Access Grants: solo llegan a objeto o prefijo.
  - DataBrew y Glue: generan copias.
  - Macie: no controla el acceso.
  - Redshift: exige otra copia.

**S3** (drones: ~2 TB diarios de imágenes, más JSON y CSV; retención inalterable de 7 años)
- Formatos heterogéneos sin esquema previo → almacenamiento de objetos, schema-on-read.
- Crecer sin administrar discos → la capacidad no se aprovisiona.
- Athena, Glue, el entrenamiento y el offline store sin copiar datos → S3 como fuente única.
- Datos fríos baratos y retención inalterable → ciclo de vida hacia Glacier + **Object Lock en modo compliance**.
- Máxima durabilidad (una temporada perdida no se puede volver a recolectar) → 11 nueves y réplicas en varias AZ.
- Por qué no las alternativas:
  - EBS: una AZ, una instancia y tamaño fijo; Athena y Glue no lo leen.
  - EFS: NFS, ~10× más caro por GB que S3 Standard; Athena y Glue no trabajan sobre él.
  - FSx for Lustre: caché rápida vinculada a S3; complementa el lago, no lo reemplaza.
  - FSx for Windows, ONTAP y OpenZFS: pensados para migrar aplicaciones.
  - Redshift, RDS y DynamoDB: esquema al escribir; en DynamoDB cada ítem llega como máximo a 400 KB.

---

## 28 · Señales de enunciado (preguntas de repaso del capítulo)

El .md original no incluye la clave de respuestas. Estas respuestas se deducen de las reglas del capítulo.

| Señal en el enunciado | Respuesta |
|---|---|
| Pasar categóricas a numéricas en un dataset con numéricas, categóricas y ordinales (opciones: one-hot, scaling, extraction, date formatting) | One-hot encoding |
| Convertir una columna en valores binarios | One-hot encoding |
| Faltantes en características categóricas, sin distorsionar los datos ni restar confiabilidad | Imputación múltiple (no Clarify, no eliminar, no recolectar) |
| Variables con rangos muy distintos; que las de valores grandes no dominen | Ambigua. La normalización atiende la escala (criterio de los puntos esenciales del libro), pero el enunciado repite casi literalmente la frase con la que el libro describe la estandarización |
| Varias categóricas de alta cardinalidad; eficiente y económico | Feature hashing |
| Modelo de reconocimiento de autos que rinde mal; rediseñar las características de las imágenes | SageMaker JumpStart |
| Palabras → vectores con significado semántico | Word embeddings |
| Técnica más común para el desbalance de clases | Data augmentation |
| Servicio para etiquetar datos | SageMaker Ground Truth [C01] |
| Para qué dividir en train/validation/test | Prevenir el overfitting y evaluar el rendimiento |
| Servicios para detectar y tratar atípicos | Data Wrangler (*Handle outliers*) o DataBrew |
| Detectar atípicos en datos sesgados | IQR (Z-score solo con datos normales) |
| Técnicas que corrigen el sesgo | Logaritmo, raíz cuadrada, Box-Cox, Yeo-Johnson (no Z-score ni MinMax) |
| Dividir una serie temporal sin fuga | Ordered split |
| Dividir un problema de clasificación desbalanceado | Stratified split |
| Que un mismo cliente no quede en train y en test a la vez | Split by key |
| Medir el desbalance antes de entrenar | Métricas pre-training de Clarify (CI, DPL) [C01] |
