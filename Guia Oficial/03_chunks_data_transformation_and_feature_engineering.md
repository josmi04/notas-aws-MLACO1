# 03 · Transformación de datos y feature engineering — chunks de repaso

Dominio 1.2 (Transform data and perform feature engineering) y 1.3 (Ensure data integrity and prepare data for modeling). Marcas: **⚠️** = corrección al libro (docs AWS de sept. 2026 o verificación numérica) · **🕒** = cambio de estado del servicio posterior al libro · **[C01]** = contenido que la guía MLA-C02 ya no nombra explícitamente.

---

## 1 · Data lake en AWS

**Qué recordar**: S3 (almacenamiento) + Glue (catálogo) + Athena (consultas) + **Lake Formation** (gestión central de permisos, seguridad y gobernanza).

| Característica | Exigencia |
|---|---|
| Almacenamiento a escala | Recibir datos en cualquier momento: intervalos fijos o tiempo real, payloads pequeños o lotes grandes |
| Movimiento de datos | Entrada y salida del lake |
| Datos crudos | Estructurados, semiestructurados o no estructurados; el formato es irrelevante al ingerir y almacenar |
| Seguridad | Controles IAM; cifrado en reposo, en tránsito **y en uso** |
| Catálogo | Datos relacionales y no relacionales; crawling, catalogación e indexación |

**Reglas**
- S3 es el almacenamiento preferido del data lake: durabilidad de 11 nueves, alta disponibilidad e integración con los servicios de ML. Es la *single source of truth* de la mayoría de los servicios de ML de AWS.
- Costo de almacenamiento con acceso desconocido → S3 **Intelligent-Tiering**, que mueve los objetos de clase automáticamente.
- Permisos centralizados, monitoreo de accesos y compliance a escala → **Lake Formation**.

---

## 2 · Tipos de datos → técnica de feature engineering

| Tipo | Subtipos / claves |
|---|---|
| Categórico | **Ordinal** (orden lógico: S, M, L, XL) · **Nominal** (sin orden: estados de EE. UU.) |
| Numérico | **Discreto** (contable: número de quejas) · **Continuo** (infinitos valores entre dos; numérico o fecha/hora) |
| Texto | NLP: sentimiento, traducción, clasificación |
| Imagen | Píxeles → bordes, formas, colores, patrones (visión por computadora) |
| Serie temporal | Observaciones a intervalos regulares con timestamp; **el orden importa** |

**Regla**: el tipo de dato determina la técnica. Aunque todos los datasets sean estructurados, pueden tener esquemas distintos y requerir transformación.

---

## 3 · Extracción vs selección vs creación de features

**Qué recordar**: el feature engineering **no agrega datos nuevos**; hace más útiles los que ya existen. A menudo requiere conocimiento del dominio.

| Técnica | ¿Reduce dimensionalidad? | Qué hace | Ejemplo |
|---|---|---|---|
| Extracción | **Sí** | Crea features nuevas a partir de las existentes, de forma automática; común en imagen, audio y texto | Píxeles → `windshield_present`, `headlight_present`; en NLP, palabras más frecuentes sin artículos ni preposiciones |
| Selección | **Sí** | Rankea las features por importancia predictiva y conserva las más relevantes; elimina las irrelevantes y las redundantes (correlacionadas) | Quitar *revenue* si se mueve en paralelo a *sales*; quitar *net profit* si el target son las ventas del día |
| Creación / transformación | **No** | Genera features nuevas desde las existentes | `dd-mm-yy` → día, mes y año por separado; agregar el día de la semana |

**Reglas**
- A mayor dimensionalidad (número de features), más difícil es entrenar bien el modelo.
- La selección se usa a menudo junto con la extracción. Los filtros usan medidas estadísticas de relación con el target.
- Extracción en datos estructurados: las features deben poder **replicarse fácilmente en producción** y en el pipeline automatizado.

**Trampa**: creación/transformación **no** es una técnica de reducción de dimensionalidad.

---

## 4 · Amazon SageMaker Feature Store

