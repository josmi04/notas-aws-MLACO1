# Capítulo 3 — Transformación de datos e ingeniería de features

**Objetivos del examen cubiertos** (Dominio 1: Preparación de datos para ML):

- 1.2 Transformar datos y realizar ingeniería de features.
- 1.3 Garantizar la integridad de los datos y prepararlos para el modelado.

> **Nota de vigencia (sep-2026).** Desde el 30-jul-2026, SageMaker Clarify y SageMaker Ground Truth (y el uso de Mechanical Turk desde SageMaker AI) no admiten clientes nuevos; quienes ya los usaban los conservan. En el MLA-C01 siguen siendo la respuesta esperada para sesgo previo al entrenamiento y para etiquetado. El MLA-C01 en inglés se ofrece hasta el 28-sep-2026; su sucesor (MLA-C02) ya no nombra estos servicios, aunque los conceptos se mantienen.

---

## Introducción: del data lake a datos listos para entrenar

En el capítulo 2 se ingirieron y almacenaron los datos en AWS. Ahora toca prepararlos para entrenar el modelo: es la fase **Process Data** del ciclo de vida de ML.

Los datos suelen estar en crudo en un **data lake**, un repositorio central que almacena volúmenes masivos de datos para que distintos equipos los cataloguen, procesen, enriquezcan y consuman. Sus características clave:

- **Almacenamiento a escala:** acepta datos en cualquier momento (por intervalos o en tiempo real), en cargas pequeñas o en grandes lotes.
- **Movimiento de datos:** permite que los datos entren y salgan del lago.
- **Datos en crudo:** estructurados, semiestructurados o no estructurados, de distintas fuentes; el formato no importa durante la ingesta y el almacenamiento.
- **Seguridad:** como puede contener datos sensibles, necesita controles robustos de IAM y cifrado tanto en reposo como en tránsito.
- **Catálogo:** almacena datos relacionales (bases operacionales, aplicaciones de negocio) y no relacionales (apps móviles, IoT, redes sociales), y permite saber qué hay en el lago mediante *crawling*, catalogación e indexación.

En AWS, un data lake se construye normalmente con **Amazon S3** (almacenamiento), **AWS Glue** (catálogo), **Amazon Athena** (consultas) y **AWS Lake Formation** (gestión centralizada de permisos, seguridad y gobierno de datos, monitoreo de accesos y cumplimiento).

Para el examen: **S3 es el almacenamiento preferido para un data lake en AWS** porque ofrece durabilidad de 99.999999999 % (11 nueves), alta disponibilidad, seguridad e integración con los servicios de procesamiento y ML de AWS. Funciona como fuente única de verdad para la mayoría de los servicios de ML. Con la clase **S3 Intelligent-Tiering** se reduce el coste de almacenamiento sin gestión manual.

> **Corrección:** Intelligent-Tiering no mueve los objetos a otra *clase* de almacenamiento. Los mueve automáticamente entre **niveles de acceso** dentro de la misma clase: Frequent → Infrequent (30 días sin acceso) → Archive Instant Access (90 días), más dos niveles de archivo opcionales (Archive Access y Deep Archive Access).

El objetivo de esta fase es producir datos de calidad: cuanto mejor sea el dataset de entrenamiento, más rápido aprende el modelo y más precisas son sus inferencias.

---

## 1. Entender el feature engineering

### 1.1 Tipos de datos

| Tipo | Descripción | Ejemplos |
|---|---|---|
| **Categórico** | Número finito de categorías, normalmente texto. **Ordinal**: con orden lógico. **Nominal**: sin orden. | Talla S/M/L/XL (ordinal); los 50 estados de EE. UU. (nominal) |
| **Numérico** | **Discreto**: número contable de valores entre dos valores cualesquiera. **Continuo**: infinitos valores entre dos valores; puede ser numérico o fecha/hora. | Nº de quejas o de defectos (discreto); longitud de una pieza, fecha y hora de un pago (continuo) |
| **Texto** | Contenido escrito que se procesa con NLP. | Reseñas, posts; análisis de sentimiento, traducción, clasificación de texto |
| **Imagen** | Valores de píxeles de los que se extraen bordes, formas, colores y patrones. | Detección de objetos, reconocimiento facial, clasificación de imágenes |
| **Serie temporal** | Observaciones registradas a intervalos regulares, cada una con su *timestamp*; el orden cronológico es esencial. | Tendencias, patrones y variaciones a lo largo del tiempo |

Identificar el tipo de dato es el primer paso, porque el data lake combina datasets de fuentes distintas: estructurados, semiestructurados o no estructurados, e incluso los estructurados pueden tener esquemas diferentes. Todo eso debe transformarse antes de alimentar un algoritmo.

### 1.2 Qué es el feature engineering

El **feature engineering** es la ciencia (y el arte) de extraer más información de los datos existentes para mejorar la capacidad predictiva del modelo y acelerar su aprendizaje. **No se agregan datos nuevos**: se reorganizan los que ya hay en un conjunto de features que el algoritmo pueda usar directamente. Suele apoyarse en conocimiento del dominio, y el tipo de dato determina la técnica más adecuada.

Una **feature** (o atributo) es una propiedad medible e individual de los datos; es un *input* que el modelo usa para predecir. En un dataset de ventas, por ejemplo, las features pueden ser la fecha, el número de visitantes y el número de pedidos.

**Ejemplo.** Se quiere predecir las ventas de un día. En los datos, las dos primeras filas tienen ventas mucho mayores; el análisis revela que los clientes compran más los fines de semana. Es decir, el **día de la semana** influye en las compras, así que se crea esa feature a partir de la fecha con un script sencillo. La idea central del feature engineering es descubrir esa información "oculta" y transformar el dataset para aprovecharla.

### 1.3 Áreas del feature engineering

Las tres áreas principales son:

1. **Extracción de features**
2. **Selección de features**
3. **Creación y transformación de features**

Las dos primeras buscan **reducir la dimensionalidad**, es decir, el número de features del dataset. A mayor dimensionalidad, más le cuesta al modelo encontrar los patrones que buscamos.

#### Extracción de features

Reduce automáticamente la dimensionalidad **creando features nuevas a partir de las existentes**. Es habitual en datasets con muchas features, sobre todo de imagen, audio o texto.

Antes de las redes neuronales, una forma de analizar imágenes era extraer características concretas. Si la imagen es un coche, se extraen elementos como parabrisas, faros, intermitentes y ruedas como features independientes; en lugar de píxeles en crudo, el dataset tiene columnas como `windshield_present` o `headlight_present`, que facilitan que el algoritmo aprenda a reconocer coches.

Los propios datos sugieren la técnica: en imágenes, extraer elementos clave; en NLP, por ejemplo, extraer las palabras más frecuentes, excluyendo artículos y preposiciones.

#### Selección de features

**Ordena las features existentes según su importancia predictiva y conserva solo las más relevantes.** Suele combinarse con la extracción. Elimina dos tipos de features:

- **Irrelevantes** para el problema.
- **Redundantes**, por estar muy correlacionadas con otras features.

Los métodos de filtrado usan medidas estadísticas para identificar las features con una relación fuerte con la variable objetivo (*target*). Estas técnicas también se aplican a datos no estructurados (imágenes, audio, vídeo), que igualmente requieren reducir la dimensionalidad.

