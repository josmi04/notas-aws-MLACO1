# Capítulo 5 — Entrenamiento y evaluación de modelos (enfoque AWS)

> Objetivos del examen — **Dominio 2: Desarrollo de modelos de ML**
> - 2.2 Entrenar y refinar modelos
> - 2.3 Analizar el desempeño del modelo

Amazon SageMaker ofrece soluciones robustas para la optimización de hiperparámetros, lo que permite a ingenieros de ML y científicos de datos buscar de forma eficiente las mejores configuraciones. También ofrece un conjunto completo de herramientas para evaluar modelos: validación cruzada exhaustiva, monitoreo del desempeño y ajuste fino del modelo a partir de lo que se observa. El capítulo cierra con la evaluación de modelos fundacionales en **Amazon Bedrock**: cómo evaluar modelos fundacionales preentrenados para asegurar que cumplan con los requisitos específicos de tus casos de uso.

---

## 1. Entrenamiento

Hay tres enfoques para entrenar modelos:

- **Entrenamiento local**: se ejecuta en una sola máquina usando sus recursos disponibles; da retroalimentación inmediata y facilita la depuración.
- **Entrenamiento remoto**: se ejecuta en un servicio en la nube (p. ej., **Amazon SageMaker**) o en un servidor remoto, aprovechando cómputo externo sin sobrecargar el sistema local ni depender de las limitaciones del hardware local.
- **Entrenamiento distribuido**: reparte la carga de entrenamiento entre varias máquinas o nodos, ya sea local o remotamente, para manejar conjuntos de datos más grandes y reducir el tiempo de entrenamiento paralelizando los cálculos.

### 1.1 Entrenamiento local

Con frecuencia los conjuntos de datos son demasiado grandes para caber en la memoria de la máquina. El entrenamiento local permite probar el algoritmo elegido con un **subconjunto de los datos**, controlando el entorno de entrenamiento y facilitando pruebas y ajustes rápidos de algoritmos, combinaciones de features e hiperparámetros.

**Scripts en lugar de notebooks.** Aunque Amazon SageMaker ofrece una interfaz de **Jupyter Notebook** para desarrollar modelos, se prefieren los scripts de Python. Los notebooks pueden **escalar verticalmente pero no horizontalmente** (*scale up but not out*), es decir, no sirven para entrenamiento distribuido en varias máquinas. Además, es difícil versionarlos bien, a diferencia de los scripts de Python, que se manejan fácilmente con sistemas de control de versiones como **Git**. Esto da un proceso de desarrollo más robusto y escalable, y facilita la colaboración y la reproducibilidad.

**Contenedores.** El entrenamiento local también permite empaquetar los programas de Python y sus dependencias en **contenedores**. Así el entorno de desarrollo es consistente entre máquinas y etapas del proyecto, y se eliminan los problemas de manejo de dependencias. Los contenedores dan un entorno portable y reproducible, lo que facilita compartir y desplegar modelos en distintos entornos.

> **Runtimes de contenedores en SageMaker.** **Docker** es el runtime de contenedores más usado para consumir imágenes de algoritmos de ML en Amazon SageMaker, pero no es la única opción. Como Amazon SageMaker usa **Amazon ECR** para obtener las imágenes de contenedor, puedes usar cualquier runtime que produzca imágenes **compatibles con OCI (Open Container Initiative)**, ya que Amazon ECR soporta el estándar OCI. Esto da flexibilidad para elegir un runtime como **Containerd** o **CRI-O**, aunque Docker es el más adoptado y el enfoque recomendado.
> Más información: https://aws.amazon.com/about-aws/whats-new/2024/06/amazon-ecr-oci-image-distribution-version-1-1

Los contenedores también simplifican el escalado y el despliegue. Una vez que el modelo se entrena localmente y sus dependencias quedan empaquetadas en un contenedor, se puede desplegar y escalar fácilmente en otros entornos, como **Amazon Elastic Kubernetes Service (EKS)**. El modelo se ejecuta de forma consistente sin importar la infraestructura subyacente, lo que mejora la confiabilidad y la escalabilidad. El entrenamiento local combinado con contenedores es un flujo de trabajo potente para desarrollar, probar y desplegar modelos de ML de forma eficiente.

### 1.2 Entrenamiento remoto

