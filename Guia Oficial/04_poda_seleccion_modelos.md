# Capítulo 4 — Selección de modelos

> **Objetivos del examen MLA-C01 cubiertos**
> Dominio 2: Desarrollo de modelos de ML — 2.1 Elegir un enfoque de modelado · 2.2 Entrenar y refinar modelos

> **Nota de vigencia (septiembre de 2026)**
> - Los snippets de SageMaker usan el **SageMaker Python SDK v2** (`sagemaker.estimator.Estimator`, `sagemaker.image_uris`, `sagemaker.Session`, `get_execution_role`). En el SDK v3 esos módulos ya no existen, así que para ejecutarlos necesitas un entorno con `pip install "sagemaker>=2,<3"`.
> - Amazon SageMaker pasó a llamarse **Amazon SageMaker AI** (diciembre de 2024). En estas notas, «SageMaker» se refiere siempre a SageMaker AI.
> - Desde octubre de 2025, Bedrock **habilita por defecto** todos los modelos *serverless*: se retiró la página *Model access* y ya no hay que «solicitar acceso» al modelo. Los modelos de Anthropic solo piden rellenar una vez un formulario de uso. El acceso se restringe con IAM y SCP.

---

## Introducción

Este capítulo trata la fase del ciclo de vida de ML (Fig. 4.1) en la que se **elige el algoritmo de ML o el servicio de IA** adecuado para el problema. La elección depende del caso de uso, de la naturaleza de los datos y del tipo de problema (clasificación, regresión, *clustering*…). Además, obliga a equilibrar **interpretabilidad, exactitud, eficiencia computacional, escalabilidad y coste**.

El desarrollo de modelos es **iterativo**: se elige un algoritmo, se entrena, se evalúa y se refina. Aquí se ven la selección y los pasos básicos para alimentar el algoritmo con datos. El ajuste de hiperparámetros y las técnicas contra el sobreajuste y el subajuste se tratan en el capítulo 5.

---

## 1. Servicios de IA de AWS

Los servicios de IA de AWS son **modelos preentrenados y totalmente gestionados** que se consumen mediante APIs. Permiten integrar visión, NLP, voz, recomendaciones o IA generativa en una aplicación **sin conocimientos profundos de ML**. Escalan automáticamente para grandes volúmenes de datos y se integran con el resto de AWS para construir flujos completos, desde la ingesta hasta el despliegue.

Para el examen hay que saber **qué servicio resuelve cada caso de uso**:

| Necesidad | Servicio |
|---|---|
| Análisis de imágenes y vídeo (objetos, rostros, texto, escenas) | **Amazon Rekognition** |
| Extracción de texto, formularios y tablas de documentos escaneados | **Amazon Textract** |
| Texto → voz (TTS) | **Amazon Polly** |
| Voz → texto (ASR) | **Amazon Transcribe** |
| Traducción automática | **Amazon Translate** |
| Análisis de texto: sentimiento, entidades, frases clave | **Amazon Comprehend** |
| Chatbots e interfaces conversacionales | **Amazon Lex** |
| Recomendaciones personalizadas | **Amazon Personalize** |
| IA generativa con modelos fundacionales (FMs) | **Amazon Bedrock** |

### 1.1 Visión

#### Amazon Rekognition

Servicio de análisis de **imágenes y vídeo** basado en *deep learning*. Identifica objetos, personas, texto, escenas y actividades. Su función más destacada es el **análisis facial**: detección, comparación y reconocimiento de rostros, tanto en tiempo real como en contenido almacenado. También detecta texto dentro de imágenes y vídeos. Se usa mediante una API sencilla, sin experiencia en ML, y escala desde proyectos pequeños hasta cargas empresariales.

**Casos de uso:** moderación de contenido inapropiado, verificación de identidad en línea, etiquetado y catalogación automática de contenido multimedia y alertas inteligentes en hogares conectados (por ejemplo, reconocer rostros u objetos concretos y avisar de actividad inusual).

#### Amazon Textract

Extrae **texto, formularios (pares clave-valor) y tablas** de documentos escaneados. A diferencia del OCR tradicional, usa ML para entender la **estructura y el diseño** del documento y conserva las relaciones semánticas entre sus elementos. Por ejemplo, distingue cabeceras, filas y columnas de una tabla y así la información extraída mantiene su estructura y su significado. Así convierte datos no estructurados en datos estructurados. Esto es clave en finanzas, documentación legal o historiales médicos, donde el contexto de un dato importa tanto como el dato.

Se integra con otros servicios para montar *pipelines*: los resultados pueden guardarse en **S3**, consultarse con **Athena** o usarse en **SageMaker**.

**Casos de uso:** facturas, recibos y documentos fiscales (finanzas); digitalización de historiales clínicos (salud); contratos (legal); formularios y solicitudes (sector público).

### 1.2 Voz

#### Amazon Polly

Servicio de **texto a voz (TTS)** con voces realistas en muchos idiomas y acentos. Sintetiza en **tiempo real con baja latencia** y admite **SSML** (*Speech Synthesis Markup Language*) para controlar el tono, la velocidad y la pronunciación.

**Casos de uso:** sistemas IVR, audiolibros, accesibilidad para personas con discapacidad visual y narración en plataformas de *e-learning*.

#### Amazon Transcribe

Servicio de **reconocimiento automático del habla (ASR)**: convierte audio o vídeo en texto, tanto por lotes como en *streaming* (tiempo real). Soporta múltiples idiomas y dialectos e incluye identificación de hablantes, puntuación automática y **vocabularios personalizados**.

**Casos de uso:** subtítulos, transcripción de pódcast y entrevistas, subtitulado en directo para personas con discapacidad auditiva y análisis y cumplimiento normativo en *call centers*.

### 1.3 Lenguaje

#### Amazon Translate

Servicio de **traducción automática neuronal**, rápido y de coste ajustado, que cubre decenas de idiomas. Ofrece traducción **en tiempo real** (webs, apps, chat) y **por lotes** (documentos o conjuntos de datos grandes). Conserva el contexto para que la traducción suene natural y se integra con otros servicios de AWS.

**Casos de uso:** contenido web, descripciones de productos en *e-commerce*, soporte al cliente multilingüe (chat, email), manuales e informes, y mensajería en tiempo real.

#### Amazon Comprehend

Servicio de **NLP** que extrae información de texto no estructurado: **sentimiento, frases clave, entidades nombradas, idioma** y organización de documentos por temas o categorías. Es configurable, porque permite crear **modelos personalizados** de clasificación y de reconocimiento de entidades adaptados a la terminología de cada sector.

> **Corrección:** para extraer entidades médicas (medicamentos, afecciones, tratamientos) de historiales clínicos, el servicio específico es **Amazon Comprehend Medical**.

**Casos de uso:** sentimiento en reseñas y redes sociales, etiquetado y organización automática de documentos y análisis de tickets de soporte para detectar problemas recurrentes.

Para un modelado de temas más avanzado, SageMaker ofrece los algoritmos integrados **LDA** y **NTM** (sección 2.2).

### 1.4 Chatbots: Amazon Lex

Amazon Lex permite crear **interfaces conversacionales de voz y texto** con la misma tecnología de *deep learning* (ASR + comprensión del lenguaje natural, NLU) que Amazon Alexa. Sus rasgos principales son estos:

- **Modelo de interacción:** en la consola se definen *intents* (intenciones), *slots* (datos que el bot debe recoger) y respuestas, sin experiencia en NLP.
- **Lógica de backend** con **AWS Lambda**.
- **Conversaciones multiturno:** mantiene el contexto para recoger información a lo largo de varios intercambios.
- **Integración nativa con Amazon Connect** (centro de contacto de AWS) para crear agentes automáticos 24/7 que descargan a los agentes humanos.
- Despliegue en web, móvil y canales de mensajería, con herramientas integradas de prueba y monitorización.

**Casos de uso:** atención al cliente, sobre todo; también búsqueda de productos y seguimiento de pedidos en *e-commerce*, citas y cribado de pacientes en salud y asistentes internos de TI y RR. HH.

### 1.5 Recomendación: Amazon Personalize

Amazon Personalize genera **recomendaciones personalizadas en tiempo real**. Entrena modelos con tres tipos de datos:

- **Interacciones** usuario-ítem (clics, visualizaciones, compras).
- **Metadatos de ítems** (género, precio…).
- **Metadatos de usuarios** (edad, etc.).

El modelo aprende patrones entre usuarios e ítems y puede actualizarse con **eventos en tiempo real** a medida que cambian las preferencias. Puedes elegir el algoritmo (*recipe*) según el caso de uso. Los enfoques que cubre son:

- **Filtrado colaborativo:** recomienda lo que gustó a usuarios similares.
- **Filtrado basado en contenido:** recomienda ítems con atributos parecidos a los que interesaron al usuario.
- **Híbrido:** combina ambos.

> **Corrección:** Personalize **no genera datos sintéticos** para «rellenar huecos» en datos dispersos. Para usuarios o ítems nuevos con poco historial (*cold start*) se apoya en los **metadatos** y en la **exploración** de ítems nuevos. Sus funciones de IA generativa son otras: **Content Generator**, que añade temas descriptivos a recomendaciones por lotes (p. ej., «Rise and shine» para desayunos); la opción de devolver **metadatos de las recomendaciones** para enriquecer *prompts* de un LLM; y la integración con **LangChain**.

**Casos de uso:** productos en *e-commerce*; películas, series y artículos en *streaming* y medios; *email marketing* personalizado; y recomendación de artículos de ayuda en soporte.

### 1.6 IA generativa: Amazon Bedrock

La IA generativa crea contenido nuevo (texto, imágenes, contenido multimodal) a partir de patrones aprendidos. **Amazon Bedrock** es un servicio **totalmente gestionado y *serverless*** para construir y escalar aplicaciones de IA generativa con **modelos fundacionales (FMs)** sin gestionar infraestructura.

- **Proveedores:** AI21 Labs, Anthropic, Cohere, Meta, Mistral AI, Stability AI y la propia Amazon (Titan y **Nova**).
- **Una sola API** para experimentar con distintos modelos, **personalizarlos** con datos propios (*fine-tuning* y RAG, *retrieval augmented generation*) y desplegarlos de forma segura.
- **Converse API:** interfaz uniforme para interactuar con los FMs en aplicaciones conversacionales. **No todos los modelos la soportan** (consulta la tabla de modelos y funciones soportados en la documentación).
- **Bedrock Marketplace:** catálogo de **FMs especializados** en tareas o dominios concretos (salud, finanzas, entretenimiento…) que se descubren, prueban e integran desde Bedrock.

#### Cómo elegir un FM

**1. Capacidades que necesitas:**

| Necesidad | Modelos de ejemplo |
|---|---|
| Texto: generación, análisis, traducción, Q&A, código, resúmenes, chat | Amazon Titan Text G1 – Premier / Express / Lite |
| Generación de imágenes a partir de texto | Amazon Nova Canvas |
| Multimodal (entrada de texto e imagen) | Amazon Nova Lite, Amazon Nova Pro |
| Solo texto, baja latencia y bajo coste | Amazon Nova Micro |

> El catálogo cambia rápido. Antes de elegir, revisa en la ficha del modelo su estado de ciclo de vida (*Active* / *Legacy* / *EOL*).