> **Corrección:** en el ejemplo del libro (target = ventas del día) se eliminan *revenue* y *net profit* argumentando que "se mueven en paralelo a las ventas" o que "no son relevantes". La decisión es correcta, pero el motivo es otro: ambas se calculan a partir de las ventas, así que no estarían disponibles al momento de predecir y usarlas sería una **fuga de información del target** (*target leakage*). Una feature muy correlacionada con el target suele ser valiosa; la redundancia se refiere a features muy correlacionadas **entre sí**.

#### Creación y transformación de features

A diferencia de las anteriores, **no reduce la dimensionalidad**: genera features nuevas a partir de las existentes. Ejemplo: si la fecha está como `dd-mm-yy`, combinar día, mes y año en una sola feature puede no ayudar; separarla en tres features (día, mes, año) puede revelar una relación significativa entre alguna de ellas y el target.

### 1.4 Amazon SageMaker Feature Store

**SageMaker Feature Store** es un repositorio totalmente administrado para **almacenar, compartir y gestionar features** de modelos de ML. Por ejemplo, en un recomendador de libros por tema, las features podrían ser la afinidad del libro con el tema, sus valoraciones y su fecha de publicación.

Resuelve dos problemas:

- Las features las usan varios equipos (científicos de datos, ingenieros de datos y de ML), y su calidad es crítica para la precisión del modelo.
- Es difícil mantener sincronizadas las features usadas para entrenar offline (en lotes) con las que se sirven para inferencia en tiempo real. Feature Store ofrece un almacén unificado y seguro para procesar, estandarizar y usar features a escala en todo el ciclo de vida (con un *online store* de baja latencia para inferencia y un *offline store* en S3 para entrenamiento).

---

## 2. Limpieza y transformación de datos

La limpieza es el paso de preprocesamiento previo al feature engineering: convierte datos "sucios" en una base estructurada, precisa y fiable.

| Tarea | Por qué importa para el feature engineering |
|---|---|
| **Gestionar valores faltantes** | Imputar, eliminar registros incompletos o gestionarlos de otro modo evita features sesgadas o imprecisas por huecos en los datos. |
| **Detectar y tratar outliers** | Evita que distorsionen la distribución de las features y el rendimiento del modelo. |
| **Deduplicar** | Quitar registros duplicados hace el dataset más eficiente y menos propenso al sobreajuste. |
| **Reformatear y estandarizar** | Tipos, unidades y formatos uniformes (p. ej., fechas y unidades de medida) hacen comparables las features. |
| **Eliminar ruido y errores** | Las features se construyen sobre datos correctos, lo que da modelos más robustos. |

### 2.1 Valores faltantes

Es habitual que falten datos, por errores de recolección o porque la fuente no los tenía. Los faltantes dificultan que el algoritmo interprete la relación entre la feature y el target, y **la mayoría de los algoritmos no los gestionan automáticamente**: hace falta criterio humano para reemplazarlos por algo significativo.

La técnica depende de cuántos datos faltan:

| Técnica | Cuándo | Consideraciones |
|---|---|---|
| **Recolectar** | Faltan muchos datos y en varias features. | La ingesta es cara y lenta: hay que sopesar el coste de recolectar, el de tener un modelo de bajo rendimiento y las alternativas (imputar o eliminar). |
| **Imputar** | Faltan pocos valores y están repartidos aleatoriamente (p. ej., por un fallo de ingesta). | Rellenar con la **media** si la distribución es normal; si no, con la **mediana** o el **valor más frecuente** (moda). |
| **Eliminar** | Faltan muchos datos pero concentrados en una misma feature. | Se descarta la feature entera; si se eliminan demasiadas, puede no quedar información suficiente para el modelo. |

En un notebook de SageMaker puedes usar `SimpleImputer` de scikit-learn:

> Los snippets del libro usan comillas tipográficas (‘ ’ “ ”) y el signo menos Unicode (−), que Python rechaza. Aquí van corregidos.

```python
from sklearn.impute import SimpleImputer
import numpy as np

A = np.array([[1, 2], [None, 4], [5, None]])
imputer = SimpleImputer(strategy='mean')
B = imputer.fit_transform(A)
print(B)
# [[1. 2.]
#  [3. 4.]
#  [5. 3.]]   ← cada faltante se reemplaza por la media de su columna (3 en ambas)
```

**AWS Glue DataBrew** también rellena valores faltantes con transformaciones integradas.

### 2.2 Detección y tratamiento de outliers

Un **outlier** es un punto que se desvía significativamente de la media. Como regla general, se considera outlier si está a **más de tres desviaciones estándar** de la media: en una distribución normal, ≈99.7 % de los datos queda dentro de ±3σ.

Los algoritmos de ML son muy sensibles a la distribución y al rango de las features, y los outliers tienden a confundirlos durante el entrenamiento.

**Dataset de ejemplo** (x = consumo de agua por día, y = consumo de energía por día; valores ilustrativos):

```
[[4, 11], [3.8, 12], [4.5, 12.5], [8, 8], [9, 8.5], [9.5, 7.5], [13, 5], [14, 4.7], [13.7, 6], [25, 23]]
```

Los datos forman tres grupos y un punto, **[25, 23]**, se aleja claramente del resto.

Conceptos clave:

- **Outlier artificial vs. natural:** algunos se deben a errores; otros son fenómenos reales que reflejan una verdad de los datos. El ingeniero de ML decide, con técnicas estadísticas, si el outlier se queda, se modifica o se elimina.
- **Ruido vs. outlier:** el ruido son datos erróneos; un outlier es un dato muy alejado de la media, que puede ser legítimo.

**Tratamientos:**

| Enfoque | Cuándo / cómo |
|---|---|
| **Eliminar** | Si el outlier se debe a ruido o a un error artificial, se borra sin afectar la calidad del modelo. |
| **Transformación logarítmica** | Reemplazar el valor por su logaritmo (base *e*, 2, 10…) comprime el rango de la feature y reduce las variaciones extremas. Ej.: log₁₀(1000) = 3 porque 10³ = 1000. |
| **Imputar** | Como con los faltantes: sustituir el outlier por la media o, mejor, por la **mediana** (la media está arrastrada por el propio outlier). Muy adecuado si el outlier es un error. |

Los outliers suelen generar distribuciones asimétricas, y la transformación logarítmica ayuda a volverlas más simétricas. En el ejemplo, al pasar el eje *y* a escala logarítmica, log₁₀(23) ≈ 1.36 frente a 0.67–1.10 del resto: el punto [25, 23] deja de desviarse tanto.

**Para el examen:** los servicios de AWS para detectar y tratar outliers son **Amazon SageMaker Data Wrangler** y **AWS Glue DataBrew**.

#### Elegir el método de detección: histogramas

Para elegir el algoritmo de detección se analiza el histograma de cada feature. Desde un notebook de SageMaker:

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

data = np.array([[4, 11], [3.8, 12], [4.5, 12.5], [8, 8], [9, 8.5],
                 [9.5, 7.5], [13, 5], [14, 4.7], [13.7, 6], [25, 23]])
df = pd.DataFrame(data, columns=['Feature1', 'Feature2'])