Después de entrenar el modelo localmente con un conjunto pequeño para desarrollo y pruebas, el siguiente paso es **lanzar un training job con el conjunto de datos completo**. El entrenamiento remoto permite aprovechar recursos de cómputo más potentes y manejar conjuntos de datos más grandes de forma eficiente, para que el modelo aprenda patrones más completos.

Con el **SDK de Python de Amazon SageMaker** puedes lanzar un training job remoto con unas cuantas líneas de código. El SDK simplifica la configuración y la gestión de los training jobs, así que los desarrolladores se pueden enfocar en sus modelos y no en la infraestructura. Puedes especificar el **tipo y número de instancias** que quieres usar, para que el job tenga los recursos necesarios.

Como es un **servicio totalmente administrado**, Amazon SageMaker se encarga de aprovisionar, escalar y administrar las instancias de entrenamiento por ti. Esto elimina la necesidad de configurar y mantener la infraestructura manualmente. SageMaker se encarga de todo, desde la carga y el preprocesamiento de datos hasta el entrenamiento y la evaluación del modelo.

Ejemplo — se crea una instancia de **estimator** y se usa el método **`fit`** para iniciar el training job:

```python
import sagemaker
from sagemaker.estimator import Estimator

# Define the IAM role with necessary permissions
role = 'arn:aws:iam::your-account-id:role/SageMakerRole'

# Create the estimator instance
estimator = Estimator(
    image_uri='your-docker-image-uri',
    role=role,
    instance_count=1,
    instance_type='ml.m5.large',
    volume_size=50,  # Size in GB
    max_run=3600,    # Maximum training time in seconds
    hyperparameters={
        'epochs': 10,
        'batch_size': 32,
        'learning_rate': 0.001,
    }
)

# Launch the training job
estimator.fit({'train': 's3://your-bucket/train-data',
               'validation': 's3://your-bucket/validation-data'})
```

La configuración del training job incluye la **URI de la imagen de Docker**, el **rol de IAM**, el **tipo de instancia** y los **hiperparámetros**. El método `fit` inicia el training job, apuntando a los **buckets de S3** donde están los datos de entrenamiento y validación. Con SageMaker puedes pasar sin fricción del desarrollo local a un entrenamiento remoto escalable y confiable sobre el conjunto de datos completo.

---

## 2. Monitoreo de training jobs

Como servicio totalmente administrado, Amazon SageMaker ofrece herramientas para monitorear y visualizar métricas de entrenamiento **en tiempo real**. Gracias a su integración con **Amazon CloudWatch**, puedes rastrear métricas como la **pérdida de entrenamiento (training loss), la exactitud (accuracy) y el uso de recursos**. Estas métricas ayudan a diagnosticar problemas, ajustar hiperparámetros y asegurar que el modelo esté aprendiendo bien.

Para empezar a monitorear un training job, especificas las métricas que quieres rastrear desde la **AWS Management Console** o con el **SDK de Python de Amazon SageMaker**. Cuando empieza el training job, SageMaker **envía automáticamente** las métricas especificadas a CloudWatch, donde puedes visualizarlas como **series de tiempo**. Esto permite tomar decisiones informadas durante el entrenamiento, como ajustar hiperparámetros o **detener un job** si no está funcionando como se esperaba. Las métricas también se pueden consultar **de forma programática**, lo que permite flujos de trabajo automatizados e integración con otras herramientas de monitoreo.

**Amazon EventBridge** ayuda a responder de forma proactiva a **training jobs fallidos**. Al integrar EventBridge con tus flujos de entrenamiento, puedes configurar **reglas y alarmas** que detecten cuando un training job falla. Por ejemplo, puedes configurar EventBridge para monitorear métricas de CloudWatch de training jobs fallidos y disparar respuestas automáticas, como:

- enviar notificaciones a tu equipo,
- reiniciar el training job,
- iniciar un rollback a una versión anterior del modelo.

EventBridge también soporta **dead-letter queues (DLQs)** para capturar eventos fallidos y reintentos, de modo que tengas un registro de cualquier problema y puedas actuar en consecuencia.

**Al terminar el job**, SageMaker proporciona logs y métricas detallados. Puedes ver gráficas de las métricas en CloudWatch y obtener los **valores finales de las métricas** llamando a la operación **`DescribeTrainingJob`**. Este análisis posterior ayuda a evaluar el desempeño del modelo e identificar áreas de mejora.