**2. Disponibilidad en la región.** Los FMs se ofrecen **por región**. Para el examen: si un modelo no está en tu región, puede usarse con **inferencia entre regiones** (*cross-Region inference*), siempre que esté soportada. Esta inferencia reparte el tráfico entre varias regiones, así que absorbe picos imprevistos y da más rendimiento (*throughput*) y resiliencia. No tiene coste adicional de enrutamiento.

> **Corrección:** no hace falta *crear* un perfil de inferencia para usarla. Basta con invocar el modelo mediante un **perfil de inferencia entre regiones predefinido por el sistema** (por ejemplo, el ID del modelo con prefijo geográfico `us.`). Los **perfiles de inferencia de aplicación**, que sí creas tú, son opcionales y sirven para seguir costes y uso.

**3. Precio.** Hay dos modelos principales:

- **On-Demand y Batch:** pagas por los **tokens de entrada y de salida** procesados. Un token es una secuencia de caracteres que el modelo trata como unidad de significado. Los modelos de imagen, como Nova Canvas, se cobran **por imagen generada**.
- **Provisioned Throughput:** te comprometes a un nivel de rendimiento durante un periodo. Suele salir más rentable con uso constante.

#### Ejemplo: generar una imagen con Nova Canvas

**Contexto.** La región por defecto de la cuenta era us-east-2 (Ohio), donde Nova Canvas no estaba disponible de forma nativa en ese momento. Para no usar inferencia entre regiones, el cliente se crea en **us-east-1** (N. Virginia). En el libro se «solicita acceso» al modelo desde la consola; hoy ese paso ya no existe (ver la nota de vigencia).

**Entorno: Amazon SageMaker Studio.** Es el IDE web para *pipelines* de ML de extremo a extremo. Trae la mayoría de librerías de ML y la aplicación **Code Editor**, basada en VS Code. Se prepara así:

1. Se crea un **dominio** en la región donde se ejecutará el código (aquí, us-east-1) y se abre Studio (Figs. 4.3–4.5).
2. Se crea y arranca un **espacio** (*space*), es decir, la instancia gestionada que ejecuta Code Editor. En el ejemplo es un `ml.t3.medium` con 5 GB de almacenamiento.
3. **Para no incurrir en costes, detén el espacio al terminar.**

> **Corrección:** el almacenamiento de un espacio de Code Editor o JupyterLab es un **volumen Amazon EBS** (5 GB por defecto, ampliable), no EFS. Ese volumen guarda el código y los artefactos en `/home/sagemaker-user` y **persiste entre sesiones**. EFS era el almacenamiento de Studio Classic.

El programa (guardado como `test_nova_canvas.py` y ejecutado en Code Editor) es el siguiente:

```python
import base64
import io
import json
import os
import boto3
from PIL import Image
from botocore.config import Config
from botocore.exceptions import ClientError

class ImageError(Exception):
    "Error devuelto por Amazon Nova Canvas"

def generate_image(model_id, body):
    """Envía el prompt a Nova Canvas y devuelve los bytes de la imagen generada."""
    bedrock = boto3.client('bedrock-runtime', region_name='us-east-1',  # región con soporte nativo
                           config=Config(read_timeout=300))
    response = bedrock.invoke_model(body=body, modelId=model_id,
                                    accept='application/json', contentType='application/json')
    response_body = json.loads(response['body'].read())
    if response_body.get('error'):            # revisar el error ANTES de leer la imagen
        raise ImageError(f"Image generation error: {response_body['error']}")
    return base64.b64decode(response_body['images'][0])   # la imagen llega en base64

model_id = 'amazon.nova-canvas-v1:0'
body = json.dumps({
    'taskType': 'TEXT_IMAGE',
    'textToImageParams': {'text': 'Generate an image of a white sand beach at sunset.'},
    'imageGenerationConfig': {'numberOfImages': 1, 'height': 1024, 'width': 1024,
                              'cfgScale': 8.0, 'seed': 0},
})

try:
    image = Image.open(io.BytesIO(generate_image(model_id, body)))
    out_dir = '/home/sagemaker-user/ch04/images'   # volumen EBS del espacio de Studio
    os.makedirs(out_dir, exist_ok=True)
    image.save(os.path.join(out_dir, 'white_sand_beach_sunset.png'))
except (ClientError, ImageError) as err:
    print(f'Error: {err}')
```

**Cómo funciona.**

1. Con `boto3` se crea un cliente `bedrock-runtime` en us-east-1.
2. `invoke_model` envía el cuerpo JSON: la tarea `TEXT_IMAGE`, el *prompt* en `textToImageParams.text` y la configuración de la imagen (número de imágenes, tamaño, `cfgScale`, `seed`).
3. La respuesta trae la imagen codificada en base64. El programa la decodifica y la guarda como `white_sand_beach_sunset.png` en el volumen del espacio, donde persiste entre sesiones (Figs. 4.6 y 4.7).

> **Correcciones al código del libro:**
> - La región se fija explícitamente con `region_name='us-east-1'`. El libro dice crear el cliente en us-east-1, pero su código dependía de la región por defecto del entorno.
> - El campo `error` se comprueba **antes** de leer `images`. En el orden del libro, una respuesta con error fallaba al acceder a la imagen y nunca llegaba a la comprobación.
>
> Se eliminó el *logging* para compactar.

#### Cuándo usar (y cuándo no) Bedrock

**Úsalo** para construir aplicaciones de IA generativa (texto, imágenes, multimodal) con FMs de alto rendimiento, infraestructura gestionada y pago por uso. Sirve tanto para experimentar como para producción, desde *startups* hasta grandes empresas.

**No es la mejor opción** en estos casos:

- El modelo que necesitas no está disponible en Bedrock ni en su Marketplace.
- Necesitas **control total** del entrenamiento (p. ej., entrenar desde cero), de la arquitectura del modelo o de la infraestructura. En ese caso, SageMaker AI o un modelo propio encajan mejor. Nota: Bedrock **sí** permite personalizar FMs con datos propios mediante *fine-tuning* y RAG.
- Tienes requisitos de latencia muy específicos.
- Las necesidades de IA son mínimas y una solución más simple y barata basta.

---

## 2. Algoritmos integrados de Amazon SageMaker AI

### Por qué usarlos

Los servicios de IA (Rekognition, Comprehend, Personalize…) son fáciles y rápidos de adoptar, pero **poco flexibles** para casos especializados. Los **algoritmos integrados** (*built-in*) de SageMaker convienen cuando el proyecto exige:

- ingeniería de características propia,
- arquitecturas u objetivos de optimización específicos,
- ajuste de hiperparámetros,
- incorporar conocimiento del dominio o preprocesamiento particular.

Están **optimizados para velocidad, escala y exactitud**. Se integran con el ecosistema de AWS: datos desde **S3, EFS o FSx for Lustre**, preprocesamiento con **Lambda**, despliegue en **endpoints de SageMaker** (o Lambda) para inferencia en tiempo real y **entrenamiento distribuido** para grandes volúmenes de datos.

La Fig. 4.8 del libro relaciona cada algoritmo integrado con el tipo de problema, el formato de entrada y los casos de uso.

### Flujo común a todos los algoritmos integrados

El procedimiento es el mismo para todos, así que no se repite en cada algoritmo:

1. **Preparar los datos** y subirlos a **Amazon S3**, EFS o FSx for Lustre.
2. **Crear un *estimator***: imagen del contenedor del algoritmo, rol IAM, tipo y número de instancias y ruta de salida en S3.
3. **Fijar los hiperparámetros** del algoritmo.
4. **Lanzar el *training job***. SageMaker aprovisiona, escala y libera la infraestructura.
5. **Desplegar** el modelo como **endpoint** para predicciones en tiempo real, o usarlo en **batch transform** para inferencia por lotes.

En cada algoritmo solo se destacan sus hiperparámetros y particularidades.

### 2.1 Aprendizaje supervisado

El aprendizaje supervisado **aprende de datos etiquetados**. Cada ejemplo tiene características (*features*) y su etiqueta (*target*). El modelo aprende la relación entre entradas y salidas para predecir datos nuevos. Se usa cuando hay etiquetas y una relación clara entre *features* y *target*.

| Tipo de problema | Objetivo | Algoritmos típicos |
|---|---|---|
| **Regresión** | Predecir un valor continuo (p. ej., precio de una vivienda) | Regresión lineal, árboles de decisión, XGBoost |
| **Clasificación binaria** | Distinguir dos clases (p. ej., spam / no spam) | Regresión logística, SVM, k-NN |
| **Clasificación multiclase** | Más de dos clases | Árboles de decisión, *random forest*, redes neuronales |
| **Pronóstico de series temporales** | Valores futuros a partir del pasado | DeepAR |
| **Embeddings** | Representaciones semánticas de objetos para tareas posteriores (recomendación, búsqueda semántica) | Object2Vec |

#### Linear Learner

**Linear Learner** resuelve problemas de **clasificación y regresión**:

- **Datos:** muestras etiquetadas $(\mathbf{x}_i, y_i)$, donde $\mathbf{x}_i \in \mathbb{R}^d$ es el vector de *features* e $y_i$ la etiqueta.
  - Regresión: $y_i$ es un número real.
  - Clasificación binaria: la etiqueta es 0 o 1.
  - Clasificación multiclase: la etiqueta va de 0 a `num_classes − 1`.
- **Qué aprende:** una **función lineal** (regresión) o una **función umbral lineal** (clasificación).
- **Entrada:** una matriz con filas = observaciones y columnas = *features*, más una columna con las etiquetas. Para entrenar acepta **RecordIO-protobuf o CSV**.
- **Hiperparámetros:**
  - Obligatorio: `predictor_type` (`binary_classifier`, `multiclass_classifier` o `regressor`), además de las ubicaciones de datos de entrada y salida.
  - `num_classes`: obligatorio solo en multiclase.
  - Opcionales: épocas, regularización (`l1`, `wd`), función de pérdida (`loss`), etc. Van en el mapa `HyperParameters` de la petición `CreateTrainingJob`.

> **Corrección:** `feature_dim` **no es obligatorio**: es opcional y por defecto vale `auto`.

Con la función de pérdida adecuada, Linear Learner cubre los tres modelos lineales clásicos:

| Modelo | Tipo de problema | `loss` en Linear Learner |
|---|---|---|
| Regresión lineal | Regresión | `squared_loss` (por defecto en regresión) |
| Regresión logística | Clasificación | `logistic` (por defecto en binaria) |
| SVM **lineal** | Clasificación binaria | `hinge_loss` |

##### Regresión lineal

Modela la relación entre una variable dependiente (*target*) y una o más independientes (*features*) ajustando una ecuación lineal:

$$y = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \dots + \beta_n x_n + \varepsilon$$

donde $\beta_0$ es el intercepto, $\beta_1,\dots,\beta_n$ son los coeficientes (pesos) y $\varepsilon$ es el error o **residuo** (diferencia entre el valor real y el predicho). El modelo aprende los coeficientes que **minimizan el error cuadrático medio (MSE)** en todo el conjunto de datos:

$$\text{MSE} = \frac{1}{n}\sum_{i=1}^{n}\left(y_i - \hat{y}_i\right)^2$$

donde $n$ es el número de observaciones, $y_i$ el valor real y $\hat{y}_i$ el predicho. Minimizar el MSE acerca las predicciones a los valores reales, y analizar los residuos indica qué tan bien funciona el modelo.