plt.figure(figsize=(12, 5))
plt.subplot(1, 2, 1)
sns.histplot(df['Feature1'], kde=True)
plt.title('Water Usage Distribution (m³/day)')
plt.subplot(1, 2, 2)
sns.histplot(df['Feature2'], kde=True)
plt.title('Energy Usage Distribution (kWh/day)')
plt.show()
```

Ambas features muestran una **distribución asimétrica a la derecha** (*right-skewed*): la mayoría de los valores se agrupan en la parte baja y una cola larga se extiende hacia la derecha con pocos valores muy altos. En este caso, esa cola es [25, 23].

Los dos snippets siguientes parten de ese mismo `df` original de dos columnas (el libro lo recrea en cada uno).

#### Método IQR (datos asimétricos)

Para datos asimétricos, el **rango intercuartílico (IQR)** es de los mejores métodos porque le afecta poco la asimetría:

1. Calcular Q1 (percentil 25) y Q3 (percentil 75).
2. Calcular IQR = Q3 − Q1.
3. Es outlier todo valor **< Q1 − 1.5·IQR** o **> Q3 + 1.5·IQR**.

Con el método `quantile` de los DataFrame de pandas:

```python
Q1 = df.quantile(0.25)
Q3 = df.quantile(0.75)
IQR = Q3 - Q1

outliers = ((df < (Q1 - 1.5 * IQR)) | (df > (Q3 + 1.5 * IQR))).any(axis=1)
df['Outlier'] = outliers
print(df)

# Tratar los outliers eliminándolos
df_cleaned = df[~df['Outlier']].drop(columns='Outlier')
print(df_cleaned)
```

El script marca **[25, 23]** como outlier y lo elimina. Lo detecta por Feature2 (23 > límite superior de 19.81); en Feature1, 25 queda justo por debajo de su límite (25.75). `any(axis=1)` marca la fila si **cualquier** columna se sale del rango.

#### Método Z-score (datos normales)

Para datos con distribución normal, el **Z-score** es muy eficaz:

1. Calcular la media μ y la desviación estándar σ.
2. Para cada punto x: **z = (x − μ) / σ**.
3. En una normal, ±1σ cubre ≈68 % de los datos; ±2σ, ≈95 %, y ±3σ, ≈99.7 %. Por eso los puntos con **|z| > 3** (el ≈0.3 % más alejado) se consideran outliers.

```python
from scipy import stats

z_scores = np.abs(stats.zscore(df))
outliers = (z_scores > 3).any(axis=1)
df['Outlier'] = outliers
print(df)

df_cleaned = df[~df['Outlier']].drop(columns='Outlier')
print(df_cleaned)
```

Aquí el Z-score **no detecta** [25, 23]. El libro lo atribuye a que la distribución no es normal sino asimétrica.

> **Corrección:** la asimetría influye, pero hay un motivo más fuerte: con n = 10 puntos, el umbral de 3 es **inalcanzable**. Con la desviación estándar poblacional (la que usa `stats.zscore`), |z| ≤ √(n − 1) = 3. De hecho, [25, 23] obtiene z ≈ 2.39 y 2.58. El Z-score con umbral 3 necesita datos aproximadamente normales **y** suficientes observaciones.

**AWS Glue DataBrew** también detecta outliers (con Z-score e IQR) y permite **reemplazarlos, eliminarlos, reescalarlos o marcarlos**.

### 2.3 Deduplicación

Eliminar puntos duplicados no es solo ordenar el dataset; mejora la precisión, el rendimiento y la fiabilidad del modelo:

- **Calidad de datos:** los duplicados distorsionan los resultados y llevan a predicciones imprecisas.
- **Rendimiento:** pueden causar **sobreajuste** (el modelo aprende ruido en lugar de la señal); sin ellos, el modelo generaliza mejor.
- **Fiabilidad:** si ciertas entradas se repiten más que otras, introducen sesgo y desbalancean los datos de entrenamiento.

Para eliminar duplicados, lo más sencillo es **AWS Glue DataBrew**: interfaz visual, sin código complejo.

### 2.4 Estandarización y reformateo

- **Estandarizar** evita que las features con valores grandes influyan desproporcionadamente frente a las de valores pequeños. Es especialmente importante en algoritmos sensibles a la escala, como la regresión lineal y las SVM (capítulo 4).
- **Reformatear** es reestructurar los datos en un formato coherente: convertir tipos, armonizar formatos y asegurar que todas las entradas siguen la misma estructura.

Flujo típico con SageMaker y AWS Glue:

1. **Cargar** el dataset en un notebook de SageMaker.
2. **Estandarizar** con `StandardScaler` de `sklearn.preprocessing` (hay más técnicas en la sección 3.1).
3. **Exportar** el dataset estandarizado a S3.
4. **Crear un job de AWS Glue** que reformatee los datos según haga falta.
5. **Entrenar** el modelo con los datos limpios y estandarizados.

```python
import pandas as pd
from sklearn.preprocessing import StandardScaler

data = pd.read_csv('your_dataset.csv')
scaler = StandardScaler()
data_scaled = scaler.fit_transform(data)
```

> **Corrección:** el libro dice que así se carga desde S3, pero la ruta es local; `read_csv` acepta una URI `s3://bucket/clave.csv` si está instalado `s3fs`. Además, `StandardScaler` solo admite columnas numéricas, y aplicarlo sobre todo el dataset contradice la regla de la sección 6: el scaler se ajusta **solo con el conjunto de entrenamiento** y después se aplica (`transform`) a validación y prueba.

### 2.5 Eliminación de ruido y errores

El objetivo es obtener datos de entrenamiento de la mejor **calidad** posible. Crear o transformar features hace los datos más informativos y mejora la precisión, la eficiencia y la generalización del modelo.

Un modelo entrenado con datos ruidosos aprende las peculiaridades o errores del dataset en lugar de los patrones subyacentes: es el **sobreajuste** (*overfitting*), con buen rendimiento en entrenamiento y malo con datos nuevos. Gestionar el ruido ayuda a que generalice. El capítulo 5 trata el sobreajuste en detalle.

---

## 3. Técnicas de feature engineering

Una vez limpios los datos, el siguiente paso es **transformar los datos en crudo en features significativas** que mejoren el rendimiento del modelo; después, el modelo queda listo para entrenar. Primero se ven las técnicas para datos estructurados y luego las de datos no estructurados.

### 3.1 Datos estructurados

Con datos estructurados puedes usar varias técnicas de extracción para reducir la dimensionalidad. Precaución: al llevar el modelo a producción o automatizar el pipeline, esas features deben poder **reproducirse exactamente igual** sobre los datos nuevos, sin perder la reducción de dimensionalidad.

#### Datos numéricos

Los datos numéricos son, en última instancia, lo que recibe el modelo: sean categóricos, texto o imagen, el resultado del feature engineering debe ser un conjunto de números bien organizado. Cómo se construyen esos números influye mucho en el rendimiento y la precisión.

##### Normalización

Lleva todas las features a una **escala común, normalmente [0, 1]**. Beneficios:

- **Peso equitativo:** ninguna feature domina por su escala. Es clave en algoritmos sensibles a la escala, como **k-NN** (calcula distancias entre puntos) y las **redes neuronales** (entrenadas con descenso de gradiente).
- **Convergencia más rápida:** con descenso de gradiente, todas las features contribuyen por igual al gradiente, lo que agiliza el entrenamiento.
- **Interpretabilidad:** la importancia de las features y los coeficientes son comparables.