**Qué recordar**: repositorio totalmente administrado para **almacenar, compartir y gestionar** features entre equipos (data scientists, ingenieros de datos y de ML) a lo largo del ciclo de vida de ML.

**Reglas**
- Resuelve la sincronización entre las features de entrenamiento offline (batch) y las de inferencia en tiempo real.
- Evita reextraer features para cada modelo nuevo: se reutilizan entre modelos y proyectos, lo que da consistencia.
- Es el destino típico de las features ya procesadas (también las de imágenes).

**Trampa**: Feature Store **almacena** features; no las transforma (eso lo hace Data Wrangler).

---

## 5 · Valores faltantes

| Estrategia | Cuándo |
|---|---|
| **Recolectar** | Faltan muchos datos en varias features. Es caro y lento: comparar el costo de recolectar, el de un modelo de bajo rendimiento y el de imputar o eliminar |
| **Imputar** | Pocos faltantes, distribuidos al azar (típico de un fallo de ingesta) |
| **Eliminar la feature** | Faltan muchos datos, pero solo en **una** feature. Riesgo: quedarse sin datos suficientes si se eliminan demasiadas features |

**Reglas de imputación**
- Distribución normal → **media**.
- No normal → **mediana** o **valor más frecuente**. En features categóricas, el valor más frecuente.

**Herramientas**: `SimpleImputer` de scikit-learn desde SageMaker; **Glue DataBrew** con transformaciones integradas.

**Trampa**: la mayoría de los algoritmos no manejan faltantes por sí solos, así que la decisión es humana.

---

## 6 · Outliers: concepto y tratamiento

**Qué recordar**
- Un outlier se desvía significativamente de la media; por convención, está a **más de 3σ** (en una normal, ±3σ contiene el 99.7 % de los datos).
- Puede ser **natural** (refleja una verdad del dato) o **artificial** (un error).
- **Ruido** ≠ outlier: ruido = datos erróneos; outlier = dato que se desvía mucho de la media.
- Los outliers suelen sesgar la distribución (cola larga) y confundir al algoritmo.

| Tratamiento | Cuándo / efecto |
|---|---|
| Eliminar | El outlier es ruido o error artificial; no afecta la calidad del modelo |
| Transformación logarítmica | Se aplica a toda la feature (no solo al outlier): comprime el rango, reduce el sesgo y acerca el outlier al resto sin perder su significado (log₁₀ 1000 = 3) |
| Imputar (media o mediana) | Excelente si el outlier viene de un error artificial |

**Servicios para detectar y tratar outliers**: **SageMaker Data Wrangler** y **AWS Glue DataBrew**. DataBrew detecta con Z-score o IQR y luego **reemplaza, elimina, reescala o marca (flag)** los outliers.

---

## 7 · Detección de outliers: IQR vs Z-score

| Método | Distribución | Regla |
|---|---|---|
| **IQR** | **Sesgada** (robusto al sesgo) | IQR = Q3 − Q1 · outlier si < Q1 − 1.5·IQR o > Q3 + 1.5·IQR |
| **Z-score** | **Normal** | z = (x − μ)/σ · outlier si \|z\| > 3 |

**Reglas**
- Primero mira el **histograma** de cada feature para elegir el método.
- Sesgo a la derecha = la mayoría de los datos en valores bajos y una cola larga hacia valores altos (ingresos, examen difícil).

**Trampa**: en datos sesgados, el Z-score puede **no detectar** el outlier. En el ejemplo del libro, IQR detecta [25, 23] y Z-score no.

⚠️ En ese ejemplo (n = 10), el Z-score también falla por dos motivos que el libro no menciona. El outlier infla μ y σ (z ≈ 2.4 y 2.6). Además, con σ poblacional, |z| ≤ √(n−1) = 3, así que ningún punto de 10 puede superar 3. La regla de examen se mantiene: **sesgada → IQR; normal → Z-score**.

---

## 8 · Deduplicación, reformateo y ruido