En conjunto, **SageMaker + CloudWatch + EventBridge** forman una solución completa para rastrear y optimizar flujos de trabajo de ML.

---

## 3. Depuración de training jobs

Depurar training jobs es complicado por la escala de los modelos y conjuntos de datos modernos: problemas de calidad de datos, modelos que no convergen, cuellos de botella de recursos. La naturaleza iterativa del entrenamiento (muchas épocas e iteraciones) puede ocultar el origen de los problemas, y se necesitan logs y métricas detallados para rastrearlos.

**Amazon SageMaker Debugger** ofrece funciones de depuración y de profiling:

**Depuración.** **Captura y guarda automáticamente tensores y métricas intermedios** durante el entrenamiento. Esto permite inspeccionar el estado interno del modelo en distintos momentos, lo que facilita identificar y diagnosticar problemas como **vanishing gradients** (gradientes que se desvanecen) y **preprocesamiento de datos incorrecto**.

**Profiling.** Permite a los ingenieros de ML y a los equipos de operaciones monitorear y analizar el desempeño de los training jobs capturando **métricas detalladas de uso de recursos del sistema: CPU, GPU, memoria y red**. Con ellas se pueden identificar **cuellos de botella de desempeño** y optimizar los jobs. SageMaker Debugger ofrece **reglas integradas (built-in rules)** que detectan automáticamente problemas de desempeño comunes, y proporciona **visualizaciones y reportes** para entenderlos y resolverlos. Así se aprovechan de forma óptima los recursos de entrenamiento, lo que **reduce costos y tiempos de entrenamiento**.

**`smdebug`.** SageMaker Debugger soporta la biblioteca de código abierto **smdebug**, que extiende la depuración y el profiling a flujos de entrenamiento personalizados y complejos. Permite **definir reglas personalizadas** y **guardar tensores, métricas y otros datos específicos** durante el entrenamiento. Se integra con **TensorFlow, PyTorch y MXNet**, así que los ingenieros de ML pueden armar flujos de depuración y profiling a la medida.

---

## 4. Ajuste de hiperparámetros con Amazon SageMaker

Con Amazon SageMaker puedes ajustar hiperparámetros de forma **manual** o **automática** con **Amazon SageMaker AI Automatic Model Tuning (AMT)**:

- **Ajuste manual**: eliges distintos valores de hiperparámetros, corres varios training jobs y evalúas su desempeño para identificar la mejor combinación. Consume mucho tiempo y recursos.
- **AMT**: usa algoritmos de optimización para buscar automáticamente los mejores hiperparámetros. **Evalúa varias configuraciones en paralelo**, lo que reduce mucho el tiempo y el esfuerzo necesarios, y logra un desempeño óptimo con mínima intervención manual.

### 4.1 Explorar el espacio de hiperparámetros con AMT

El **espacio de hiperparámetros** es el rango de valores que pueden tomar los distintos hiperparámetros. Explorarlo es clave para encontrar la mejor combinación.

AMT automatiza el ajuste de hiperparámetros con algoritmos de optimización como **optimización bayesiana, búsqueda aleatoria (random search) y búsqueda en malla (grid search)**. Te permite **definir rangos** para los hiperparámetros y evalúa automáticamente varias combinaciones en paralelo.

Además, los **hyperparameter tuning jobs de Amazon SageMaker** dan una forma estructurada de configurar y lanzar experimentos de ajuste. Puedes especificar:

- los **rangos de hiperparámetros**,
- la **métrica objetivo** (*objective metric*),
- los **límites de recursos** del tuning job.

SageMaker también soporta **warm starts**, que permiten usar lo aprendido en **tuning jobs anteriores** para guiar búsquedas posteriores, lo que hace la optimización más eficiente.

> **Hiperparámetros de los algoritmos integrados.** Cada algoritmo integrado (built-in) de SageMaker tiene sus propios hiperparámetros. Para verlos, entra a https://docs.aws.amazon.com/sagemaker/latest/dg/algos.html, ve a la columna "Built-in algorithms", localiza tu algoritmo, entra al link y busca la sección de hiperparámetros. Ejemplo (DeepAR): https://docs.aws.amazon.com/sagemaker/latest/dg/deepar_hyperparameters.html

### 4.2 La métrica objetivo