Ejemplo con `LinearRegression` de scikit-learn:

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.linear_model import LinearRegression

# Datos sintéticos: y = 4 + 3x + ruido
np.random.seed(0)
X = 2 * np.random.rand(100, 1)
y = 4 + 3 * X + np.random.randn(100, 1)

model = LinearRegression().fit(X, y)
y_pred = model.predict(X)

plt.scatter(X, y, color='blue', label='Data Points')
plt.plot(X, y_pred, color='red', label='Regression Line')
for i in range(len(X)):                      # residuos (línea discontinua)
    plt.plot([X[i], X[i]], [y[i], y_pred[i]], color='green', linestyle='--')
plt.xlabel('X'); plt.ylabel('y'); plt.legend(); plt.show()

print(f"Mean Squared Error: {np.mean((y - y_pred) ** 2)}")
```

**Cómo funciona.**

1. **Datos:** 100 muestras de una sola *feature*, uniforme entre 0 y 2, con $y = 4 + 3x$ más ruido.
2. **Ajuste:** con `LinearRegression`.
3. **Visualización (Fig. 4.9):** puntos, recta de regresión y residuos como líneas discontinuas.
4. **MSE:** cuantifica el rendimiento del modelo.

##### Regresión logística

Pese al nombre, es un algoritmo de **clasificación**. Modela la **probabilidad de que un ejemplo pertenezca a la clase positiva** a partir de una combinación lineal de las *features*. Es especialmente útil cuando la variable dependiente es **binaria**.

> **Corrección:** el libro habla de la probabilidad de «predecir la categoría correcta». Lo que modela es la probabilidad de la **clase positiva**.

Aplica la **función logística (sigmoide)** a la combinación lineal:

$$\sigma(z) = \frac{1}{1 + e^{-z}}, \qquad z = \beta_0 + \beta_1 x_1 + \dots + \beta_n x_n$$

La sigmoide transforma cualquier número real en un valor del intervalo $(0, 1)$, interpretable como probabilidad (Fig. 4.10). Su curva en S tiende a 1 cuando $z$ es muy grande y a 0 cuando es muy pequeño, sin alcanzar nunca esos extremos. Para clasificar se usa un **umbral**, normalmente 0,5.

El siguiente programa predice si una flor del **dataset Iris** es *Iris-virginica* usando solo la longitud y la anchura del sépalo. Iris contiene 150 flores de tres especies (*setosa*, *versicolor*, *virginica*) con cuatro medidas: longitud y anchura del sépalo y del pétalo. Se carga con `load_iris` de `sklearn.datasets`.

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn import datasets
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split

iris = datasets.load_iris()
X = iris.data[:, :2]                   # longitud y anchura del sépalo
y = (iris.target == 2).astype(int)     # 1 = Iris-virginica, 0 = otra especie
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
model = LogisticRegression().fit(X_train, y_train)

# Probabilidad de la clase positiva sobre una malla que cubre el espacio de features
x_min, x_max = X[:, 0].min() - 1, X[:, 0].max() + 1
y_min, y_max = X[:, 1].min() - 1, X[:, 1].max() + 1
xx, yy = np.meshgrid(np.arange(x_min, x_max, 0.01), np.arange(y_min, y_max, 0.01))
Z = model.predict_proba(np.c_[xx.ravel(), yy.ravel()])[:, 1].reshape(xx.shape)

contour = plt.contourf(xx, yy, Z, alpha=0.8, cmap=plt.cm.Greys, levels=np.linspace(0, 1, 11))
plt.scatter(X[y == 0, 0], X[y == 0, 1], c='white', edgecolors='k', marker='o', label='Not Iris-Virginica')
plt.scatter(X[y == 1, 0], X[y == 1, 1], c='white', edgecolors='k', marker='s', label='Iris-Virginica')
plt.colorbar(contour, label='Probability of Iris-Virginica')
plt.xlabel('Sepal length'); plt.ylabel('Sepal width'); plt.legend(); plt.show()
```

**Cómo funciona.**

1. Selecciona dos *features* y binariza el *target* (virginica o no).
2. Separa entrenamiento y prueba con `train_test_split` y entrena el modelo con `fit`.
3. Crea una malla de puntos con `np.meshgrid` y calcula en cada punto la probabilidad de *virginica* con `predict_proba`.
4. Dibuja esas probabilidades con `contourf` y superpone los datos: cuadrados = *virginica*, círculos = otras especies (Fig. 4.11).

**Por qué la frontera es lineal.** La regresión logística ajusta una combinación lineal de las *features* a los ***log-odds*** del evento. Si $p$ es la probabilidad de que la muestra sea *virginica*:

$$\ln\frac{p}{1-p} = z \quad\Longleftrightarrow\quad p = \sigma(z)$$

El término $\ln\frac{p}{1-p}$ es la **función logit**. Como $z$ es lineal en las *features*, la **frontera de decisión** ($p = 0{,}5$, es decir, $z = 0$) es una **recta**. A un lado quedan los puntos con $p > 0{,}5$ y al otro los de $p < 0{,}5$. Las curvas de nivel de probabilidad de la Fig. 4.11 son rectas paralelas.

> **Corrección:** esas rectas **no** están igualmente espaciadas. Por la forma de la sigmoide, están más juntas cerca de $p = 0{,}5$ y más separadas hacia 0 y 1.

##### Máquinas de vectores de soporte (SVM)

**En clasificación**, una SVM busca el **hiperplano que separa las clases con el margen máximo**. El margen es la distancia entre el hiperplano y los puntos más cercanos de cada clase, llamados **vectores de soporte**. Maximizar el margen mejora la generalización. Con el **truco del kernel** (lineal, polinómico, RBF), las SVM resuelven problemas no lineales, porque trazan una frontera lineal en un espacio transformado de mayor dimensión.

**En regresión (SVR)**, se busca la función más «plana» posible cuyas desviaciones respecto a los valores reales no superen un margen de tolerancia $\varepsilon$. Los puntos que caen dentro de ese margen se ignoran, lo que hace el modelo robusto frente a ruido y *outliers*. También admite kernels.

> **Nota SageMaker:** Linear Learner solo implementa **SVM lineal** (`hinge_loss`, clasificación binaria). No hay SVM con kernel integrada; para usarla, necesitas tu propio código (p. ej., scikit-learn).

##### Cuándo usar cada modelo lineal

**Regresión lineal.**

- **Úsala** cuando hay una relación claramente lineal y un *target* continuo, por ejemplo el precio según los metros cuadrados y el número de habitaciones. Es fácil de implementar e interpretar, útil cuando importa la transparencia.
- **Evítala** con relaciones no lineales o interacciones entre variables, con muchos *outliers* o con multicolinealidad, a los que es sensible. En esos casos, usa regresión polinómica, árboles o *gradient boosting*.

**Regresión logística.**

- **Úsala** en clasificación binaria (spam, diagnóstico positivo/negativo, *churn*) cuando la relación entre las *features* y los *log-odds* es lineal. Es interpretable y da la **probabilidad** de pertenencia a la clase.
- **Evítala** con relaciones no lineales, donde árboles, SVM o redes neuronales la superan. También supone ausencia de multicolinealidad y necesita suficientes datos, así que no es adecuada para datos pequeños, muy correlacionados o de muy alta dimensión.

> **Multicolinealidad:** dos o más predictores están muy correlacionados, de modo que uno se puede predecir casi linealmente a partir de los otros. Esto genera redundancia y coeficientes poco fiables. Se mitiga con **regularización**, que penaliza los coeficientes grandes (capítulo 5).

**SVM.**

- **Úsala** con datos de **alta dimensión** y clases bien separadas por un margen claro. También cuando hay **más *features* que muestras**, porque es menos propensa al sobreajuste.
- **Evítala** con **datasets grandes** (entrenamiento lento y mucha memoria) y con datos ruidosos o clases solapadas, donde *random forest* o *gradient boosting* rinden mejor. Exige ajustar bien el kernel y la regularización.
- En resumen: datasets pequeños o medianos (hasta unas 10 000 muestras), de alta dimensión y con clases separables.

> **Sobreajuste (*overfitting*):** el modelo aprende también el ruido y los *outliers* del entrenamiento. Rinde muy bien en entrenamiento y mal con datos nuevos. Se mitiga con validación cruzada, poda de árboles, regularización o modelos más simples (capítulo 5).

#### k-Nearest Neighbors (k-NN)

k-NN es un método supervisado de **clasificación y regresión** basado en una idea simple: los puntos similares están cerca en el espacio de *features*. Para predecir un punto nuevo, busca sus **k vecinos más cercanos** y combina sus etiquetas:

- **Clasificación:** la etiqueta más frecuente entre los vecinos (voto mayoritario).
- **Regresión:** la media de las etiquetas de los vecinos.

**El hiperparámetro k** se fija antes del aprendizaje y define el «vecindario» que influye en la predicción:

- Un **k pequeño** (p. ej., $k = 1$) hace el modelo muy sensible al ruido.
- Un **k grande** suaviza las predicciones, pero puede ignorar patrones locales.

Se elige equilibrando ambos efectos, normalmente con validación cruzada (capítulo 5).

**Coste y distancia.** Es costoso con muchos datos, porque calcula la distancia del punto consultado a todos los demás. La métrica de distancia (euclídea, Manhattan, Minkowski) influye mucho en el rendimiento y debe ajustarse a la naturaleza de los datos.

Ejemplo con una versión bidimensional de Iris:

```python
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.colors import ListedColormap
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.neighbors import KNeighborsClassifier

iris = load_iris()
X, y = iris.data[:, :2], iris.target   # solo 2 features para poder graficar
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

k = 3
knn = KNeighborsClassifier(n_neighbors=k).fit(X_train, y_train)

# Frontera de decisión: predicción de k-NN sobre una malla
x_min, x_max = X[:, 0].min() - 1, X[:, 0].max() + 1
y_min, y_max = X[:, 1].min() - 1, X[:, 1].max() + 1
xx, yy = np.meshgrid(np.arange(x_min, x_max, 0.1), np.arange(y_min, y_max, 0.1))
Z = knn.predict(np.c_[xx.ravel(), yy.ravel()]).reshape(xx.shape)
plt.contourf(xx, yy, Z, alpha=0.3, cmap=ListedColormap(('gray', 'lightgray', 'darkgray')))

# Forma del marcador = clase real; relleno = entrenamiento, hueco = prueba
labels = ['Iris-setosa', 'Iris-versicolor', 'Iris-virginica']
for i, marker in enumerate(['o', 's', '^']):
    plt.scatter(X_train[y_train == i, 0], X_train[y_train == i, 1], c='black',
                marker=marker, edgecolor='k', label=f'Training {labels[i]}')
    plt.scatter(X_test[y_test == i, 0], X_test[y_test == i, 1], facecolors='none',
                edgecolors='k', marker=marker, label=f'Test {labels[i]}')
plt.xlabel(iris.feature_names[0]); plt.ylabel(iris.feature_names[1])
plt.title(f'k-NN Classification (k={k})'); plt.legend(); plt.show()
```

**Cómo funciona.**

1. Carga Iris con dos *features* (longitud y anchura del sépalo) y separa entrenamiento y prueba.
2. Entrena k-NN con $k = 3$.
3. Pinta las regiones de decisión predichas (Fig. 4.12): *setosa* a la izquierda, *versicolor* abajo, *virginica* a la derecha.
4. Superpone los puntos: la forma del marcador indica la **clase real**, así se ven los aciertos y los errores. Los puntos de prueba van huecos (`facecolors='none'`, `edgecolors='k'`).