**Deduplicación**
- Los duplicados sesgan resultados, causan **overfitting** (el modelo aprende ruido) e introducen **sesgo** si unas entradas se repiten más que otras (datos desbalanceados).
- Opción más simple, visual y sin código → **AWS Glue DataBrew**.

**Estandarización y reformateo**
- Estandarizar evita que las features con valores grandes dominen. Es clave en algoritmos sensibles a la escala (regresión lineal, SVM).
- Reformatear = tipos de dato, formatos (fechas) y unidades uniformes, con la misma estructura en todas las entradas.
- Flujo típico: cargar desde S3 en un notebook de SageMaker → `StandardScaler` → exportar a S3 → **job de Glue** para reformatear → entrenar.

**Ruido y errores**: un modelo entrenado con datos ruidosos aprende rarezas del training set → **overfitting** (bien en entrenamiento, mal con datos nuevos). Limpiarlos mejora la generalización.

---

## 9 · Normalización vs estandarización

| | Normalización | Estandarización |
|---|---|---|
| Resultado | Rango común, normalmente **[0, 1]** | **Media 0, desviación estándar 1** (varianza 1) |
| Implementación | **MinMax** scaling | **Z-score**: (x − μ)/σ |
| Foco | **Escala** (rango) | **Distribución** (centrar y escalar) |
| ¿Centra en 0? | No es necesario | Sí |
| Beneficios / uso | Peso igual entre features; algoritmos de distancia (**k-NN**, redes neuronales); convergencia más rápida del gradient descent; interpretabilidad de coeficientes | Datos aproximadamente normales; regresión lineal/logística, algoritmos con gradient descent |

**Reglas de examen**
- "Importa la escala / que ninguna feature domine por su magnitud" → normalización (MinMax).
- "Importa la distribución / datos normales" → estandarización (Z-score).
- Ambas son formas de *scaling* (ajustar rango o distribución).

**Trampa**: **ni Z-score ni MinMax corrigen el sesgo**. Cambian la escala o el centro, pero la forma de la distribución se mantiene.

---

## 10 · Técnicas de scaling

| Técnica (sklearn) | Fórmula / rango | Cuándo | Ojo |
|---|---|---|---|
| MinMax (`MinMaxScaler`) | [0, 1] | Normalizar a un rango fijo | Preserva las relaciones; **no corrige el sesgo** |
| Z-score (`StandardScaler`) | μ = 0, σ = 1 | Datos normales | No corrige el sesgo; μ y σ se ven afectados por outliers |
| **Robust** (`RobustScaler`) | (x − mediana)/IQR | **Outliers** y datos sesgados | Menos sensible a outliers |
| **MaxAbs** (`MaxAbsScaler`) | x / máx\|x\| → [−1, 1] | Datos **dispersos** (sparse) o con muchos ceros | **Preserva los ceros** y la distribución original |
| **Power** (`PowerTransformer`) | Box-Cox / Yeo-Johnson | Estabilizar la varianza y acercar a la normal | **Box-Cox: solo valores positivos**; **Yeo-Johnson: positivos y negativos** |

**Reglas**
- Outliers → Robust scaling.
- Sparse/ceros → MaxAbs.
- Varianza alta o sesgo → power transform (con negativos → Yeo-Johnson).

---

## 11 · Transformaciones contra el sesgo (log, raíces)

**Logarítmica**
- Reduce el **sesgo a la derecha**: comprime la cola y acerca la distribución a la normal.
- **Limita el impacto de los outliers** sin eliminarlos.
- **Linealiza** relaciones multiplicativas o exponenciales: y = 1, 10, 100, 1000 → log₁₀ y = 0, 1, 2, 3.
- **No se aplica a valores 0 o negativos** (log indefinido en cualquier base) → desplazar los datos a positivos o usar raíz cúbica.

**Raíces**

| Transformación | Dominio | Efecto |
|---|---|---|
| Raíz cuadrada | ≥ 0 (positivos y cero) | Menor que el del log |
| Raíz cúbica | **Cualquier real, incluidos negativos y 0** | Menor que el del log |