Lo que "óptimo" significa para AMT lo define la **métrica de evaluación** con la que se mide el desempeño del modelo, que depende del tipo de problema de ML (clasificación, regresión, clustering, etc.). En el tuning job, esa es la **métrica objetivo**, que el proceso de optimización busca **maximizar o minimizar** (accuracy, pérdida u otra métrica de desempeño).

### 4.3 Búsqueda bayesiana con AMT

La búsqueda bayesiana es una técnica de optimización de hiperparámetros (HPO) que usa inferencia bayesiana para modelar la relación entre los hiperparámetros y la función objetivo, y guía la búsqueda de los hiperparámetros óptimos con base en conocimiento previo y evaluaciones anteriores. A diferencia de grid search, que recorre exhaustivamente un espacio predefinido, o random search, que muestrea hiperparámetros al azar, la búsqueda bayesiana elige de forma inteligente el siguiente conjunto de hiperparámetros a evaluar **prediciendo su desempeño**, lo que mejora la eficiencia y suele requerir **menos evaluaciones** para encontrar la mejor configuración. Esto la hace especialmente eficaz para modelos complejos con espacios de hiperparámetros grandes. Es la estrategia que se usa con SageMaker AI AMT en el ejemplo detallado más abajo.

---

## 5. Evaluación de modelos fundacionales (Amazon Bedrock)

Amazon Bedrock ofrece varias herramientas de evaluación para medir el desempeño y la efectividad de **modelos fundacionales (FMs)** y **knowledge bases**.

### Evaluaciones automáticas
Usan **métricas predefinidas** como **exactitud (accuracy), robustez y toxicidad**. Puedes usar tus propios conjuntos de datos o **conjuntos integrados y curados**. Son rápidas y eficientes, y entregan puntajes y métricas calculados.

### Evaluaciones humanas
Para **métricas más subjetivas o personalizadas**, como **amabilidad, estilo y alineación con la voz de la marca**, Bedrock permite configurar flujos de evaluación humana. Puedes usar **tu propio equipo de evaluadores** o dejar que **AWS administre** las evaluaciones, que aportan calificaciones y preferencias sobre métricas específicas.

### LLM-as-a-Judge
Usa un **modelo de lenguaje grande (LLM) como juez** para evaluar las salidas del modelo. Proporcionas conjuntos de prompts personalizados y eliges métricas como **corrección, completitud y nocividad** (*correctness, completeness, harmfulness*).

### Evaluaciones programáticas
Usan algoritmos y métricas tradicionales de lenguaje natural como **BERT score, F1 y coincidencia exacta (exact matching)**. Puedes usar conjuntos de prompts integrados o traer los tuyos.

### Evaluaciones de knowledge bases
Para evaluar la **calidad de la recuperación** y flujos completos de **generación aumentada por recuperación (RAG)**, Bedrock mide las capacidades de recuperación y de generación de tus knowledge bases. Métricas: **relevancia del contexto, cobertura del contexto, fidelidad (faithfulness), corrección y completitud**.

---

## 6. Ejemplo detallado de ajuste de modelo

**Objetivo:** entrenar y evaluar el algoritmo **XGBoost** sobre el conjunto de datos **Digits** usando **Amazon SageMaker AI AMT**. El resultado esperado es la combinación óptima de hiperparámetros, obtenida al evaluar **20 modelos**, cada uno con una combinación distinta. La métrica de desempeño es la **accuracy del modelo**, y los resultados se grafican.

El programa se ejecutó con la función **Code Editor** de **Amazon SageMaker Studio**:

```python
import boto3
import sagemaker
from sagemaker import get_execution_role
from sagemaker.tuner import HyperparameterTuner, IntegerParameter, ContinuousParameter
from sagemaker.inputs import TrainingInput
from sagemaker.analytics import HyperparameterTuningJobAnalytics
from sklearn.datasets import load_digits
from sklearn.model_selection import train_test_split
import pandas as pd
import os
import matplotlib.pyplot as plt

# Load the Digits dataset
digits = load_digits()
X = digits.data
y = digits.target

# Split the dataset
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Convert to DataFrame and ensure the target is the first column
train_data = pd.DataFrame(X_train)
train_data['target'] = y_train
train_data.insert(0, 'target', train_data.pop('target'))

val_data = pd.DataFrame(X_test)
val_data['target'] = y_test
val_data.insert(0, 'target', val_data.pop('target'))

# Create directories if they don't exist
data_dir = './ch05/data'
os.makedirs(data_dir, exist_ok=True)

# Save locally
train_data.to_csv(os.path.join(data_dir, 'train.csv'), index=False)
val_data.to_csv(os.path.join(data_dir, 'validation.csv'), index=False)

# Upload to S3
s3 = boto3.client('s3')
bucket_name = 'ch05-ml-hpo'
prefix = 'sagemaker/xgboost-digits'
s3.upload_file(os.path.join(data_dir, 'train.csv'), bucket_name, f'{prefix}/train/train.csv')
s3.upload_file(os.path.join(data_dir, 'validation.csv'), bucket_name, f'{prefix}/validation/validation.csv')

# Define the role and session
role = get_execution_role()
session = sagemaker.Session()

# Define the XGBoost image
region = boto3.Session().region_name
xgb_image = sagemaker.image_uris.retrieve("xgboost", region, "1.2-1")

# Define the S3 paths for the input data
s3_train_data = f's3://{bucket_name}/{prefix}/train/train.csv'
s3_val_data = f's3://{bucket_name}/{prefix}/validation/validation.csv'

# Define the data channels
train_data = TrainingInput(s3_data=s3_train_data, content_type='csv')
val_data = TrainingInput(s3_data=s3_val_data, content_type='csv')

# Define the XGBoost estimator with required hyperparameters
xgb = sagemaker.estimator.Estimator(
    image_uri=xgb_image,
    role=role,
    instance_count=1,
    instance_type='ml.m5.large',
    output_path=f's3://{bucket_name}/{prefix}/output',
    sagemaker_session=session,
    hyperparameters={
        'objective': 'multi:softmax',  # Example for multi-class classification
        'num_class': 10,               # Number of classes in the Digits dataset
        'num_round': 100               # Default value; will be tuned
    }
)

# Define the hyperparameter ranges
hpt_ranges = {
    'alpha': ContinuousParameter(0.01, 0.5),
    'eta': ContinuousParameter(0.1, 0.5),
    'min_child_weight': ContinuousParameter(0.0, 2.0),
    'max_depth': IntegerParameter(1, 10)
}

# Define the hyperparameter tuner
tuner = HyperparameterTuner(
    estimator=xgb,
    base_tuning_job_name='bayesian',
    objective_metric_name='validation:accuracy',
    hyperparameter_ranges=hpt_ranges,
    strategy='Bayesian',
    max_jobs=20,          # Increase to 20 max jobs
    max_parallel_jobs=3,  # Number of parallel jobs
    objective_type='Maximize'
)

# Launch the hyperparameter tuning job
tuner.fit({'train': train_data, 'validation': val_data})
tuner.wait()

# Get the best training job name
best_training_job_name = tuner.best_training_job()

# Retrieve the tuning results
tuning_job_name = tuner.latest_tuning_job.name
tuner_analytics = HyperparameterTuningJobAnalytics(tuning_job_name)
tuner_df = tuner_analytics.dataframe()

# Extract the best hyperparameters
if 'FinalHyperParameters' in tuner_df.columns:
    best_hyperparameters = tuner_df.loc[
        tuner_df['TrainingJobName'] == best_training_job_name, 'FinalHyperParameters'
    ].values[0]
else:
    best_hyperparameters = tuner_df.loc[
        tuner_df['TrainingJobName'] == best_training_job_name
    ].drop(['TrainingJobName', 'FinalObjectiveValue'], axis=1).to_dict('records')[0]

# Print the best training job and hyperparameters
print(f"Best Training Job: {best_training_job_name}")
print("Best Hyperparameters:")
for key, value in best_hyperparameters.items():
    print(f"  {key}: {value}")

# Plot the tuning job results
plt.figure(figsize=(12, 8))
plt.scatter(tuner_df.index, tuner_df['FinalObjectiveValue'], label='Tuning Jobs')
plt.scatter(tuner_df.loc[tuner_df['TrainingJobName'] == best_training_job_name].index,
            tuner_df.loc[tuner_df['TrainingJobName'] == best_training_job_name]['FinalObjectiveValue'],
            color='red', marker='*', s=200, label='Best Job')

plt.xlabel('Iteration')
plt.ylabel('Cross-Validation Accuracy')
plt.title('Hyperparameter Tuning Results')

# Display the best hyperparameters on the plot
best_hyperparameters_str = '\n'.join([f'{key}: {value}' for key, value in best_hyperparameters.items()])
plt.figtext(0.5, -0.15, f'Best Hyperparameters:\n{best_hyperparameters_str}',
            ha='center', va='top', fontsize=10,
            bbox=dict(facecolor='lightgrey', alpha=0.5))

# Update legend position below the x-axis
plt.legend(loc='upper center', bbox_to_anchor=(0.5, -0.2), ncol=3)

# Save the plot as an image file
output_dir = './ch05/images'
os.makedirs(output_dir, exist_ok=True)
plt.savefig(os.path.join(output_dir, 'tuning_results.png'), bbox_inches='tight')
```