Varios puntos de entrenamiento de *virginica* caen en la región de *versicolor*: el modelo con $k = 3$ los clasifica mal. En el capítulo 5 se verá cómo evaluar el modelo y ajustar $k$.

**En SageMaker** (ver el flujo común), el estimador de k-NN se configura con, entre otros, el número de vecinos $k$ y la métrica de distancia.

**Cuándo usar k-NN.**

- **Úsalo** en clasificación o regresión con datasets **pequeños** (menos de ~1000 muestras) o **moderados** (hasta ~10 000), de **pocas dimensiones** y donde la noción de «cercanía» tenga sentido. No supone una forma concreta de frontera, así que también captura fronteras no lineales. Es **fácil de explicar**: cada predicción se justifica con los vecinos que la originan.
- **Evítalo** con datasets grandes (alto coste de cómputo y memoria en inferencia) y con **alta dimensionalidad** (la «maldición de la dimensionalidad» vuelve poco informativas las distancias). También rinde mal con datos ruidosos si no se ajusta con cuidado, y con muchas *features* irrelevantes o correlacionadas; en esos casos considera SVM o *random forest*.
- Requiere preprocesamiento, sobre todo **escalado de *features***.

#### Árboles de decisión, Random Forest y XGBoost

Un **árbol de decisión** es un modelo en forma de árbol para clasificación y regresión:

- El **nodo raíz** representa todo el dataset.
- Los **nodos internos** dividen los datos según una condición.
- Las **ramas** son los resultados de cada decisión.
- Las **hojas** dan la predicción final.

El árbol **particiona recursivamente** el dataset buscando que cada subconjunto sea lo más **homogéneo (puro)** posible: misma clase en clasificación, valores similares en regresión. El proceso tiene cuatro pasos:

1. **Selección de *feature*:** en cada nodo se elige la mejor división según la impureza de Gini, la ganancia de información (entropía) o la reducción de varianza (en regresión). La varianza de un nodo es:
   $$\text{Var} = \frac{1}{n}\sum_{i=1}^{n}\left(y_i - \bar{y}\right)^2$$
2. **División:** cada subconjunto se convierte en un nodo nuevo y el proceso se repite.
3. **Criterio de parada:** profundidad máxima, mínimo de muestras por nodo o ausencia de mejora en pureza.
4. **Poda:** elimina ramas poco significativas para evitar el sobreajuste (p. ej., *cost-complexity pruning*).

Los árboles son interpretables y capturan relaciones no lineales, pero tienen **alta varianza**: un pequeño cambio en los datos puede producir un árbol muy distinto. Los **métodos de ensamble** lo resuelven:

- ***Random forest*:** muchos árboles entrenados **en paralelo**.
- ***Gradient boosting*:** árboles construidos **secuencialmente**, cada uno corrigiendo los errores de los anteriores.

##### Random Forest

*Random forest* combina muchos árboles, cada uno entrenado con una parte aleatoria de los datos.

> **Corrección:** el libro describe mal la aleatorización. Tiene dos fuentes:
> - **Bootstrapping:** cada árbol se entrena con una muestra aleatoria **con reemplazo** del conjunto de entrenamiento.
> - **Aleatoriedad en las *features*:** en **cada división** solo se considera un subconjunto aleatorio de *features*.

Las predicciones se agregan por **voto mayoritario** (clasificación) o **promedio** (regresión). Así se **reduce la varianza** y el modelo es más robusto. Como los árboles son independientes, se entrenan en paralelo y rápido. La contrapartida es que no corrigen secuencialmente los errores unos de otros, como hace el *boosting*.

> **Corrección importante para el examen:** Random Forest **no es un algoritmo integrado de SageMaker**. No lo confundas con **Random Cut Forest** (detección de anomalías). Para usarlo, entrénalo con tu propio script, p. ej., con scikit-learn. Los algoritmos integrados basados en árboles son **XGBoost, LightGBM y CatBoost**.

##### XGBoost

**XGBoost** (*extreme gradient boosting*) es un algoritmo de árboles muy eficiente y preciso. Construye un **ensamble secuencial**: cada árbol nuevo corrige los errores de los anteriores. Un resultado útil del entrenamiento es la **importancia de las *features***, que ayuda a seleccionar variables, ajustar el modelo y entender los datos.

**Comparación con *random forest*** (Fig. 4.13):

| | Random Forest | XGBoost |
|---|---|---|
| Construcción | Árboles independientes, en paralelo | Árboles secuenciales que corrigen errores previos |
| Qué reduce principalmente | Varianza | Sesgo (y controla la varianza con regularización y submuestreo) |
| Exactitud | Buena y robusta | Suele ser superior en datos tabulares complejos |
| Sensibilidad a hiperparámetros | Baja | Alta: requiere ajuste cuidadoso |

> **Corrección:** el libro afirma que XGBoost reduce «tanto sesgo como varianza» y que es «menos propenso al sobreajuste que *random forest*». Es más preciso decir lo siguiente: el *boosting* reduce sobre todo el sesgo; XGBoost incorpora **regularización L1/L2**, *shrinkage* (`eta`) y submuestreo para controlar el sobreajuste; y aun así puede sobreajustar si no se ajusta bien. *Random forest* es más robusto con la configuración por defecto.

Ejemplo con Iris: entrenamiento, importancia de *features* y predicción de puntos nuevos.

```python
import numpy as np
import matplotlib.pyplot as plt
import xgboost as xgb
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(iris.data, iris.target, test_size=0.2, random_state=42)
dtrain = xgb.DMatrix(X_train, label=y_train)    # estructura de datos interna de XGBoost

params = {'max_depth': 3, 'eta': 0.1, 'objective': 'multi:softprob', 'num_class': 3}
bst = xgb.train(params, dtrain, 50)             # 50 rondas de boosting

# multi:softprob devuelve una probabilidad por clase → argmax = clase predicha
new_data = np.array([[5.1, 3.5, 1.4, 0.2], [6.2, 3.4, 5.4, 2.3]])
class_names = {0: 'Iris-setosa', 1: 'Iris-versicolor', 2: 'Iris-virginica'}
for i, (point, probs) in enumerate(zip(new_data, bst.predict(xgb.DMatrix(new_data)))):
    print(f'Prediction for datapoint {i+1} ({point}): {class_names[int(np.argmax(probs))]}')

# Importancia de features: f0..f3 = sepal length, sepal width, petal length, petal width
xgb.plot_importance(bst)
plt.show()
```

**Resultados.** La longitud del pétalo (`f2`) resulta la *feature* más influyente (Fig. 4.14). Los dos puntos nuevos, cuatro medidas en cm cada uno, se clasifican como *setosa* y *virginica* (Fig. 4.15).

> Se quitaron del código del libro el `DMatrix` de prueba y el import de `accuracy_score`, que no se usaban, y la leyenda personalizada del gráfico.

**En SageMaker**, estos son los hiperparámetros principales de XGBoost:

| Hiperparámetro | Qué controla |
|---|---|
| `objective` | Función objetivo: `binary:logistic` (clasificación binaria), `reg:squarederror` (regresión), `multi:softprob`… |
| `num_round` | Número de rondas de *boosting* (**obligatorio**) |
| `eta` | *Learning rate* |
| `max_depth` | Profundidad de cada árbol |
| `subsample` | Fracción de muestras usada en cada árbol |
| `colsample_bytree` | Fracción de *features* usada en cada árbol |

La lista completa está en la documentación de hiperparámetros de XGBoost.

##### Cuándo usar cada uno

**Árbol de decisión.**

- **Úsalo** cuando la **interpretabilidad** es crítica: es fácil de visualizar y explicar a los interesados, trabaja con datos numéricos y categóricos y necesita poco preprocesamiento.
- **Evítalo** con datasets grandes y ruidosos o cuando se requiere máxima exactitud, porque tiende a sobreajustar.

***Random forest*.**

- **Úsalo** para mejorar la exactitud y reducir el sobreajuste de un árbol individual, con datasets grandes y de alta varianza, o cuando interesa la importancia de *features*.
- **Evítalo** en tiempo real o con recursos limitados si el modelo tiene muchos árboles profundos, porque se vuelve costoso.

**XGBoost.**

- **Úsalo** cuando prima la **máxima exactitud y eficiencia**: datasets grandes, interacciones complejas y competiciones.
- **Evítalo** si no puedes invertir en ajustar hiperparámetros o si la tarea es simple y valen más la interpretabilidad y la rapidez de despliegue.

#### Recomendación: Factorization Machines y Object2Vec

Los sistemas de recomendación manejan datos **de alta dimensión y muy dispersos**. En SageMaker hay dos algoritmos para ello:

- **Factorization Machines:** aprende **factores latentes** de las interacciones usuario-ítem.
- **Object2Vec:** aprende **embeddings**, es decir, representaciones vectoriales continuas que capturan relaciones semánticas, de objetos diversos.

##### Factorization Machines

Diseñado para datasets **dispersos de alta dimensión**, típicos de recomendación y predicción de clics. Captura **interacciones entre *features*** que un modelo lineal tradicional pasaría por alto. Extiende la factorización de matrices a un rango más amplio de interacciones. Gracias a los factores latentes, modela esas interacciones de forma compacta y maneja bien la dispersión, incluso con matrices usuario-ítem enormes y casi vacías. Soporta **clasificación binaria y regresión**.

**Hiperparámetros:**

- **Obligatorios:**
  - `predictor_type` (`binary_classifier` o `regressor`).
  - `num_factors`: dimensión de la factorización.
  - `feature_dim`: número total de *features*.
- **Opcionales:** `mini_batch_size`, `epochs`.
- ***Learning rates*:** `bias_lr`, `linear_lr` y `factors_lr`.
- **Regularización (*weight decay*):** `bias_wd`, `linear_wd` y `factors_wd`.

> **Corrección:** el libro presenta `bias_lr`, `linear_lr` y «`factor_lr`» como parámetros de regularización contra el sobreajuste. En realidad son ***learning rates*** (y el tercero se llama `factors_lr`). La regularización se controla con los parámetros `*_wd`.

**Casos de uso:** recomendación en *e-commerce* a partir de compras e interacciones y **predicción de CTR** (*click-through rate*) en publicidad online. En general, cualquier escenario con datos dispersos a gran escala.

##### Object2Vec

Algoritmo neuronal de **embeddings**. Genera embeddings **densos y de baja dimensión** a partir de objetos de alta dimensión, capturando sus relaciones semánticas. **Generaliza Word2Vec** a distintos tipos de datos, estructurados y no estructurados. Es **supervisado**: aprende a partir de pares de objetos etiquetados con su relación.

**Hiperparámetros:**

| Hiperparámetro | Qué controla |
|---|---|
| `enc0_max_seq_len` | Longitud máxima de secuencia del codificador enc0 |
| `enc0_vocab_size` | Tamaño del vocabulario de enc0 |
| `enc_dim` | Dimensión del embedding de salida |
| `dropout` | Probabilidad de *dropout* |
| `early_stopping_patience` | Épocas consecutivas sin mejora antes de parar |
| `comparator_list` | Cómo se comparan los embeddings |
| `learning_rate`, `mini_batch_size`, `optimizer` | Parámetros de optimización |