**Resumen de examen**
- **Corrigen el sesgo**: log, raíz cuadrada, raíz cúbica, Box-Cox, Yeo-Johnson.
- **No lo corrigen**: Z-score, MinMax.
- **Manejan outliers**: eliminar, log, imputar (media o mediana), Robust scaling, binning.

---

## 12 · Binning (bucketing)

**Qué recordar**: divide datos numéricos continuos en intervalos discretos (bins), es decir, **numérico → categórico**. Simplifica el modelo y lo hace **más robusto a outliers y ruido**.

**Ejemplo**: el peso de un vehículo se convierte en 5 columnas (`is_minicompact`, `is_subcompact`, `is_compact`, `is_midsize`, `is_large`).

---

## 13 · Encoding de datos categóricos

**Qué recordar**: encoding = string → formato numérico (entero, vector, matriz o tensor), porque la mayoría de los algoritmos solo entienden números.

| Técnica | Resultado | Úsala para | Riesgo |
|---|---|---|---|
| **Label** | Un entero único por categoría (White, Black, Red → 0, 1, 2) | Datos **ordinales**; modelos **basados en árboles** | Impone un orden artificial que confunde a los algoritmos que interpretan los números como magnitudes |
| **One-hot** | Una columna binaria por categoría distinta (1 presente / 0 ausente) | Nominales de **baja cardinalidad**; modelos **no basados en árboles** (regresión lineal, k-NN, redes neuronales) | **Aumenta la dimensionalidad** con alta cardinalidad |
| **Binary** | Categoría → número → binario → **una columna por bit** (~log₂ k columnas) | Reducir la explosión dimensional del one-hot | **Pérdida de información**: difumina las distinciones entre categorías |
| **Feature hashing** | Función hash (mód n) → vector de **tamaño fijo n** elegido por el ingeniero | **Alta cardinalidad**, datasets grandes, memoria acotada, **costo-eficiente** | **Colisiones**, mitigadas con suficientes buckets |

**Ejemplo del libro** (10 filas, 8 colores distintos): one-hot da 8 columnas; binary (category_encoders), 4 columnas; hashing con n_features = 3, 3 columnas. El número de filas (10) no cambia.

**Reglas**
- Mitigar la dimensionalidad del one-hot → binary encoding, o **PCA después del one-hot**.
- Alta cardinalidad + eficiencia + costo → feature hashing.
- Convertir una columna en valores binarios (0/1) → one-hot.

---

## 14 · Series temporales en SageMaker Data Wrangler

**Qué recordar**: solución **low-code** para limpiar, transformar y preparar series temporales según el formato de entrada del modelo de forecasting. Primero se usan sus **visualizaciones** para entender los patrones y luego se crean las features.

| Transformación | Qué hace |
|---|---|
| **Featurize datetime** | Primer paso (buena práctica): desagrega el timestamp en `date_month`, `date_day`, `date_week_of_year`, `date_day_of_year` y `date_quarter` |
| **One-hot encode** | Trata componentes de fecha como categóricos (`date_quarter` 0–3 → 4 columnas binarias) |
| **Lag features** | Valores del **target** en timestamps previos. Capturan la **autocorrelación** (correlación serial: la serie con sus propios valores pasados). Genera varios lags sobre una ventana |
| **Rolling window features** | Estadísticas sobre las observaciones de una ventana. La extracción automática usa el paquete open source **tsfresh** |

**Trampa**: lags y ventanas son técnicas de **series temporales**; no sirven para imágenes ni para texto.

---

## 15 · Feature engineering de imágenes

| Paso | Servicio |
|---|---|
| Extraer features (vía principal) | **SageMaker JumpStart**: modelos de visión preentrenados (CNN → vectores de features); detección de objetos, clasificación, etc. |
| Análisis básico | **Amazon Rekognition**: objetos, escenas, texto y rostros; etiquetas y bounding boxes sin esfuerzo manual |
| Transformar | **Data Wrangler**: EDA, normalización de píxeles, **PCA** para reducir dimensionalidad |
| Almacenar y reutilizar | **Feature Store** |