### 6.1 Aspectos clave del entrenamiento y la evaluación

**Imagen del algoritmo integrado.** Como usamos el **algoritmo integrado XGBoost de Amazon SageMaker**, necesitamos obtener la **URI de la imagen** de XGBoost para nuestra región:

```python
xgb_image = sagemaker.image_uris.retrieve("xgboost", region, "1.2-1")
```

**Canales de datos de entrada.** Son las fuentes donde el algoritmo espera encontrar los **datos de entrenamiento** y los **datos de evaluación**. Los primeros se usan para entrenar el modelo y los segundos para ajustar sus hiperparámetros (evaluación del modelo):

```python
train_data = TrainingInput(s3_data=s3_train_data, content_type='csv')
val_data = TrainingInput(s3_data=s3_val_data, content_type='csv')
```

**Estimator.** Al instanciar el estimator `xgb` le pasamos:

- la URI de la imagen de XGBoost `xgb_image`,
- el **rol de IAM** que se usa en esta sesión,
- el **número de instancias EC2** `1`,
- el **tipo de instancia** `ml.m5.large`,
- la **ruta de S3** donde se guardarán los artefactos generados tras el entrenamiento (`output_path`),
- la **sesión** iniciada por el cliente de **boto3** (el SDK de AWS para Python),
- el diccionario de **hiperparámetros**.

Firma del Estimator: https://sagemaker.readthedocs.io/en/stable/api/training/estimators.html

El diccionario `hyperparameters` es **específico del algoritmo XGBoost**:

- `objective = multi:softmax` → clasificación multiclase usando el objetivo softmax,
- `num_class` → número de clases (10 en Digits),
- `num_round` → número de rondas de boosting (iteraciones).

Hiperparámetros de XGBoost: https://docs.aws.amazon.com/sagemaker/latest/dg/xgboost-tuning.html

**Rangos de hiperparámetros y tuner.** Las llaves del diccionario `hpt_ranges` también son específicas de XGBoost. Una vez definidas, optimizamos con optimización bayesiana creando un objeto `tuner`:

```python
tuner = HyperparameterTuner(
    estimator=xgb,
    base_tuning_job_name='bayesian',
    objective_metric_name='validation:accuracy',
    hyperparameter_ranges=hpt_ranges,
    strategy='Bayesian',
    max_jobs=20,
    max_parallel_jobs=3,
    objective_type='Maximize'
)
```

y ajustándolo con los conjuntos de entrenamiento y validación:

```python
tuner.fit({'train': train_data, 'validation': val_data})
```

El tuner se construye con:

- `estimator` → el objeto `xgb` creado antes,
- `validation:accuracy` como **métrica objetivo** a **maximizar**,
- `hpt_ranges` como rangos de hiperparámetros,
- una estrategia de ajuste **bayesiana**,
- y, lo más importante, el **número máximo de tuning jobs (20)** y de **jobs en paralelo (3)** que usará el tuner durante la optimización.

Clase HyperparameterTuner: https://sagemaker.readthedocs.io/en/stable/api/training/tuner.html

**Resultados.** Cuando termina el tuning job, se identifica el mejor training job:

```python
best_training_job_name = tuner.best_training_job()
```

y se analizan los resultados del ajuste (con `HyperparameterTuningJobAnalytics`) para extraer la mejor configuración de hiperparámetros, que luego se visualiza y se grafica.

### 6.2 El programa en acción