**Cuándo usarlo.**

- **Úsalo** para generar embeddings de calidad a partir de datos complejos: recomendación, búsqueda por similitud, *clustering* o *features* para modelos supervisados posteriores.
- **Evítalo** si solo necesitas transformaciones lineales simples o si los datos no tienen relaciones complejas que capturar. Si se requiere entrenamiento en tiempo real o latencias muy estrictas a gran escala, hay opciones más adecuadas.

#### Pronóstico: DeepAR

**DeepAR** es el algoritmo integrado de **pronóstico de series temporales**. Usa **redes neuronales recurrentes (RNN)** para capturar patrones temporales complejos. A diferencia de los métodos estadísticos clásicos, entrena **un solo modelo con muchas series relacionadas** (distintos productos o ubicaciones) y aprende patrones compartidos que mejoran cada pronóstico individual.

Genera **pronósticos probabilísticos**: en lugar de un único valor, devuelve una **distribución** de valores futuros. Así se cuantifica la incertidumbre y se pueden gestionar riesgos y optimizar recursos considerando todo el abanico de escenarios.

**Hiperparámetros:**

| Hiperparámetro | Qué controla |
|---|---|
| `context_length` | Pasos del pasado que ve el modelo antes de predecir (**obligatorio**) |
| `prediction_length` | Horizonte: pasos futuros a predecir (**obligatorio**) |
| `epochs` | Pasadas por los datos de entrenamiento (**obligatorio**) |
| `time_freq` | Granularidad de la serie: `M`, `W`, `D`, `H`, `min`… (**obligatorio**) |
| `num_layers`, `num_cells` | Profundidad y tamaño de la RNN |
| `mini_batch_size`, `learning_rate` | Eficiencia y ajuste del entrenamiento |

> **Corrección:** el libro no menciona `time_freq`, que también es obligatorio.

**Cuándo usarlo.**

- **Úsalo** con **muchas series temporales relacionadas** a gran escala cuando se quiere cuantificar la incertidumbre: demanda, inventario, predicciones financieras.
- **Evítalo** con pocos datos o una única serie simple, donde basta ARIMA o suavizado exponencial, y cuando se necesita entrenamiento en tiempo real o latencias muy bajas.

### 2.2 Aprendizaje no supervisado

En un crucigrama «sin diagrama» hay que resolver las pistas y además descubrir la estructura de la cuadrícula. El aprendizaje no supervisado funciona igual: **no hay etiquetas**, y el algoritmo busca patrones y estructuras directamente en los datos. En SageMaker cubre cuatro familias:

| Tarea | Algoritmos integrados | Idea |
|---|---|---|
| ***Clustering*** | K-means | Agrupar puntos similares, como ordenar piezas de un puzle por secciones |
| **Reducción de dimensionalidad** | PCA | Simplificar los datos, como doblar un mapa para ver solo una zona |
| **Modelado de temas** | LDA, NTM | Descubrir temas ocultos en grandes colecciones de texto |
| **Detección de anomalías** | Random Cut Forest, IP Insights | Detectar valores atípicos (fraude, irregularidades) |

#### Clustering: K-means

El *clustering* organiza los datos en **grupos significativos** y convierte puntos aislados en información interpretable, útil para tomar decisiones.

**K-means** agrupa los datos en un número **predefinido** $k$ de clústeres, asignando iterativamente cada punto al clúster más similar:

- **Similitud:** se mide con una distancia, normalmente la **euclídea** (distancia en línea recta en el espacio de *features*).
- **Objetivo:** minimizar la suma de distancias al cuadrado de cada punto al **centroide** de su clúster.
- **Centroide:** la media de los puntos del clúster, su «centro de masa».

##### Método del codo

La dificultad típica es elegir $k$. El **método del codo** grafica la ***within-cluster sum of squares* (WCSS)** frente al número de clústeres. La WCSS siempre baja al aumentar $k$, pero a partir de cierto punto baja mucho más despacio y forma un «codo». Ese punto sugiere el $k$ óptimo, que equilibra subajuste y sobreajuste.

Dados los clústeres $C_1, \dots, C_k$ con centroides $\mu_1, \dots, \mu_k$:

$$\text{WCSS} = \sum_{i=1}^{k}\sum_{x \in C_i}\lVert x - \mu_i \rVert^2$$

donde $k$ es el número de clústeres, $x$ un punto del clúster $C_i$, $\mu_i$ su centroide y $\lVert x - \mu_i \rVert^2$ la distancia euclídea al cuadrado.

Gráfico del codo para Iris con `KMeans` de `sklearn.cluster`:

```python
import os
import matplotlib.pyplot as plt
from sklearn.cluster import KMeans
from sklearn.datasets import load_iris
from sklearn.preprocessing import StandardScaler

os.makedirs('./images', exist_ok=True)
scaled_data = StandardScaler().fit_transform(load_iris().data)

wcss = []
for k in range(1, 11):
    kmeans = KMeans(n_clusters=k, random_state=0).fit(scaled_data)
    wcss.append(kmeans.inertia_)               # inertia_ = WCSS del modelo con k clústeres

plt.plot(range(1, 11), wcss, marker='o')
plt.xlabel('Number of Clusters'); plt.ylabel('Within-Cluster Sum of Squares (WCSS)')
plt.grid(True)
plt.savefig('./images/elbow_method_plot.png')
```

**Qué ocurre dentro del bucle.** `KMeans(n_clusters=k, random_state=0)` crea el modelo con semilla fija para que sea reproducible, y `fit` lo entrena en cuatro pasos:

1. **Inicialización de centroides.**
2. **Asignación:** cada punto va al centroide más cercano (distancia euclídea).
3. **Actualización:** cada centroide se recalcula como la media de sus puntos.
4. **Repetición** de asignación y actualización hasta la convergencia, cuando los centroides apenas cambian.

Después, `kmeans.inertia_` (la WCSS) se añade a la lista. Cuanto menor es, más compactos son los clústeres.

> **Corrección:** la inicialización no es necesariamente aleatoria. scikit-learn usa por defecto **k-means++**, que dispersa los centroides iniciales. El K-means de SageMaker permite elegirla con `init_method` (`random` por defecto o `kmeans++`).

El gráfico (Fig. 4.16) muestra el codo en **$k = 3$**: más clústeres no mejoran sustancialmente la compacidad. En Iris esto coincide con las tres especies.

##### Clustering de Iris en 3D

Iris tiene cuatro dimensiones y no se puede graficar directamente. Aquí se usan tres *features* (longitud del sépalo, longitud y anchura del pétalo). Un enfoque mejor es reducir la dimensionalidad con PCA, como se ve en la siguiente sección.

```python
import os
import matplotlib.pyplot as plt
from sklearn.datasets import load_iris
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans

os.makedirs('./ch04/images', exist_ok=True)
# 3 features: sepal length, petal length, petal width
scaled_data = StandardScaler().fit_transform(load_iris().data[:, [0, 2, 3]])

kmeans = KMeans(n_clusters=3, random_state=42).fit(scaled_data)
clusters = kmeans.labels_                      # clúster asignado a cada punto de entrenamiento

ax = plt.figure(figsize=(10, 8)).add_subplot(111, projection='3d')
for c, (color, marker) in enumerate(zip(['blue', 'orange', 'purple'], ['o', 's', 'D'])):
    pts = scaled_data[clusters == c]
    ax.scatter(pts[:, 0], pts[:, 1], pts[:, 2], c=color, marker=marker,
               edgecolor='black', label=f'Cluster {c + 1}')
for i, (x, y, z) in enumerate(kmeans.cluster_centers_):
    ax.scatter(x, y, z, s=300, c='yellow', marker='*', edgecolor='black', label=f'Centroid {i + 1}')
ax.set_xlabel('Standardized Sepal Length'); ax.set_ylabel('Standardized Petal Length')
ax.set_zlabel('Standardized Petal Width'); ax.legend()
plt.savefig('./ch04/images/kmeans_iris_3d_plot.png')
```

`kmeans.labels_` es un array de enteros con el clúster asignado a cada punto de entrenamiento. Se usa para colorear y agrupar los puntos del gráfico (Fig. 4.17). En él se ven los tres clústeres bien separados y los centroides como estrellas amarillas.

> **No confundas etiquetas con predicciones:**
> - `kmeans.fit()` **entrena** el modelo.
> - `kmeans.labels_` devuelve el clúster de cada punto **de entrenamiento**. Es una salida del ajuste, no un predictor.
> - `kmeans.predict()` asigna clúster a **datos nuevos**.

##### K-means en SageMaker

El siguiente snippet obtiene el contenedor de K-means, crea el estimador con $k$, entrena con datos en S3, despliega un endpoint y predice:

```python
import sagemaker
from sagemaker import get_execution_role
from sagemaker.inputs import TrainingInput
from sagemaker.serializers import CSVSerializer
from sagemaker.deserializers import JSONDeserializer
from sklearn.datasets import load_iris
from sklearn.preprocessing import StandardScaler

sagemaker_session = sagemaker.Session()
role = get_execution_role()
container = sagemaker.image_uris.retrieve('kmeans', sagemaker_session.boto_region_name)

# El escalador se AJUSTA con los datos de entrenamiento (los que se estandarizan y
# suben a S3) y se reutiliza en inferencia
scaler = StandardScaler().fit(load_iris().data)

kmeans = sagemaker.estimator.Estimator(
    container, role, instance_count=1, instance_type='ml.m4.xlarge',
    output_path='s3://your-bucket/path-to-output', sagemaker_session=sagemaker_session)
kmeans.set_hyperparameters(k=3, feature_dim=4)      # ambos son obligatorios

# CSV sin cabecera y sin columna de etiqueta → label_size=0
kmeans.fit({'train': TrainingInput('s3://your-bucket/path-to-train-data',
                                   content_type='text/csv;label_size=0')})

predictor = kmeans.deploy(initial_instance_count=1, instance_type='ml.m4.xlarge',
                          serializer=CSVSerializer(), deserializer=JSONDeserializer())

new_data = [[5.1, 3.5, 1.4, 0.2], [6.2, 3.4, 5.4, 2.3]]
print(predictor.predict(scaler.transform(new_data)))  # closest_cluster y distance_to_cluster
```

> **Correcciones al snippet del libro:**
> - Llamaba a `transform()` sobre un `StandardScaler` **sin ajustar**, lo que lanza `NotFittedError`. Hay que reutilizar el escalador ajustado con los datos de entrenamiento.
> - Faltaba `feature_dim`, que junto con `k` es **obligatorio** en el K-means de SageMaker.
> - Los datos CSV deben declararse con `content_type='text/csv'` (con `label_size=0` si no hay etiqueta). Si no se declara, los algoritmos integrados esperan RecordIO-protobuf.
> - `deploy()` sin *serializer* crea un predictor que solo acepta bytes. Con `CSVSerializer` y `JSONDeserializer` se le pueden pasar arrays y recibir JSON.

**Cuándo usar K-means.**

- **Úsalo** cuando sabes aproximadamente **cuántos clústeres** esperas y los grupos son **bien separados, esféricos y sin solapamiento**. Ejemplos: segmentación de clientes, compresión de imágenes, detección de anomalías. Escala bien con muchas muestras y funciona mejor con datos numéricos bien estructurados.
- **Evítalo** con clústeres de formas, tamaños o densidades variables, con muchos *outliers* (lo distorsionan) o cuando no hay forma de estimar $k$. Tampoco sirve con *features* no numéricas. Alternativas: **DBSCAN** o *clustering* **jerárquico**.