**Trampa**: "reentrenar o mejorar las features de un modelo que reconoce autos" → JumpStart, no tokenización, lags ni binning en Data Wrangler.

---

## 16 · Feature engineering de texto: servicios

| Necesidad | Servicio |
|---|---|
| Features de alto nivel: **entidades, frases clave, sentimiento, idioma** | **Amazon Comprehend** |
| Extraer texto de **documentos** → información estructurada | **Amazon Textract** |
| Extracción personalizada: **word embeddings** (Word2Vec), **topic modeling** (LDA) | Algoritmos integrados y frameworks de **SageMaker** (cap. 4) |

**Trampa**: Comprehend **analiza** texto; Textract **extrae** texto de documentos.

---

## 17 · Técnicas de texto (NLP)

| Técnica | Qué hace | Ejemplo |
|---|---|---|
| **Tokenización** | Divide el texto en tokens (palabras o frases). Es el **primer paso** | "Machine learning is fascinating" → 4 tokens |
| **Stop words** | Elimina palabras comunes sin información (the, is, and) → menos ruido | "The cat is on the mat" → [cat, on, mat] |
| **Stemming** | Recorta prefijos o sufijos | running → run |
| **Lematización** | Reduce a la forma base con **reglas lingüísticas** | running → run |
| **N-grams** | Secuencias contiguas de n ítems; capturan contexto | Bigramas: (Machine, learning), (learning, is), (is, fun) |
| **Word embeddings** | Palabras → **vectores continuos** con significado semántico (Word2Vec, GloVe) | King y Queen quedan cerca en el espacio vectorial |

**Reglas**
- Palabras → vectores numéricos **con semántica** → word embeddings (no one-hot ni tokenización).
- Stemming = corte mecánico; lematización = reglas lingüísticas.

**Bedrock y tokens**: los foundation models de Bedrock tokenizan el prompt para interpretarlo. El precio se calcula por **tokens de entrada + tokens de salida**: un prompt más largo cuesta más.

---

## 18 · Etiquetado: Amazon SageMaker Ground Truth [C01]

🕒 **Ground Truth** y **Mechanical Turk** están cerrados a clientes nuevos desde el **30-07-2026**. Los clientes existentes siguen usándolos, pero no habrá nuevas funciones. La guía C02 ya no nombra Ground Truth, aunque el concepto de etiquetado sigue en el temario. MTurk salió del alcance de C02.

**Qué recordar**: el etiquetado agrega **ground truth** (las respuestas correctas) a los datos crudos. Es imprescindible en aprendizaje supervisado. Feature engineering = modificar o crear features; etiquetado = enriquecer con labels.

**Flujo**: datos crudos en **S3** → labeling job (tarea + instrucciones) → tipo de tarea (clasificación de imagen o texto, detección de objetos…) → etiquetado automático opcional → revisión humana → guardar las etiquetas **versionadas** (S3 o Feature Store) → entrenar en SageMaker.

**Fuerza laboral**: **Mechanical Turk** (crowdsourcing global), **fuerza privada** o **proveedores externos**.

**Control de calidad**
- **Consensus labeling**: varios anotadores etiquetan el mismo dato.
- **Audit sampling**: expertos revisan un subconjunto.

⚠️ **Automated data labeling** no consiste en "etiquetar con modelos preentrenados", como dice el libro. Es **active learning**:
1. Una muestra aleatoria va a humanos.
2. Con esas etiquetas se entrena y valida un modelo.
3. El modelo etiqueta solo los objetos que superan un **umbral de confianza**.
4. Los de baja confianza van a humanos y el ciclo se repite.

Reduce costo y tiempo, pero genera costos de entrenamiento e inferencia en SageMaker. Solo está disponible para algunos tipos de tarea integrados.

---

## 19 · Desbalance de clases: técnicas