Úsala cuando necesites una escala común pero no que los datos estén centrados en cero. Mejora la precisión y la estabilidad, sobre todo en algoritmos basados en distancias. Se implementa normalmente con **MinMax scaling** (ver "Escalado").

##### Estandarización

Transforma cada feature para que tenga **media 0 y desviación estándar 1**, usando el **Z-score**:

$$z = \frac{x - \mu}{\sigma}$$

donde *x* es el punto, μ la media y σ la desviación estándar de la feature. Restar μ centra los datos en 0; dividir entre σ los escala según su dispersión, de modo que la varianza (σ²) pasa a ser 1.

El libro la presenta como ideal para algoritmos que "asumen datos normales", como la regresión lineal, la logística o los que usan descenso de gradiente.

> **Corrección:** estandarizar **no vuelve normal** una distribución (solo la desplaza y reescala), y ni la regresión lineal ni la logística exigen features normales. Regla práctica: estandarización cuando los datos son aproximadamente normales o el algoritmo funciona mejor con datos centrados y de escala comparable (regresión lineal y logística, SVM, descenso de gradiente).

##### Escalado

**Escalado** es el término general para ajustar el rango o la distribución de las features. La normalización (centrada en el **rango**, 0 a 1) y la estandarización (centrada en la **distribución**, μ = 0 y σ = 1) son tipos de escalado.

Si los datos no son normales, con asimetría u outliers, hacen falta técnicas distintas del Z-score. Las que entran en el examen:

| Técnica | Qué hace | Cuándo usarla | Clase de `sklearn.preprocessing` |
|---|---|---|---|
| **Robust scaling** | Resta la **mediana** y divide entre el **IQR**. | Datos con outliers: la mediana y el IQR no se dejan arrastrar por ellos. | `RobustScaler` |
| **MinMax scaling** | Lleva cada feature a un rango fijo, normalmente [0, 1]. Es la técnica de **normalización**, no de estandarización (el libro la llama "de estandarización"). | Escala común. Conserva la forma de la distribución de cada feature: **no corrige la asimetría** y es sensible a outliers. | `MinMaxScaler` |
| **MaxAbs scaling** | Divide cada valor entre el **máximo valor absoluto** de la feature → rango [−1, 1]. | Datos **dispersos** (muchos ceros): no desplaza los datos, así que conserva los ceros y la distribución original. | `MaxAbsScaler` |
| **Transformaciones de potencia** (Box-Cox, Yeo-Johnson) | Estabilizan la varianza y acercan los datos a una normal. | Datos asimétricos. **Box-Cox** solo admite valores estrictamente positivos; **Yeo-Johnson**, positivos, negativos y cero. | `PowerTransformer` |

Recordatorio: la **varianza** (σ²) es el promedio de las diferencias al cuadrado respecto a la media; nunca es negativa y mide la dispersión (alta = datos alejados de la media; baja = cercanos).

**Robust scaling** sobre el `df` original de dos columnas de la sección 2.2 (el dataset de outliers):

```python
from sklearn.preprocessing import RobustScaler

scaler = RobustScaler()
data_scaled = scaler.fit_transform(df)
df_scaled = pd.DataFrame(data_scaled, columns=['Feature1', 'Feature2'])
print(df_scaled)
```

> **Corrección:** el libro afirma que, tras el robust scaling, outliers como [25, 23] "ya no sesgan el dataset". En realidad el outlier **sigue siendo extremo** (queda en 1.93 y 2.74, frente a valores entre −0.67 y 0.79 del resto) y la asimetría no cambia, porque es una transformación lineal. Lo que garantiza es que el outlier **no distorsiona la escala del resto**. En cambio, MinMax sobre este mismo dataset comprime los otros 9 puntos de Feature1 en [0, 0.48], porque el outlier fija el máximo.

**MinMax, MaxAbs y Yeo-Johnson** (el libro repite el mismo snippet tres veces cambiando solo la clase):

```python
import numpy as np
import pandas as pd
from sklearn.preprocessing import MinMaxScaler, MaxAbsScaler, PowerTransformer

data = np.array([[1, 2], [3, 4], [5, 6], [7, 8], [-1, -2], [-3, -4]])
df = pd.DataFrame(data, columns=['Feature1', 'Feature2'])

for scaler in [MinMaxScaler(), MaxAbsScaler(), PowerTransformer(method='yeo-johnson')]:
    df_scaled = pd.DataFrame(scaler.fit_transform(df), columns=['Feature1', 'Feature2'])
    print(type(scaler).__name__, "\n", df_scaled)

# Feature1 → MinMax: 0.4, 0.6, 0.8, 1.0, 0.2, 0.0
#            MaxAbs: 0.14, 0.43, 0.71, 1.0, -0.14, -0.43
#            Yeo-Johnson: valores centrados en 0 (PowerTransformer además estandariza por defecto)
```

Con estos datos (hay negativos), `PowerTransformer(method='box-cox')` falla: por eso se usa Yeo-Johnson. Las transformaciones de potencia reducen la asimetría y estabilizan la varianza, lo que da modelos más robustos y precisos.

##### Transformación logarítmica

Además de tratar outliers, es una técnica de feature engineering para datos numéricos:

- **Reduce la asimetría:** muchos datos reales son asimétricos a la derecha (p. ej., ingresos, con pocos muy altos; o notas de un examen difícil, con pocos alumnos arriba). El logaritmo comprime la cola y acerca la distribución a una normal.
- **Limita el impacto de los outliers:** los acerca al grueso de los datos sin eliminarlos.
- **Linealiza relaciones:** si la relación con el target es multiplicativa o exponencial, el logaritmo la vuelve lineal y fácil de capturar para modelos lineales.

Ejemplo: `[[3, 1], [4, 10], [5, 100], [6, 1000]]`, aplicando log₁₀ a *y*, se convierte en `[[3, 0], [4, 1], [5, 2], [6, 3]]`, una relación lineal.

> ⚠️ El logaritmo **no se aplica a 0 ni a negativos** (está indefinido en cualquier base). En ese caso, desplaza los datos para que sean positivos o usa otra transformación, como la raíz cúbica.

##### Raíz cuadrada o cúbica

Sirven para reducir la varianza alta (además de las transformaciones de potencia). Modifican la distribución, pero **menos que el logaritmo**. La **raíz cúbica** admite negativos y cero; la **cuadrada**, solo positivos y cero.

##### Binning

El **binning** (*bucketing*) divide datos numéricos continuos en **intervalos discretos** (*bins*): convierte datos numéricos en categóricos. Simplifica el modelo y lo hace más robusto a outliers y ruido. Ejemplo: agrupar el peso de los vehículos en cinco categorías, que luego se representan como cinco columnas: `is_minicompact`, `is_subcompact`, `is_compact`, `is_midsize` e `is_large`.

#### Datos categóricos

Las features categóricas capturan relaciones y características que los números pueden no reflejar. El primer paso es la **codificación** (*encoding*): transformar las cadenas en valores numéricos (un entero, un array, una matriz o un tensor de enteros), porque la mayoría de los algoritmos solo entienden números. Ej.: White, Black y Red pueden codificarse como `[1, 0, 0]`, `[0, 1, 0]` y `[0, 0, 1]`.