#### Reducción de dimensionalidad: PCA

La reducción de dimensionalidad simplifica datasets complejos **reduciendo el número de *features*** y conservando la información esencial. Facilita la visualización y el análisis, comprime los datos y mejora la eficiencia de los modelos posteriores. Hay dos técnicas comunes:

- **PCA:** busca las direcciones de máxima varianza.
- **t-SNE:** preserva la estructura local entre puntos.

SageMaker ofrece **PCA** como algoritmo integrado.

**PCA** (*principal component analysis*) identifica los **componentes principales**: direcciones **ortogonales** que explican la mayor varianza de los datos. Transforma los datos a un nuevo sistema de coordenadas definido por esos componentes y se queda con los primeros. Así reduce *features* conservando la mayor variabilidad posible, y además mitiga la multicolinealidad y el riesgo de sobreajuste.

El siguiente programa reduce Iris de cuatro a tres dimensiones con PCA y luego agrupa en tres clústeres con K-means:

```python
import os
import matplotlib.pyplot as plt
from sklearn.datasets import load_iris
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA
from sklearn.cluster import KMeans

os.makedirs('./ch04/images', exist_ok=True)
scaled_data = StandardScaler().fit_transform(load_iris().data)

pca_data_3d = PCA(n_components=3).fit_transform(scaled_data)   # 4 → 3 dimensiones
kmeans = KMeans(n_clusters=3, random_state=42).fit(pca_data_3d)
clusters = kmeans.predict(pca_data_3d)         # aquí coincide con kmeans.labels_

ax = plt.figure(figsize=(10, 8)).add_subplot(111, projection='3d')
for c, (color, marker) in enumerate(zip(['blue', 'orange', 'purple'], ['o', 's', 'D'])):
    pts = pca_data_3d[clusters == c]
    ax.scatter(pts[:, 0], pts[:, 1], pts[:, 2], c=color, marker=marker,
               edgecolor='black', label=f'Cluster {c + 1}')
for i, (x, y, z) in enumerate(kmeans.cluster_centers_):
    ax.scatter(x, y, z, s=300, c='yellow', marker='*', edgecolor='black', label=f'Centroid {i + 1}')
    ax.text(x, y, z, f'C{i + 1}', fontsize=12, ha='center', va='center')
ax.set_xlabel('Principal Component 1'); ax.set_ylabel('Principal Component 2')
ax.set_zlabel('Principal Component 3'); ax.legend()
plt.savefig('./ch04/images/pca_kmeans_iris_plot_3d.png')
plt.close()
```

**Resultados (Fig. 4.18).**

- Tres clústeres: círculos, cuadrados y rombos.
- Los ejes son los componentes principales PC1, PC2 y PC3, que capturan la máxima varianza y preservan la estructura esencial.
- Los centroides aparecen como estrellas etiquetadas C1–C3.

La clara separación muestra que PCA conserva bien la estructura de Iris y que K-means identifica los grupos.

##### PCA en SageMaker

La implementación de SageMaker es escalable y admite datos **densos y dispersos**. Funcionamiento:

1. Se crea un estimador PCA indicando cuántos componentes conservar.
2. Al entrenar, calcula los componentes principales.
3. El modelo entrenado se usa de dos formas: desplegado como **endpoint** para transformar datos al vuelo, o en **batch transform** para grandes volúmenes.

```python
import boto3
import pandas as pd
import sagemaker
from sagemaker import get_execution_role
from sagemaker.inputs import TrainingInput
from sklearn.datasets import load_iris

sagemaker_session = sagemaker.Session()
role = get_execution_role()

# Iris → CSV sin cabecera → S3
iris = load_iris()
pd.DataFrame(iris.data, columns=iris.feature_names).to_csv('iris.csv', index=False, header=False)
s3_bucket = sagemaker_session.default_bucket()
s3_data_path = f's3://{s3_bucket}/pca/iris.csv'
boto3.Session().resource('s3').Bucket(s3_bucket).Object('pca/iris.csv').upload_file('iris.csv')

pca_image_uri = sagemaker.image_uris.retrieve('pca', boto3.Session().region_name)
pca = sagemaker.estimator.Estimator(
    image_uri=pca_image_uri, role=role, instance_count=1,
    instance_type='ml.m4.xlarge', output_path=f's3://{s3_bucket}/pca/output')

# feature_dim, num_components y mini_batch_size son obligatorios
pca.set_hyperparameters(feature_dim=4, num_components=3, mini_batch_size=50,
                        subtract_mean=True, algorithm_mode='regular')
pca.fit({'train': TrainingInput(s3_data_path, content_type='text/csv;label_size=0')})

# Batch transform (no se despliega un endpoint): proyecta cada fila a 3 componentes
pca_transformer = pca.transformer(
    instance_count=1, instance_type='ml.m4.xlarge', strategy='SingleRecord',
    accept='application/jsonlines', assemble_with='Line')
pca_transformer.transform(data=s3_data_path, content_type='text/csv',
                          split_type='Line')      # espera a que termine; devuelve None

# Salida: <archivo de entrada>.out, una línea {"projection": [...]} por registro
transformed = pd.read_json(f'{pca_transformer.output_path}/iris.csv.out', lines=True)
print(transformed.head())
```

> **Correcciones al snippet del libro:**
> - Faltaba `mini_batch_size`, **obligatorio** en PCA.
> - El CSV debe declararse con `content_type='text/csv'`.
> - El comentario decía «Deploy», pero `transformer()` prepara un **batch transform**; no despliega ningún endpoint.
> - `transform()` **devuelve `None`** y ya espera a que termine, así que `transformed_output.wait()` y `transformed_output.output_path` fallaban. La ruta se lee de `pca_transformer.output_path`.
> - El archivo de salida se llama como el de entrada más `.out` (`iris.csv.out`, no `train.csv.out`).
> - PCA **no devuelve CSV**: responde en JSON, JSON Lines o RecordIO. Por eso se pide `application/jsonlines` y se lee con `read_json`.

El ajuste de hiperparámetros, el despliegue y la monitorización de modelos se ven en los capítulos 5, 6 y 7.

**Cuándo usar PCA.**

- **Úsalo** con datos de **alta dimensión** y **multicolinealidad**. Casos típicos: compresión de imágenes (menos valores para representar la imagen sin gran pérdida), **análisis exploratorio** (proyectar a 2D o 3D para visualizar) y **preprocesamiento** para reducir ruido y riesgo de sobreajuste.
- **Limitaciones:**
  - Supone relaciones **lineales**.
  - Asume que la **varianza** mide la importancia: puede descartar direcciones de poca varianza pero relevantes.
  - Los componentes son combinaciones de *features* originales y son **difíciles de interpretar**.

#### Modelado de temas: LDA y NTM

El modelado de temas es **no supervisado**. Descubre los **temas subyacentes** de una colección de documentos a partir de los **patrones de co-ocurrencia de palabras**, sin etiquetas previas. Es análogo al *clustering*: agrupa documentos por su temática. Resulta muy útil para explorar grandes volúmenes de texto no estructurado. SageMaker ofrece **LDA** y **NTM** listos para usar.

##### Latent Dirichlet Allocation (LDA)

Modelo **probabilístico generativo** con dos supuestos:

- Cada **documento es una mezcla de temas**.
- Cada **tema es una mezcla de palabras**.

La **distribución de Dirichlet** modela la incertidumbre sobre las proporciones de temas de cada documento, que siempre suman 1. Su forma se controla con **parámetros de concentración**, que determinan cómo se reparte la probabilidad entre temas. Actualizando iterativamente las distribuciones «temas por documento» y «palabras por tema», LDA descubre los temas que mejor representan el texto.

En SageMaker se especifica el **número de temas**, entre otros hiperparámetros. El modelo entrenado **transforma documentos nuevos** en su distribución de temas.

**Casos de uso:** agrupar noticias por tema (política, deportes, tecnología) para navegar por temática, detectar temas y tendencias en artículos científicos, *clustering* de documentos, recomendación de contenido y recuperación de información.

**Limitación:** el supuesto de temas con distribución de Dirichlet puede no captar dependencias más complejas entre palabras y temas. Ahí entra NTM.

##### Neural Topic Model (NTM)

NTM aprende las distribuciones de temas con **redes neuronales**, en lugar de fijarlas mediante una Dirichlet como LDA. Eso le da más **flexibilidad** para capturar patrones complejos en corpus grandes. Se usa igual que LDA: se entrena y luego transforma documentos nuevos en distribuciones de temas.

> **Corrección:** el libro atribuye a NTM la capacidad de captar «dependencias temporales o secuenciales» del texto. Es falso: igual que LDA, NTM trabaja sobre **bolsas de palabras** (conteos por documento) y **no modela el orden** de las palabras.

**Cuándo usarlo.**

- **Úsalo** con corpus grandes y complejos donde la relación palabra-tema no encaja bien en una Dirichlet. Ejemplos: análisis de *feedback* de clientes, exploración temática de grandes colecciones de texto.
- **Evítalo** con datasets pequeños o recursos limitados, donde LDA es más simple, barato y suficiente. Tampoco aporta mucho si los temas ya están bien definidos o el texto es muy estructurado.

#### Detección de anomalías: Random Cut Forest e IP Insights

La detección de anomalías es no supervisada: identifica valores atípicos **sin ejemplos etiquetados de anomalías**. Resulta útil cuando las etiquetas escasean o las anomalías son raras e impredecibles. SageMaker ofrece dos algoritmos integrados, ambos aptos para detección en tiempo real:

- **Random Cut Forest (RCF):** anomalías en datos generales.
- **IP Insights:** uso anómalo de direcciones IPv4.

##### Random Cut Forest (RCF)

RCF construye un **bosque de árboles aleatorios**, cada uno sobre una muestra aleatoria de los datos de entrenamiento. Cada árbol asigna a un punto una **puntuación de anomalía**: el cambio esperado en la complejidad del árbol al añadir ese punto. Ese cambio es aproximadamente **inversamente proporcional a la profundidad** del punto en el árbol. La puntuación final es la media de las de todos los árboles.

> **Corrección:** el libro dice que se marcan como anomalías los puntos que quedan «inusualmente profundos». Es al revés. Un punto anómalo, lejos del resto, queda **aislado con pocos cortes, cerca de la raíz** (poca profundidad), y por eso recibe una **puntuación alta**. Los puntos normales quedan más profundos.

Los hiperparámetros clave son `num_trees` (número de árboles) y `num_samples_per_tree` (muestras por árbol). El modelo entrenado se despliega en un endpoint para detectar anomalías en tiempo real.

**Cuándo usarlo.**

- **Úsalo** con datos **grandes y de alta dimensión**: fraude en transacciones, intrusiones en tráfico de red, **mantenimiento predictivo** (datos de sensores que anticipan fallos).
- **Evítalo** si las anomalías ya están **etiquetadas**, porque un clasificador supervisado será más preciso, o con datasets muy pequeños, donde el muestreo aleatorio puede perder patrones y bastan técnicas estadísticas simples.

##### IP Insights

IP Insights aprende los **patrones de uso de direcciones IPv4** asociándolas a **entidades** (IDs de usuario, números de cuenta):