**Qué recordar**: una clase mayoritaria (p. ej., personas sanas frente a una enfermedad rara) sesga el modelo hacia ella. Esto afecta el rendimiento y la equidad justo cuando la clase minoritaria es la crítica.

| Técnica | Cómo | Contra |
|---|---|---|
| **Data augmentation** (la más común y recomendada) | Más datos de la minoritaria a partir de los existentes: imagen (rotaciones, flips, color), texto (sinónimos, inserción aleatoria) | Costo, cómputo y tiempo |
| **Oversampling** | Duplicar ejemplos de la minoritaria o generar sintéticos. **SMOTE** interpola entre ejemplos de la minoritaria | — |
| **Undersampling** | Eliminar al azar ejemplos de la mayoritaria | **Pérdida de información** |
| **Class weighting** | Mayor peso en la función de pérdida a los errores sobre la minoritaria | — |

**Regla**: augmentation primero; si no es viable por costo, cómputo o tiempo → oversampling, undersampling o class weighting.

**Trampa**: data augmentation (nuevos datos **derivados del training set**) ≠ datos sintéticos (generados **sin usar** el dataset original).

---

## 20 · SageMaker Clarify: métricas de sesgo pre-entrenamiento [C01]

🕒 **Clarify** está cerrado a clientes nuevos desde el **30-07-2026**; los existentes siguen usándolo. La guía C02 ya no lo nombra, aunque el concepto de medir el sesgo en los datos sigue.

**Qué recordar**: Clarify calcula métricas **sobre los datos crudos antes de entrenar**. Son **agnósticas al modelo** y cada una corresponde a una noción distinta de equidad. Se configuran en el SDK con `BiasConfig`.
- **Facet**: la feature que se analiza por sesgo.
- **Facet a**: valor favorecido por el sesgo.
- **Facet d**: valor desfavorecido.

| Métrica | Fórmula | Rango | Interpretación |
|---|---|---|---|
| **Class Imbalance (CI)** | (nₐ − n_d)/(nₐ + n_d) | [−1, +1] | +1 = solo miembros de a · 0 = balance · −1 = solo miembros de d |
| **Difference in Proportions of Labels (DPL)** | qₐ − q_d (proporción de etiquetas positivas en cada facet) | [−1, +1] binaria o multicategoría; (−∞, +∞) continua | + = a tiene más resultados positivos · 0 = paridad demográfica · − = d tiene más |

**Reglas**
- En todas las métricas, un valor **cercano a 0 indica ausencia de sesgo o desbalance**.
- Con la métrica calculada se decide la acción: oversampling, undersampling o class weighting.
- Ejemplo del libro: 99.9 % de transacciones no fraudulentas con facet a = 0 → CI ≈ +1 → rebalancear (el libro propone undersampling).

**Trampas**
- CI mide el desbalance en el **número de miembros** entre valores del facet (representación). Para comparar **resultados de la etiqueta** entre grupos se usa DPL.
- ⚠️ La tabla 3.1 del libro incluye **EOD** y **PPD**, pero no son métricas pre-entrenamiento de Clarify: requieren predicciones del modelo. Las pre-entrenamiento son **CI, DPL, KL, JS, LP, TVD, KS y CDD**.

---

## 21 · Train / validation / test y data leakage

| Split | Propósito | Proporción típica |
|---|---|---|
| **Training** | El modelo aprende patrones y ajusta sus parámetros | 60–80 % |
| **Validation** | Ajustar **hiperparámetros** y detectar overfitting | 10–20 % |
| **Test** | Datos **nunca vistos** → evaluación **imparcial** final de la generalización | 10–20 % |

**Data leakage**: información de test se filtra en el entrenamiento. Resultado: métricas infladas artificialmente y mala generalización.

**Cómo evitarlo**
- Separación estricta de los splits.
- Validación cruzada.
- Vigilar los pasos de preprocesamiento.
- **Normalizar o estandarizar después del split**: calcular los parámetros del scaler solo con train y aplicarlos a validation y test.
- Si se detecta leakage → reevaluar con datos bien separados.