- Al ejecutar el programa aparecen **tres training jobs nuevos** en la lista de training jobs. ¿Por qué tres? Porque en el constructor del tuner se fijó en tres el **número máximo de jobs en paralelo**.
- El programa tarda unos **15 minutos** en terminar (**45 segundos por job**). Conviene **monitorear de forma proactiva el estado de tus training jobs en SageMaker Studio**. Como se especificó un máximo de 20 training jobs, **se crean 20 jobs en total**.
- Los training jobs terminan solos al completarse: cuando un training job acaba, **SageMaker apaga automáticamente las instancias de entrenamiento**, así que no generas cargos adicionales.

> Para limitar la duración de un training job a un tiempo determinado, fija el argumento **`max_run`** del Estimator con un timeout en segundos. Pasado ese tiempo, SageMaker **termina el job sin importar su estado actual**. Su valor por defecto es **86,400 segundos**.

- Al completarse con éxito, todos los jobs quedan con estado **Completed**.
- La gráfica resultante se guarda en la carpeta `images` del **sistema de archivos EFS** conectado a la instancia de SageMaker Studio. En ella, el **cuarto job de izquierda a derecha, marcado con una estrella, es el mejor training job** (máxima accuracy), y su combinación de hiperparámetros aparece en la parte inferior de la gráfica.
- Cuando termines de usar SageMaker Studio, **detén tu instancia** (botón Stop) para evitar cargos no deseados.

### 6.3 Resumen del ejercicio

Se usó SageMaker AI AMT para optimizar un modelo XGBoost con el conjunto de datos Digits. Pasos: cargar y dividir el dataset, guardar los datos localmente y **subirlos a un bucket de S3**, definir el **estimator de XGBoost** con los hiperparámetros requeridos y configurar un **tuner de hiperparámetros** con los rangos especificados. El tuner hizo **optimización bayesiana** para encontrar la mejor combinación mientras el programa esperaba a que terminaran los tuning jobs. Después se identificaron el mejor training job y la mejor combinación de hiperparámetros, y los resultados se visualizaron y guardaron como imagen.

---

## 7. Resumen (partes de AWS)

- El entrenamiento puede ser **local, remoto o distribuido**; las técnicas de monitoreo y depuración (CloudWatch, EventBridge, SageMaker Debugger) aseguran que el entrenamiento corra sin problemas, ya sea en máquinas locales o distribuido en recursos de la nube.
- Las estrategias para explorar el espacio de hiperparámetros usan **SageMaker AI AMT**.
- **Amazon Bedrock** ofrece herramientas para evaluar modelos fundacionales: evaluaciones automáticas, evaluaciones humanas y evaluaciones de knowledge bases; un marco de evaluación completo para construir y desplegar aplicaciones de IA generativa con confianza y responsabilidad.
- El ejemplo detallado aplicó **SageMaker AI AMT con optimización bayesiana** para ajustar hiperparámetros y evaluar el modelo sobre el conjunto de datos Digits.

---

## 8. Puntos esenciales para el examen (partes de AWS)

**Conoce la diferencia entre entrenamiento local, remoto y distribuido.** El entrenamiento local entrena el modelo en una sola máquina con sus recursos disponibles. El entrenamiento remoto ejecuta el proceso en un servicio en la nube (p. ej., Amazon SageMaker) o en un servidor remoto, aprovechando cómputo externo sin sobrecargar el sistema local. El entrenamiento distribuido reparte la carga entre varias máquinas o nodos, local o remotamente, para manejar conjuntos de datos más grandes y reducir el tiempo de entrenamiento paralelizando los cálculos.

**Conoce la diferencia entre la búsqueda bayesiana y otras técnicas de ajuste de hiperparámetros.** La búsqueda bayesiana usa inferencia bayesiana para modelar la relación entre los hiperparámetros y la función objetivo (la métrica que la optimización busca maximizar o minimizar, como accuracy o pérdida), prediciendo el desempeño de los hiperparámetros candidatos y requiriendo a menudo menos evaluaciones que grid search (exhaustiva) o random search (muestreo aleatorio). Se usó en el ejemplo detallado para optimizar el modelo XGBoost con **Amazon SageMaker AI Automatic Model Tuning**.

---

## 9. Pregunta de repaso (AWS)

**9. ¿Cuál de las siguientes es una forma común de monitorear y depurar training jobs distribuidos en Amazon SageMaker?**

- A. Usar Jupyter Notebooks
- B. Habilitar Spot Instances
- C. Implementar Amazon SageMaker Debugger
- D. Deshabilitar Auto Scaling