- Una red neuronal aprende **representaciones latentes (embeddings)** de entidades e IPs, reutilizables en otras tareas.
- Consultado con un par **(entidad, IPv4)**, devuelve una puntuación de **lo anómalo que es** ese uso.

Los datos se preparan como pares (entidad, IPv4). Hiperparámetros como el número de vectores de entidad y el tamaño del embedding. Se despliega en endpoint o se usa por lotes.

**Cuándo usarlo.**

- **Úsalo** para detectar **inicios de sesión fraudulentos**, patrones de acceso inusuales, **cuentas comprometidas** o creación de recursos desde IPs anómalas (p. ej., en *e-commerce* y servicios online).
- **Evítalo** cuando las IPs cambian con frecuencia o las entidades no tienen patrones estables, porque el modelo depende de asociaciones estables; ahí rinden mejor sistemas de reglas u otros detectores. Si hay datos etiquetados, un enfoque supervisado será más preciso.

### 2.3 Análisis de texto: BlazingText y Seq2Seq

El análisis de texto transforma texto no estructurado en información estructurada útil para decidir: clasificación, sentimiento, entidades, temas, resúmenes. Ya se vieron **Comprehend** (servicio gestionado) y **LDA/NTM** (temas). Aquí se ven dos algoritmos integrados más: **BlazingText** y **Sequence-to-Sequence**.

#### BlazingText

Implementación muy optimizada y escalable de dos técnicas:

| Técnica | Tipo | Qué hace | `mode` |
|---|---|---|---|
| **Word2Vec** | No supervisado | Aprende **embeddings de palabras** a partir de grandes corpus, capturando relaciones semánticas y sintácticas (similitudes, analogías) | `cbow`, `skipgram`, `batch_skipgram` |
| **Clasificación de texto** | Supervisado | Clasifica textos en categorías predefinidas | `supervised` |

> **Corrección:** el clasificador de BlazingText no es un modelo de *deep learning* complejo. **Extiende el clasificador de fastText**, un modelo ligero, y por eso es tan rápido.

Usa **multihilo y aceleración por GPU** para entrenar rápido con grandes volúmenes. Es útil para similitud semántica, sentimiento y clasificación de documentos.

> **Embedding:** representación **vectorial densa** de datos, normalmente palabras, que captura su significado y sus relaciones. Convierte datos categóricos en numéricos continuos. En los embeddings de palabras, las palabras con significado o uso parecido quedan **cerca** en el espacio vectorial: la similitud semántica se traduce en proximidad. Técnicas comunes: **Word2Vec, GloVe, FastText**.

```python
import sagemaker

sagemaker_session = sagemaker.Session()
role = '<your-iam-role>'
container = sagemaker.image_uris.retrieve('blazingtext', sagemaker_session.boto_region_name)

bt_estimator = sagemaker.estimator.Estimator(
    container, role, instance_count=1, instance_type='ml.c4.2xlarge',
    output_path='s3://<your-bucket>/output', sagemaker_session=sagemaker_session)

# Argumentos con nombre (**kwargs), no un diccionario.
# mode='supervised' → clasificación; 'cbow' | 'skipgram' | 'batch_skipgram' → Word2Vec
bt_estimator.set_hyperparameters(mode='supervised', epochs=10, min_count=2, learning_rate=0.05)

bt_estimator.fit({'train': 's3://<your-bucket>/train'})
predictor = bt_estimator.deploy(initial_instance_count=1, instance_type='ml.m4.xlarge')
```

El **modo de operación** se fija en `set_hyperparameters()`, y los hiperparámetros válidos dependen del modo: Word2Vec o clasificación de texto.

> **Corrección:** el libro dice que `set_hyperparameters()` recibe «un diccionario como único parámetro». En realidad recibe **argumentos con nombre** (`**kwargs`), como en el código.

**Cuándo usarlo.**

- **Úsalo** para clasificación de texto o embeddings **eficientes y escalables** sobre grandes volúmenes: reseñas, redes sociales, categorización de documentos. Word2Vec sirve para similitud semántica, *clustering* de palabras y *features* de tareas NLP posteriores.
- **Evítalo** en tareas que necesitan **contexto más allá de la palabra**. Para embeddings contextuales de frases o párrafos (búsqueda semántica, RAG) son más adecuados los **modelos de embeddings de Bedrock**, como `cohere.embed-english-v3` y `cohere.embed-multilingual-v3`. Tampoco sirve para personalizaciones fuera de la clasificación y los embeddings.

> **Corrección:** el libro sugiere esos modelos de embeddings para *question answering* o IA conversacional. Un modelo de embeddings **no responde preguntas ni conversa**: para eso se necesita un FM de **generación de texto**, al que los embeddings pueden alimentar vía RAG.

#### Sequence-to-Sequence (Seq2Seq)

Algoritmo **supervisado** para tareas donde la **entrada es una secuencia de tokens** (texto o audio) y la **salida es otra secuencia**. Sus usos típicos son la **traducción automática**, el **resumen de textos** y la conversión de **voz a texto**. Usa arquitecturas codificador-decodificador con **RNN y CNN con mecanismos de atención**.

**Cuándo usarlo.**

- **Úsalo** en traducción, resumen, *question answering* o generación de respuestas de chatbot: cualquier tarea que transforme o genere secuencias.
- **Evítalo** en tareas que no generan secuencias:
  - La clasificación simple de texto, como el sentimiento de un tuit, es más eficiente con clasificadores tradicionales o CNN.
  - Las tareas de *features* estáticas (clasificación de imágenes, predicción de series sin salida secuencial) tienen algoritmos más apropiados.

### 2.4 Procesamiento de imágenes

SageMaker ofrece algoritmos integrados de visión:

- **Image Classification:** clasifica la imagen completa.
- **Object Detection:** identifica y localiza objetos.
- **Semantic Segmentation:** clasifica cada píxel.

Además, JumpStart ofrece modelos preentrenados de **embeddings de imagen**, que convierten imágenes en vectores de tamaño fijo para tareas posteriores. El libro los presenta como algoritmo integrado, pero no lo son.

#### Image Classification

Asigna **una etiqueta a la imagen completa** («gato», «perro», «pájaro»…). Es **supervisado** y se basa en **CNN**, que aprenden jerarquías espaciales de *features* mediante capas de convolución, *pooling* y capas totalmente conectadas. Durante el entrenamiento minimiza el error de clasificación sobre muchas imágenes etiquetadas.

> **Corrección:** el algoritmo integrado tiene **dos variantes**, no tres:
> - **Image Classification – MXNet**.
> - **Image Classification – TensorFlow**, con *transfer learning* sobre modelos preentrenados de TensorFlow Hub.
>
> No existe una variante integrada de PyTorch.

**Cuándo usarlo.**

- **Úsalo** para categorizar imágenes por su contenido: radiografías o resonancias (detección de enfermedades), organización de productos en *retail*, reconocimiento de caras, matrículas u objetos de interés en seguridad.
- **Evítalo** si hay que **localizar** varios objetos (detección), **segmentar** regiones (segmentación semántica), extraer información estructurada como **OCR** (Textract) o cuando el objetivo no es clasificar.

#### Object Detection

Además de identificar los objetos de la imagen, los **localiza con cajas delimitadoras** (*bounding boxes*): clasificación y localización a la vez. Es supervisado y necesita imágenes anotadas con la clase y la caja de cada objeto.

> **Corrección:** el algoritmo integrado no implementa R-CNN ni YOLO. Tiene dos variantes:
> - **Object Detection – MXNet:** usa **SSD** (*Single Shot MultiBox Detector*) con una red base **VGG-16 o ResNet-50**, preentrenada en ImageNet o desde cero.
> - **Object Detection – TensorFlow:** hace *transfer learning* con modelos preentrenados de TensorFlow.
>
> R-CNN y YOLO son arquitecturas populares en general; en SageMaker aparecen como modelos preentrenados de JumpStart.

**Cómo funciona SSD.** Usa como base una CNN preentrenada para clasificación. Para cada posición de sus mapas de *features* predice varias cajas y las probabilidades de clase de su contenido. Durante el entrenamiento minimiza el error respecto a las cajas y clases reales. La implementación de SageMaker incluye **aumento de datos** (volteo, reescalado, *jittering*) para ganar robustez y evitar el sobreajuste.

**Cuándo usarlo.**

- **Úsalo** cuando importan tanto la **identificación como la localización**: conducción autónoma (peatones, vehículos, obstáculos), seguridad y vigilancia (caras, vehículos, bolsos en tiempo real), *retail* (contar productos en estanterías, seguir movimientos de clientes), imagen médica (localizar anomalías) e ISR militar (inteligencia, vigilancia y reconocimiento).
- **Evítalo** si basta con clasificar la imagen completa (Image Classification), si hay que segmentar regiones con precisión de píxel (segmentación semántica) o con datos no visuales.

#### Semantic Segmentation

Asigna una **clase a cada píxel** de la imagen. Da la comprensión más detallada de las tres tareas:

| Tarea | Qué produce |
|---|---|
| Image Classification | Una etiqueta para toda la imagen |
| Object Detection | Cajas delimitadoras + clase de cada objeto |
| **Semantic Segmentation** | **Mapa de segmentación**: una clase por píxel |

Parte de una red de clasificación preentrenada (*backbone* **ResNet-50 o ResNet-101**) modificada para producir mapas de segmentación en lugar de probabilidades de clase. Aprende a asignar etiquetas a nivel de píxel con una función de pérdida entre etiquetas predichas y reales. Usa ***upsampling*** y ***skip connections*** para recuperar resolución espacial y bordes finos. Necesita, por cada imagen de entrenamiento, una **máscara** que etiquete cada píxel.

> **Corrección:** en SageMaker el hiperparámetro `algorithm` admite **`fcn`** (por defecto), **`psp`** y **`deeplab`** (DeepLabV3). **U-Net no es una opción**.

**Cuándo usarlo.**

- **Úsalo** cuando se necesita precisión **a nivel de píxel**: imagen médica (tejidos, órganos, tumores en RM/TC), conducción autónoma (carretera, acera, vehículos, peatones), agricultura (cultivos frente a malas hierbas, salud de las plantas) e imágenes satelitales (bosques, zonas urbanas, agua).
- **Evítalo** si basta con clasificar la imagen o con cajas delimitadoras, que no requieren anotación por píxel. Tampoco si el coste computacional es prohibitivo o hay **pocos datos etiquetados**, porque necesita muchas imágenes anotadas.

---

## 3. Criterios para seleccionar un modelo

| Criterio | Qué valorar | Ejemplos del capítulo |
|---|---|---|
| **Exactitud** | Objetivo principal de cualquier modelo | XGBoost y SVM destacan en clasificación y regresión |
| **Interpretabilidad** | Esencial donde hay que entender y justificar las decisiones | Linear Learner y regresión logística; los árboles de decisión mapean visualmente el proceso |
| **Escalabilidad** | Capacidad de manejar volúmenes crecientes de datos | Linear Learner, K-means; RCF para anomalías en *big data* |
| **Latencia y velocidad** | Tiempo de inferencia en aplicaciones en tiempo real | Modelos ligeros, p. ej., *random forest* con un número moderado de árboles (fraude, recomendaciones) |
| **Recursos** | Cómputo para entrenamiento e inferencia, frente a infraestructura y presupuesto | Linear Learner y PCA requieren menos que las redes profundas. k-NN apenas cuesta entrenarlo, pero su inferencia es costosa en memoria y cómputo con muchos datos |
| **Disponibilidad y calidad de datos** | Cantidad y calidad de los datos | LDA y BlazingText rinden con grandes corpus; K-means funciona con datasets más pequeños |
| **Regulación y ética** | Cumplimiento normativo y consecuencias de las predicciones (salud, finanzas) | Regresión logística y árboles de decisión, por su transparencia |
| **Coste** | Implementación y mantenimiento, incluidos cloud y etiquetado de datos | Linear Learner es barato; las CNN son costosas por su demanda computacional |