**Regla**: "propósito del split" → **prevenir overfitting y evaluar el rendimiento**.

---

## 22 · Split en SageMaker Data Wrangler

**Qué recordar**: la transformación **Split data** divide el dataset en 2 o 3 subconjuntos con porcentajes configurables (p. ej., 70/20/10), con poco o nada de código.

| Método | Qué garantiza | Cuándo |
|---|---|---|
| **Random** | Muestras aleatorias sin solapamiento | No hace falta preservar el orden |
| **Ordered** | Conserva el orden secuencial (los primeros X % van a train) → sin solapamiento pasado/futuro | **Series temporales** o cuando importa el orden |
| **Stratified** | Cada split mantiene la **misma proporción de clases** que el original | **Clasificación desbalanceada** |
| **Split by key** | Ninguna combinación de valores de las columnas clave aparece en más de un split (p. ej., un `customer_id` solo en un split) | Evitar leakage en datos **no ordenados**; mantener agrupados los registros relacionados |

⚠️ El libro atribuye al **random split** que cada subconjunto conserve la distribución de categorías. Eso solo lo garantiza el **stratified split**; el random solo toma muestras aleatorias sin solapamiento.

**Trampas**
- Series temporales + evitar leakage → ordered, no random.
- Mismo cliente o entidad en train y test → split by key.

---

## 23 · Necesidad → servicio (visión del capítulo)

Orden del proceso: limpieza → feature engineering → etiquetado → desbalance → split → entrenamiento.

| Necesidad | Servicio |
|---|---|
| Limpieza visual sin código: duplicados, faltantes, outliers (Z-score/IQR) | **AWS Glue DataBrew** |
| Feature engineering low-code, series temporales, EDA y visualizaciones, split | **SageMaker Data Wrangler** |
| Reformatear o transformar datos a escala (ETL) | **Job de AWS Glue** |
| Código propio (pandas, scikit-learn) | Notebooks de **SageMaker** |
| Almacenar, compartir y reutilizar features | **SageMaker Feature Store** |
| Features de imágenes | **JumpStart** (preentrenados) · **Rekognition** (básico) |
| Features de texto | **Comprehend** (análisis) · **Textract** (extracción de documentos) |
| Etiquetado | **SageMaker Ground Truth** 🕒 |
| Sesgo en los datos antes de entrenar | **SageMaker Clarify** 🕒 |
| Gobernanza y permisos del data lake | **Lake Formation** |

---

## 24 · Escenarios tipo (preguntas de repaso) → regla

| Escenario | Regla |
|---|---|
| Convertir categóricas a numéricas para mejorar la precisión (entre las opciones: one-hot, scaling, extracción, formato de fecha) | **One-hot encoding** |
| Convertir una columna en valores binarios | **One-hot encoding** |
| Faltantes en features categóricas sin distorsionar los datos | **Imputación** (la opción *multiple imputations*); no eliminar la feature, no recolectar, no Clarify |
| Features con magnitudes muy distintas; evitar que las de valores grandes dominen | **Normalización** (el libro asocia "igual peso por escala" a normalización; la estandarización también iguala escalas, pero se elige por la distribución) |
| Categóricas de alta cardinalidad, eficiente y costo-efectivo | **Feature hashing** |
| Mejorar las features de un modelo de imágenes de autos | **SageMaker JumpStart** |
| Palabras → vectores con significado semántico | **Word embeddings** |
| Técnica más común contra el desbalance de clases | **Data augmentation** |
| Etiquetado de datos en AWS | **SageMaker Ground Truth** (no Comprehend, Rekognition ni Clarify) |
| Propósito del split train/validation/test | **Prevenir overfitting y evaluar el rendimiento** |

---

*Fuentes de las correcciones ⚠️/🕒: docs de SageMaker AI (métricas pre-entrenamiento de Clarify: CI y DPL; automated data labeling de Ground Truth; transformación Split data de Data Wrangler; avisos de disponibilidad de Clarify y Ground Truth). La cota del Z-score se verificó numéricamente con el dataset del libro.*