##### Label encoding

Asigna un **entero único a cada categoría**: `["White", "Black", "Red"]` → `[0, 1, 2]`.

- **Problema:** impone un orden artificial, lo que confunde a los algoritmos que interpretan los números como ordenados.
- **Mejor uso:** algoritmos **basados en árboles**, que manejan bien ese orden implícito, y datos **ordinales** (asignando los enteros según el orden real: S = 0, M = 1, L = 2).

##### One-hot encoding

Convierte una feature categórica en un conjunto de **features binarias, una por categoría**: 1 si la categoría está presente y 0 si no.

```python
import pandas as pd
from sklearn.preprocessing import OneHotEncoder

data = {'Color': ['White', 'Black', 'Red', 'Blue', 'Green',
                  'Yellow', 'Pink', 'Brown', 'White', 'Black']}
df = pd.DataFrame(data)

encoder = OneHotEncoder(sparse_output=False)
one_hot = encoder.fit_transform(df[['Color']])
one_hot_df = pd.DataFrame(one_hot, columns=encoder.get_feature_names_out(['Color']))
print(one_hot_df)
```

El dataset tiene 10 filas con **8 colores distintos**, así que el resultado tiene **8 columnas binarias** (`Color_Black`, `Color_Blue`, `Color_Brown`, `Color_Green`, `Color_Pink`, `Color_Red`, `Color_White`, `Color_Yellow`), cada una con 10 valores.

- **Mejor uso:** features con **pocas categorías** y algoritmos **no basados en árboles** (regresión lineal, k-NN, redes neuronales; capítulo 4).
- **Problema:** **aumenta la dimensionalidad**, sobre todo con features de **alta cardinalidad** (muchas categorías únicas); el impacto depende del algoritmo y de los datos. Se mitiga con **binary encoding** o aplicando **PCA** después del one-hot (capítulo 4).

##### Binary encoding

Evita la "explosión" de dimensionalidad del one-hot: convierte cada categoría en un número, lo expresa en **binario** y separa **cada bit en una columna**. Es una forma de compactar el one-hot. Con la librería externa `category_encoders`, sobre el mismo `df`:

```python
import category_encoders as ce

encoder = ce.BinaryEncoder(cols=['Color'])
df_binary_encoded = encoder.fit_transform(df)
print(df_binary_encoded)
```

Resultado: **4 columnas en lugar de 8**; el número de filas (10) no cambia.

> **Corrección:** el libro explica que "8 categorías necesitan 4 bits", pero con 3 bits se representan 8 valores (0–7). Salen 4 porque `category_encoders` numera las categorías **desde 1** (White = 1 … Brown = 8) y 8 = 1000₂ requiere 4 bits. En general, binary encoding usa del orden de log₂(k) columnas frente a las k del one-hot.

- **Mejor uso:** reducir la dimensionalidad que genera el one-hot.
- **Desventaja:** las categorías comparten bits (p. ej., White = 0001 y Green = 0101 coinciden en el último), así que el modelo puede ver parecidos que no existen y distinguir peor las categorías, lo que puede afectar su rendimiento.

##### Feature hashing

Usa una **función hash** para convertir datos categóricos de **alta cardinalidad** en un **número fijo de features** que eliges tú. Como el resultado es un vector de tamaño fijo, es eficiente, escalable y controla el uso de memoria. Funcionamiento:

1. **Entrada:** un valor de la feature (una palabra, una categoría…).
2. **Hash:** la función devuelve un entero determinista para ese valor.
3. **Módulo:** el hash se reduce módulo el tamaño del vector para caer en su rango.
4. **Actualización:** ese índice indica qué posición del vector se actualiza.

> **Corrección:** el libro dice que el hash devuelve un valor "único" y que el módulo es "opcional". El hash es **determinista pero no único**: valores distintos pueden **colisionar** en la misma posición. En la práctica el módulo siempre se aplica (así se fija el tamaño del vector).

Las colisiones se mitigan usando suficientes posiciones (*buckets*). Así, el feature hashing equilibra la preservación de información con la eficiencia computacional y es una solución **eficiente y rentable para alta cardinalidad**.

```python
import pandas as pd
from sklearn.feature_extraction import FeatureHasher

colors = ['White', 'Black', 'Red', 'Blue', 'Green', 'Yellow', 'Pink', 'Brown', 'White', 'Black']
data = [{'Color': c} for c in colors]

hasher = FeatureHasher(n_features=3, input_type='dict')
hashed_features = hasher.transform(data)          # FeatureHasher no necesita fit
hashed_df = pd.DataFrame(hashed_features.toarray(),
                         columns=[f'feature_{i}' for i in range(hashed_features.shape[1])])
print(hashed_df)
```

Cada una de las 10 filas se convierte en un vector de **3 features** (fijado con `n_features=3`). Los valores son −1, 0 o 1, no enteros arbitrarios, porque scikit-learn también usa el hash para decidir el signo. Con 8 categorías y solo 3 posiciones, **las colisiones son inevitables**: en esta ejecución, White y Blue producen el mismo vector, igual que Green, Pink y Brown, y el modelo no podría distinguirlas. En la práctica, `n_features` debe ser mucho mayor que el número de categorías.

#### Series temporales

Una serie temporal se captura repetidamente en el tiempo y **cada punto depende de sus valores pasados** (como las escenas de una película), por eso la dimensión temporal es clave.

**SageMaker Data Wrangler** ofrece una solución *low-code* para limpiar, transformar y preparar series temporales según el formato que requiere el modelo de pronóstico. Primero se entienden los patrones del dataset con sus visualizaciones, que orientan la estrategia de modelado; después se crean features que mejoren la precisión del pronóstico. Transformaciones de Data Wrangler:

| Transformación | Qué hace |
|---|---|
| **Featurize datetime** | Buena práctica para empezar: descompone el *timestamp* en features como `date_month`, `date_day`, `date_week_of_year`, `date_day_of_year` y `date_quarter`, lo que permite detectar patrones por componente. |
| **Encode categorical** (*One-hot encode*) | Trata componentes de fecha como categóricos: p. ej., `date_quarter` (4 valores posibles) se convierte en 4 columnas binarias, una por trimestre. |
| **Lag features** | Buena práctica sobre el **target**: valores en *timestamps* anteriores, útiles para predecir valores futuros. Ayudan a identificar la **autocorrelación** (correlación de la serie con sus propios valores pasados). Crea varios lags dentro de una ventana. |
| **Rolling window features** | Calcula propiedades estadísticas sobre un conjunto de observaciones definido por el tamaño de ventana. Data Wrangler automatiza esta extracción con el paquete open source **tsfresh**, sin programar librerías de procesamiento de señales. |

El dataset transformado ya puede usarse como entrada de un algoritmo de pronóstico.

### 3.2 Datos no estructurados

Imágenes, texto, audio y vídeo no encajan en tablas ni tienen formato predefinido. Para el examen interesan las técnicas de **imagen** y **texto**.

#### Imágenes

Se extraen features significativas (bordes, texturas, colores) para mejorar tareas de reconocimiento y clasificación, como en el ejemplo del coche de la sección 1.3. El flujo en AWS:

1. **Extraer features:**
   - **SageMaker JumpStart:** la vía principal. Modelos de visión preentrenados (detección de objetos, clasificación de imágenes…), normalmente pasando las imágenes por una **CNN** y obteniendo los vectores de features resultantes.
   - **Amazon Rekognition:** análisis básico según el caso de uso. Detecta objetos, escenas, texto y caras, y devuelve features de alto nivel como etiquetas y *bounding boxes*, útiles para detección de objetos y clasificación.
2. **Transformar:** con **SageMaker Data Wrangler** se hace análisis exploratorio (EDA) y transformaciones como normalizar los valores de píxel a una escala común o reducir la dimensionalidad con PCA.
3. **Almacenar:** en **SageMaker Feature Store**, para compartir y reutilizar features entre modelos y proyectos sin volver a extraerlas.

#### Texto

- **Amazon Comprehend:** extrae features de alto nivel: entidades, frases clave, sentimiento e idioma.
- **Amazon Textract:** extrae texto de documentos y convierte datos no estructurados en información estructurada.
- **Algoritmos y frameworks de SageMaker** (capítulo 4): extracción más personalizada, como *word embeddings* (Word2Vec) o modelado de temas (Latent Dirichlet Allocation).

Técnicas que entran en el examen:

| Técnica | Qué hace | Ejemplo |
|---|---|---|
| **Tokenización** | Divide el texto en unidades más pequeñas (**tokens**). Es el primer paso para preparar texto. | "Machine learning is fascinating" → ["Machine", "learning", "is", "fascinating"] |
| **Eliminación de stop words** | Quita palabras comunes que no aportan significado ("the", "is", "and"); reduce el ruido. | Quitar "the" e "is" de "The cat is on the mat" → ["cat", "on", "mat"] |
| **Stemming y lematización** | Reducen las palabras a su forma base. El *stemming* recorta prefijos/sufijos con heurísticas; la lematización usa reglas lingüísticas y devuelve una palabra real (el lema). | "running" → "run" |
| **N-gramas** | Secuencias contiguas de *n* elementos; capturan contexto y relaciones entre palabras. | Bigramas de "Machine learning is fun" → (Machine, learning), (learning, is), (is, fun) |
| **Word embeddings** | Word2Vec, GloVe: convierten palabras en vectores continuos que capturan su significado semántico. | "King" y "Queen" quedan cerca en el espacio vectorial |

**Tokenización y Amazon Bedrock.** Los modelos fundacionales (FM) de Bedrock tokenizan el prompt para interpretar su significado y contexto. La tokenización también define el **precio**: en los modelos de texto bajo demanda se paga por **tokens de entrada procesados y tokens de salida generados**, así que un prompt más largo cuesta más. El capítulo 4 incluye un ejemplo de generación de imágenes con un FM de Bedrock.

> **Corrección:** los tokens de un FM suelen ser **fragmentos de palabra** (*subwords*), no palabras y símbolos completos. Además, el libro ilustra el cobro por tokens con un prompt de generación de imágenes, pero los modelos de imagen de Bedrock se cobran **por imagen generada**, no por tokens (y el *Provisioned Throughput* se cobra por hora).

---

## 4. Etiquetado de datos

El etiquetado y el feature engineering están entrelazados: ambos convierten datos en crudo en una forma que el algoritmo puede aprender. El feature engineering selecciona, modifica y crea features; el **etiquetado** enriquece los datos con **etiquetas**, los valores reales del target (*ground truth*).

> **Corrección:** el libro describe las etiquetas como "las predicciones reales". Una etiqueta es el **valor verdadero** que el modelo debe aprender a predecir, no una predicción.

En aprendizaje supervisado este paso es fundamental: las etiquetas son las "respuestas correctas" con las que el modelo aprende y definen qué debe predecir o clasificar. Ej.: etiquetar cada imagen como "gato" o "perro" permite que el modelo aprenda a distinguirlos.

### Amazon SageMaker Ground Truth

Servicio diseñado para etiquetar datos que combina **automatización** con **intervención humana** (*human-in-the-loop*). Proceso:

1. **Almacenar los datos** en crudo (imágenes, texto…) en S3, organizados en carpetas o buckets por tipo o proyecto.
2. **Crear un job de etiquetado:** elegir el tipo de tarea (clasificación de imágenes, clasificación de texto, detección de objetos…), indicar el dataset de entrada en S3 y definir las instrucciones.
3. **Etiquetado automático** (*Automated Data Labeling*, opcional) y configuración de sus parámetros.
4. **Etiquetado y revisión humana:** la fuerza laboral puede ser **Amazon Mechanical Turk** (marketplace de *crowdsourcing* con trabajadores de todo el mundo para tareas difíciles para una computadora), una **fuerza laboral privada** o **proveedores externos**. Los anotadores usan la interfaz de Ground Truth para añadir o corregir etiquetas, *bounding boxes* u otras anotaciones. Control de calidad: **consenso** (varios anotadores etiquetan el mismo dato) y **auditoría** (expertos revisan una muestra).
5. **Guardar los datos etiquetados** en S3, con versionado para rastrear cambios.
6. **Entrenar** el modelo en SageMaker.

> **Corrección (paso 3):** el etiquetado automático no usa "modelos preentrenados". Usa **aprendizaje activo**: Ground Truth entrena un modelo con una parte etiquetada por humanos, etiqueta automáticamente los datos en los que ese modelo supera un **umbral de confianza** y envía el resto a humanos.
>
> **Corrección (paso 5):** Ground Truth escribe los resultados en **S3** (un *output manifest*); no los guarda "automáticamente" en Feature Store. Para usarlos ahí hay que ingerirlos en un paso aparte.

---

## 5. Desbalance de clases

Tras limpiar, hacer feature engineering y etiquetar, el dataset puede estar bien sintácticamente pero no semánticamente, por una **distribución desigual de las etiquetas**. Ej.: en un caso de salud, la mayoría de las muestras son personas sanas (clase mayoritaria) y pocas tienen una enfermedad rara (clase minoritaria), justo la que se quiere predecir. Eso es **desbalance de clases**.

El modelo resultante queda **sesgado hacia la clase mayoritaria**, lo que afecta su rendimiento y su equidad, sobre todo cuando la clase minoritaria es la crítica.

### Técnicas de mitigación

La **aumentación de datos** es la buena práctica general: genera más ejemplos de la clase minoritaria para cerrar la brecha. En imágenes, con rotaciones, volteos o ajustes de color; en texto, con sustitución de sinónimos o inserción aleatoria.

Si no es viable por coste, recursos de cómputo o tiempo:

| Técnica | Qué hace | Riesgo / nota |
|---|---|---|
| **Oversampling** | Aumenta la clase minoritaria duplicando ejemplos o generando sintéticos. **SMOTE** crea muestras sintéticas **interpolando** entre ejemplos minoritarios existentes. | SMOTE equilibra sin limitarse a duplicar registros. |
| **Undersampling** | Elimina aleatoriamente ejemplos de la clase mayoritaria. | Puede perder información valiosa. |
| **Ponderación de clases** (*class weighting*) | Al calcular la pérdida, da más peso a los errores sobre la clase minoritaria. | Obliga al modelo a prestarle más atención. |