Estos criterios suelen estar **en tensión**. El caso típico es exactitud frente a interpretabilidad: los modelos complejos y precisos son difíciles de entender, y los simples e interpretables pueden ser menos precisos. El trabajo del ingeniero de ML en AWS es **equilibrar estos compromisos** según los objetivos de negocio y las restricciones operativas, y reevaluar el modelo con datos y *feedback* nuevos para mantener rendimiento, transparencia y confianza.

---

## 4. Resumen

- **Servicios de IA de AWS:** modelos preentrenados y totalmente gestionados, ideales para integrar IA rápido y con poca experiencia en ML:
  - Rekognition (imagen y vídeo), Textract (documentos), Polly (TTS), Transcribe (ASR).
  - Translate (traducción), Comprehend (NLP), Lex (chatbots), Personalize (recomendaciones).
  - Bedrock (IA generativa con FMs, incluidos los Nova).
- **SageMaker AI:** más flexibilidad para tareas especializadas, con algoritmos integrados y soporte para *frameworks* como TensorFlow, PyTorch y MXNet.
  - **Supervisados:** Linear Learner (regresión y clasificación), XGBoost (*boosting* de árboles), k-NN (clasificación y regresión), Factorization Machines (recomendación y clasificación a gran escala), Object2Vec (embeddings) y DeepAR (series temporales).
  - **No supervisados:** K-means (*clustering*), PCA (reducción de dimensionalidad), LDA y NTM (temas), RCF e IP Insights (anomalías).
  - **Redes profundas para texto e imagen:** BlazingText (embeddings y clasificación de texto), Seq2Seq (traducción, resumen), Image Classification, Object Detection y Semantic Segmentation.
- **Criterios de selección:** exactitud, interpretabilidad, escalabilidad, latencia, recursos, datos, regulación y coste.

---

## 5. Puntos clave para el examen

- **Servicios de IA para visión:**
  - **Rekognition** analiza imágenes y vídeo: objetos, rostros, texto, escenas, celebridades y análisis facial (incluidas emociones).
  - **Textract** extrae texto, tablas y formularios de documentos escaneados.
- **Voz y chatbots:**
  - **Polly** = texto a voz.
  - **Transcribe** = voz a texto (ASR).
  - **Lex** = interfaces conversacionales (ASR + NLU) para chatbots y asistentes de voz.
- **Lenguaje:**
  - **Comprehend** = NLP (sentimiento, entidades, frases clave); para datos clínicos, **Comprehend Medical**.
  - **Translate** = traducción en tiempo real y por lotes.
- **IA generativa:** **Bedrock** ofrece FMs de varios proveedores y de Amazon (Titan, Nova) para generar texto, imágenes y embeddings. Si el modelo no está en tu región, usa **inferencia entre regiones** con un perfil de inferencia.
- **Clasificación frente a regresión:** la clasificación asigna clases discretas (¿es spam?); la regresión predice valores continuos (precio de una vivienda).
- **Linear Learner:** un único algoritmo integrado que, según la función de pérdida, hace regresión lineal, regresión logística o **SVM lineal**. Es muy interpretable: se ve la contribución de cada *feature*.
- **k-NN:** algoritmo integrado, supervisado, para clasificación y regresión. Predice según los vecinos más cercanos, no supone una forma de frontera (capta fronteras no lineales) y es fácil de explicar. Úsalo con datasets pequeños o moderados y de pocas dimensiones.
- **Árboles en SageMaker:**
  - El algoritmo integrado de árboles es **XGBoost** (también LightGBM y CatBoost): *boosting* secuencial, alta exactitud y eficiencia en datos tabulares grandes.
  - **Random Forest no es un algoritmo integrado.** Promedia árboles independientes para reducir el sobreajuste; se usa con tu propio script, p. ej., scikit-learn.
- **Clustering:** **K-means** es no supervisado. **No lo confundas con k-NN**, que es supervisado.
- **Reducción de dimensionalidad:** **PCA** es no supervisado y reduce *features* conservando la máxima varianza.
- **Modelado de temas:** **LDA** y **NTM** son no supervisados y descubren temas en texto sin etiquetas.
- **Anomalías:** **Random Cut Forest** (anomalías generales, fraude, intrusiones; alta puntuación = punto aislado a poca profundidad) e **IP Insights** (uso anómalo de IPv4 por entidad).
- **Texto:** **BlazingText** (Word2Vec y clasificación de texto a gran escala) y **Seq2Seq** (traducción, resumen, voz a texto).
- **Imagen:** **Image Classification** (imagen completa), **Object Detection** (cajas delimitadoras) y **Semantic Segmentation** (cada píxel).

---

## 6. Preguntas de repaso

**1.** ¿Qué servicio de IA de AWS usarías para analizar grandes volúmenes de texto no estructurado y extraer información como entidades y sentimiento?
- A. Amazon Textract
- B. Amazon Lex
- C. Amazon Comprehend
- D. Amazon Polly

**2.** ¿Qué servicio de IA de AWS está diseñado específicamente para aplicaciones de IA generativa y ofrece una plataforma para crear texto, imágenes y otros contenidos?
- A. Amazon Rekognition
- B. Amazon Bedrock
- C. Amazon Translate
- D. Amazon Transcribe

**3.** Para detectar objetos y personas en vídeo en tiempo real, ¿qué servicio de IA de AWS es el más apropiado?
- A. Amazon Textract
- B. Amazon Lex
- C. Amazon Rekognition
- D. Amazon Comprehend

**4.** ¿Qué servicio de AWS ofrece una solución eficiente y escalable para reducir la dimensionalidad de grandes datasets?
- A. Amazon Translate
- B. Amazon Polly
- C. Principal Component Analysis (PCA) en Amazon SageMaker
- D. K-Means en Amazon SageMaker

**5.** ¿Qué algoritmo de Amazon SageMaker es especialmente útil para clasificación y regresión por su eficiencia y alto rendimiento?
- A. Random Forest
- B. XGBoost
- C. k-Nearest Neighbors
- D. Principal Component Analysis

**6.** ¿Qué algoritmo supervisado de Amazon SageMaker es ideal para reducir el sobreajuste promediando múltiples árboles de decisión?
- A. Linear Learner
- B. BlazingText
- C. Random Forest
- D. Latent Dirichlet Allocation

**7.** ¿Para qué tipo de tareas es especialmente adecuado el algoritmo Linear Learner de Amazon SageMaker?
- A. Clasificación de texto
- B. Clustering
- C. Clasificación y regresión
- D. Detección de anomalías

**8.** ¿Qué algoritmo de Amazon SageMaker usarías si necesitas resultados muy interpretables en un modelo de relación lineal?
- A. Random Cut Forest
- B. Linear Learner
- C. Neural Topic Model
- D. DeepAR

**9.** ¿Qué algoritmo supervisado de Amazon SageMaker está diseñado específicamente para el pronóstico de series temporales?
- A. IP Insights
- B. DeepAR
- C. Neural Topic Model
- D. Sequence-to-Sequence

**10.** ¿Qué algoritmo supervisado de Amazon SageMaker es conocido por combinar (*boosting*) aprendices débiles para crear un modelo predictivo fuerte?
- A. K-Means
- B. Latent Dirichlet Allocation
- C. XGBoost
- D. Random Cut Forest

**11.** Necesitas un algoritmo no supervisado y muy interpretable en Amazon SageMaker para reducir la dimensionalidad preservando la máxima varianza. ¿Qué algoritmo integrado usarías?
- A. K-Means
- B. Random Cut Forest
- C. Principal Component Analysis
- D. Neural Topic Model

**12.** Para detectar eventos raros y anomalías con alta exactitud en flujos de datos, ¿qué algoritmo no supervisado de Amazon SageMaker elegirías?
- A. Latent Dirichlet Allocation
- B. IP Insights
- C. Random Cut Forest
- D. Factorization Machines

**13.** Si necesitas descubrir temas ocultos en grandes datasets de texto con alta interpretabilidad y exactitud, ¿qué algoritmo de Amazon SageMaker seleccionarías?
- A. K-Means
- B. Latent Dirichlet Allocation
- C. Principal Component Analysis
- D. Random Cut Forest

**14.** Para agrupar con exactitud grandes datasets en grupos predefinidos según la similitud de sus *features*, ¿qué algoritmo muy interpretable de Amazon SageMaker usarías?
- A. Random Cut Forest
- B. Principal Component Analysis
- C. K-Means
- D. Neural Topic Model

**15.** ¿Qué algoritmo de Amazon SageMaker destaca por su exactitud y rendimiento en clasificación de texto a gran escala, siendo además rentable?
- A. BlazingText
- B. Sequence-to-Sequence
- C. Latent Dirichlet Allocation
- D. IP Insights

**16.** Identifica el algoritmo de Amazon SageMaker que ofrece alto rendimiento y exactitud en tareas de traducción de texto, siendo rentable e interpretable.
- A. Random Cut Forest
- B. Sequence-to-Sequence
- C. BlazingText
- D. Principal Component Analysis

**17.** Para identificar y localizar con alta exactitud múltiples objetos en una imagen, ¿qué algoritmo de Amazon SageMaker destaca en rendimiento y eficiencia de costes?
- A. Image Classification
- B. Object Detection
- C. Semantic Segmentation
- D. Factorization Machines

**18.** ¿Qué algoritmo de Amazon SageMaker ofrece alta exactitud y rentabilidad para clasificar imágenes en categorías predefinidas, garantizando interpretabilidad y rendimiento?
- A. Latent Dirichlet Allocation
- B. Image Classification
- C. Object Detection
- D. IP Insights

**19.** Identifica el algoritmo de Amazon SageMaker que permite un análisis detallado de imágenes a nivel de píxel, con alta exactitud e interpretabilidad y un rendimiento eficiente.
- A. Random Cut Forest
- B. Image Classification
- C. Semantic Segmentation
- D. BlazingText

**20.** ¿Qué algoritmo de Amazon SageMaker usa embeddings de palabras para tareas de NLP, equilibrando exactitud, rentabilidad, interpretabilidad y rendimiento?
- A. Random Cut Forest
- B. Principal Component Analysis
- C. Latent Dirichlet Allocation
- D. BlazingText

### Respuestas

| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 | 18 | 19 | 20 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| C | B | C | C | B | C | C | B | B | C | C | C | B | C | A | B | B | B | C | D |

> **Notas sobre las preguntas:**
> - **P6:** la respuesta que espera el libro es C (el concepto de promediar árboles), pero la premisa es incorrecta: Random Forest **no es un algoritmo integrado de SageMaker**. En un examen real, una opción «Random Forest integrado» sería sospechosa.
> - **P11:** PCA es la respuesta correcta, pero sus componentes **no son muy interpretables**, como se explica en la sección de PCA. Lo que decide la respuesta es «no supervisado + máxima varianza».