**Aumentación vs. datos sintéticos:** la aumentación crea datos nuevos modificando los ejemplos reales de entrenamiento; los datos sintéticos se generan en lugar de derivarse de registros concretos (p. ej., con simuladores o modelos generativos). Matiz: las muestras de SMOTE se llaman "sintéticas" aunque se obtienen interpolando ejemplos reales.

### Amazon SageMaker Clarify

Clarify calcula **métricas de sesgo previas al entrenamiento** (*pre-training bias metrics*). Cada métrica corresponde a una noción distinta de equidad.

> **Corrección:** Clarify **detecta y mide** el sesgo; no lo mitiga. La mitigación (resampling, ponderación…) la decides y aplicas tú.

Conceptos:

- **Faceta** (*facet*): la feature (atributo sensible, p. ej., edad o género) que se analiza para detectar sesgo. **Faceta a** es el valor favorecido por el sesgo y **faceta d**, el desfavorecido.
- Todas las métricas comparten que **0 (o cerca de 0) indica que no hay sesgo**.

Las dos métricas más usadas:

| Métrica | Qué mide | Interpretación |
|---|---|---|
| **Class Imbalance (CI)** | Si una faceta está infrarrepresentada: (nₐ − n_d) / (nₐ + n_d), con nₐ y n_d = número de muestras de cada faceta. | Rango [−1, +1]. 0 = mismo número de muestras. Valores positivos = la faceta a tiene más muestras; negativos = la faceta d. Cerca de ±1 = muy desbalanceado. |
| **Difference in Proportions of Labels (DPL)** | Diferencia en la proporción de resultados positivos entre la faceta a y la faceta d. | Rango [−1, +1] para etiquetas binarias o multicategoría y (−∞, +∞) para continuas. 0 = igual proporción. +1 = la faceta a tiene mayor proporción de positivos; −1 = la faceta d. |

> **Corrección:** la tabla 3.1 del libro añade *Equal opportunity difference* (EOD) y *Predictive parity difference* (PPD) con descripciones sin sentido ("sesgo de detección de objetos", "sesgo de clasificación de imágenes"). No son métricas previas al entrenamiento de Clarify. Las métricas pre-training son **CI, DPL, KL, JS, LP, TVD, KS y CDD** (https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-measure-data-bias.html). Además, el libro interpreta CI como desbalance "hacia la clase mayoritaria/minoritaria". En realidad mide el desbalance entre **facetas**, no entre clases de la etiqueta.

**Ejemplo del libro (corregido).** En un dataset de transacciones con tarjeta, el 99.9 % tiene `is_fraudulent = 0`. El libro usa `is_fraudulent` como faceta, obtiene CI ≈ 1 y concluye que "hay que hacer undersampling". Dos problemas:

1. `is_fraudulent` es la **etiqueta**, no una faceta. Ese 99.9/0.1 es un desbalance de clases que se ve directamente en la distribución de la etiqueta. CI sirve para ver si un **grupo** (p. ej., un rango de edad) está infrarrepresentado.
2. Ninguna métrica decide la técnica. Con un 0.1 % de fraudes, hacer undersampling hasta equilibrar descartaría casi todos los datos; se elige entre oversampling/SMOTE, undersampling o ponderación según el volumen de datos y el coste.

Con las métricas calculadas se decide el siguiente paso para preparar el dataset. En el SDK de SageMaker, la clase **`BiasConfig`** (`sagemaker.clarify` en el SDK v2; `sagemaker.core.clarify` en v3) configura el análisis de sesgo: etiqueta, faceta y sus valores.

---

## 6. División de datos

Con el desbalance gestionado, el dataset está listo para dividirse en entrenamiento, validación y prueba. Es clave para entrenar el modelo de forma eficaz y evaluarlo con precisión.

| Conjunto | Para qué sirve | Analogía (preparar un examen) | Proporción típica |
|---|---|---|---|
| **Entrenamiento** | El modelo aprende patrones y relaciones, y ajusta sus parámetros para minimizar errores. Ej.: muchos emails etiquetados como spam o no spam. | El material de estudio | 60–80 % |
| **Validación** | Ajustar los **hiperparámetros** y detectar el sobreajuste, comprobando que el modelo generaliza a datos no vistos. | Los simulacros: indican qué mejorar sin ser parte del estudio | 10–20 % |
| **Prueba** | Evaluación final e **imparcial** con datos que el modelo **nunca ha visto**. Ej.: un conjunto aparte de emails. | El examen real, supervisado y con preguntas nuevas | 10–20 % |

En resumen: el entrenamiento construye el modelo, la validación lo ajusta y previene el sobreajuste, y la prueba mide objetivamente su capacidad de generalizar. Una partición correcta evita la fuga de datos y el sobreajuste, y da una medida fiable del rendimiento real.

### Fuga de datos (data leakage)

Hay **fuga de datos** cuando el modelo accede durante el entrenamiento a información que no tendría en un escenario real. El caso típico es que datos de prueba "se filtren" al conjunto de entrenamiento. Resultado: métricas infladas artificialmente y mala generalización.

Cómo evitarla:

- Mantener una separación clara entre entrenamiento y prueba.
- Vigilar el preprocesamiento: **normalizar/estandarizar después de dividir**, ajustando el scaler solo con entrenamiento.
- Si se detecta fuga, reevaluar el modelo con datos correctamente separados.

> **Corrección:** el libro cita la validación cruzada como técnica contra la fuga. Por sí sola no la evita: si el preprocesamiento se ajusta con todos los datos antes de hacer los *folds*, la fuga sigue ahí.

### División con SageMaker Data Wrangler

Además de ser la herramienta central para el feature engineering, **Data Wrangler** divide el dataset en entrenamiento, validación y prueba con poco o ningún código, mediante la transformación **Split data**:

| Tipo de split | Qué hace | Cuándo usarlo |
|---|---|---|
| **Random split** | Reparte las filas al azar entre los subconjuntos. | Cuando no hace falta conservar el orden de los datos. |
| **Ordered split** | Divide respetando el orden, de modo que la información pasada y futura no se mezcla entre subconjuntos. | Series temporales o cualquier caso en que el orden sea crítico. |
| **Stratified split** | Cada subconjunto mantiene la **misma proporción de clases** que el dataset original. | Clasificación con datos desbalanceados: evaluación y entrenamiento más fiables. |
| **Split by key** | Ninguna combinación de valores de las columnas clave aparece en más de un split. | Evitar la fuga en datos no ordenados y mantener juntos los registros relacionados. |

> **Corrección:** el libro atribuye al *random split* que "cada subconjunto tiene una distribución de categorías similar". No lo garantiza: eso solo lo asegura el **stratified split**.

Después se indican los porcentajes (p. ej., 70 % entrenamiento, 20 % validación y 10 % prueba) y Data Wrangler divide el dataset automáticamente.

---

## Resumen

- Preparar los datos consiste en convertir el dataset en crudo en un conjunto de features significativas, en un formato que el algoritmo pueda entender, para obtener predicciones precisas y fiables.
- Los **tipos de datos** (categórico, numérico, texto, imagen, serie temporal) determinan las técnicas de transformación.
- La **selección y extracción de features** reducen la dimensionalidad y dejan solo lo relevante para el modelo.
- La **limpieza** abarca valores faltantes, outliers, duplicados, formato y ruido.
- Se vieron técnicas de feature engineering para datos numéricos, categóricos, series temporales y no estructurados.
- **Data Wrangler** es la herramienta para hacer feature engineering y **Feature Store**, para almacenar, compartir y gestionar las features.
- **Ground Truth** etiqueta los datos, **Clarify** mide el sesgo y el desbalance antes del entrenamiento, y **Data Wrangler** divide el dataset en entrenamiento, validación y prueba.

Los próximos capítulos tratan cómo elegir el enfoque de modelado, entrenar y refinar el modelo y evaluar su rendimiento.

---

## Puntos clave para el examen

**Outliers.** Eliminarlos (cuando sea posible), transformarlos (logaritmo, que además reduce la asimetría) o imputarlos con la mediana o la media para conservar la integridad del dataset.

**Asimetría (skewness):**

| Corrigen la asimetría | No la corrigen |
|---|---|
| Transformación logarítmica, raíz cuadrada (o cúbica), Box-Cox, Yeo-Johnson | **Z-score** (media 0 y σ 1, pero si los datos eran asimétricos siguen siéndolo) y **MinMax** (cambia el rango, no la forma). Tampoco robust scaling ni MaxAbs, que también son lineales. |

**Normalización vs. estandarización:**

| | Normalización | Estandarización |
|---|---|---|
| Qué importa | La **escala/rango** | La **distribución** |
| Resultado | Todas las features en el mismo rango (normalmente 0–1) | Media 0 y desviación estándar 1 |
| Técnica | **MinMax scaling** | **Z-score**, para datos (aprox.) normales |

**Datos categóricos:**

| Técnica | Cuándo |
|---|---|
| **Label encoding** | Datos ordinales y modelos basados en árboles |
| **One-hot encoding** | Pocas categorías, datos nominales, modelos no basados en árboles |
| **Binary encoding** | Reducir la dimensionalidad que genera el one-hot |
| **Feature hashing** | Alta cardinalidad, cuando hay que optimizar recursos y la solución debe ser rentable |

**Imágenes.** Extraer features con **SageMaker JumpStart** (o **Rekognition** para análisis básico) → transformar con **Data Wrangler** (normalización, reducción de dimensionalidad) → almacenar en **Feature Store**.

**Texto.** Extraer features con **Comprehend** o **Textract**. Para extracción personalizada en SageMaker: tokenización, stemming y lematización para seleccionar palabras o reducirlas a su raíz antes del análisis semántico.

**Etiquetado.** **SageMaker Ground Truth** enriquece el dataset con las etiquetas (valores reales del target) de las que aprende el modelo.

**Desbalance de clases.** Elegir la feature sensible al sesgo (faceta), calcular las métricas pre-training de **Clarify** (CI, DPL) y decidir la estrategia: oversampling, undersampling o ponderación de clases.

**Entrenamiento, validación y prueba.** Con el entrenamiento el modelo aprende (su "educación"). La validación, tras entrenar, sirve para ajustar hiperparámetros y mejorar la generalización (el "simulacro"). La prueba es la evaluación final e imparcial con datos nuevos (el "examen final").

---

## Preguntas de repaso

**1.** Preparas un dataset con features numéricas, categóricas y ordinales. Para entrenar un modelo predictivo y aumentar su precisión, debes transformar las features categóricas en valores numéricos. ¿Qué solución es la más adecuada?
A. One-hot encoding · B. Feature scaling · C. Feature extraction · D. Date formatting

**2.** Quieres convertir una columna de tus datos de entrenamiento en valores binarios. ¿Qué técnica es la más adecuada?
A. One-hot encoding · B. Tokenización · C. Label encoding · D. Feature hashing

**3.** Durante la preparación descubres valores faltantes en algunas columnas de un dataset con features categóricas. Debes asegurarte de que esto no distorsione los datos ni reduzca la fiabilidad del modelo. ¿Qué solución es la más adecuada?
A. Amazon SageMaker Clarify · B. Imputación múltiple · C. Eliminar la feature · D. Recolectar los datos

**4.** Durante el análisis observas variables de entrada con rangos muy distintos. Quieres evitar que las features con valores grandes influyan demasiado en la capacidad predictiva del modelo. ¿Qué transformación es la más adecuada?
A. Normalización · B. Estandarización · C. Binning · D. One-hot encoding

**5.** Tu dataset tiene varias features categóricas, todas de alta cardinalidad. Quieres hacer feature engineering de forma eficiente y rentable. ¿Qué solución encaja mejor?
A. Label encoding · B. Lag features · C. Binary encoding · D. Feature hashing

**6.** Un modelo entrenado para reconocer coches no rinde bien. Debes rehacer las features del dataset de imágenes. ¿Qué solución encaja mejor?
A. Tokenización con Data Wrangler · B. Lag features con Data Wrangler · C. Binning con Data Wrangler · D. Amazon SageMaker JumpStart

**7.** ¿Qué técnica avanzada de feature engineering para texto convierte palabras en vectores numéricos que capturan su significado semántico?
A. One-hot encoding · B. Tokenización · C. Word embeddings · D. Normalización

**8.** ¿Cuál es la técnica más común para gestionar el desbalance de clases?
A. Codificación de datos · B. Aumentación de datos · C. Escalado de features · D. División de datos

**9.** ¿Qué servicio de AWS se usa para etiquetar datos?
A. Amazon Comprehend · B. Amazon SageMaker Ground Truth · C. Amazon Rekognition · D. Amazon SageMaker Clarify

**10.** ¿Para qué se divide un dataset en entrenamiento, validación y prueba?
A. Para mejorar la precisión del modelo · B. Para que el modelo tenga datos diversos · C. Para prevenir el sobreajuste y evaluar el rendimiento del modelo · D. Para simplificar el procesamiento de datos

### Respuestas

| # | Resp. | Justificación |
|---|---|---|
| 1 | **A** | Convierte categorías en columnas numéricas binarias; las demás opciones no codifican categorías. |
| 2 | **A** | One-hot genera una columna binaria (0/1) por categoría. |
| 3 | **B** | La imputación múltiple (no explicada en el capítulo) genera varios valores plausibles por faltante y los combina, así que no distorsiona los datos. Eliminar o recolectar son opciones para cuando faltan muchos datos, y Clarify no trata faltantes. |
| 4 | **A** (más probable) | Normalizar lleva todas las features al mismo rango. Pregunta ambigua: la sección 2.4 atribuye ese mismo efecto a la estandarización, así que B también es defendible. |
| 5 | **D** | Feature hashing = alta cardinalidad con eficiencia y bajo coste (binary encoding reduce la dimensionalidad, pero no es la respuesta "rentable" del capítulo). |
| 6 | **D** | JumpStart ofrece modelos de visión preentrenados para extraer features de imágenes; las demás opciones son técnicas para texto, series temporales o datos numéricos. |
| 7 | **C** | Los word embeddings (Word2Vec, GloVe) capturan el significado semántico. |
| 8 | **B** | La aumentación de datos es la práctica recomendada en el capítulo. |
| 9 | **B** | Ground Truth es el servicio de etiquetado. |
| 10 | **C** | Entrenamiento para aprender, validación para ajustar y detectar sobreajuste, prueba para la evaluación imparcial. |
