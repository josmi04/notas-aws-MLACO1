# Capítulo 6 · Despliegue y orquestación de modelos

**Objetivos del examen MLA-C01 cubiertos** (Dominio 3: despliegue y orquestación de flujos de trabajo de ML):

- **3.1** Seleccionar la infraestructura de despliegue según la arquitectura y los requisitos existentes.
- **3.2** Crear y programar (*scripting*) la infraestructura según la arquitectura y los requisitos existentes.
- **3.3** Usar herramientas de orquestación automatizada para montar pipelines de integración y entrega continuas (CI/CD).

---

## Servicios de AWS para desplegar modelos

En el capítulo 5 se vio cómo entrenar un modelo y evaluar su rendimiento. Este capítulo cubre las dos fases siguientes del ciclo de vida de ML: **desplegar el modelo** y **obtener inferencias**. Desplegar un modelo consiste en integrarlo, junto con todos sus recursos, en un entorno de producción para que genere predicciones (inferencias).

En todo el capítulo, **aplicación de ML** (*ML app*) designa la aplicación que recibe las peticiones de inferencia, pasa los datos al modelo y devuelve la respuesta al solicitante. Esa respuesta puede llegar:

- **en tiempo real**, de inmediato;
- **casi en tiempo real**, con unos segundos de retraso;
- **al cabo de un tiempo indeterminado**, normalmente porque las peticiones se procesan por lotes y el solicitante no espera una respuesta inmediata. En ese caso, el tiempo depende, entre otros factores, del tamaño del lote y de la cantidad de datos de cada petición (la **carga útil** o *payload*).

Según el caso de uso, puedes consumir **servicios de IA de AWS** llamando directamente a sus API, o exponer **tu propio modelo** mediante un endpoint de **Amazon SageMaker AI** (nuevo nombre de Amazon SageMaker). En ambos casos, el despliegue debe seguir los seis pilares del AWS Well-Architected Framework: excelencia operativa, seguridad, fiabilidad, eficiencia del rendimiento, optimización de costes y sostenibilidad.

---

## Despliegue de servicios de IA de AWS

Además de servir para construir, entrenar y desplegar modelos propios, SageMaker AI se integra con los servicios de IA totalmente gestionados de AWS (vistos en el capítulo 4). Estos servicios se incorporan a las aplicaciones mediante llamadas a API, y en su mayoría ofrecen modelos preentrenados.

> **Para el examen:** los modelos de estos servicios se alojan en cuentas de AWS gestionadas por el propio servicio, así que tu acceso a ellos es limitado: solo los consumes mediante llamadas a API.

Las llamadas a estos servicios pueden integrarse en tus flujos de trabajo de ML y orquestarse con ellos. La orquestación, un requisito clave del examen, se trata al final del capítulo. Así puedes, por ejemplo, procesar imágenes y vídeo a escala sin acceder a los modelos subyacentes.

| Servicio | Qué hace | Cliente boto3 | Métodos clave |
|---|---|---|---|
| [Rekognition](https://docs.aws.amazon.com/rekognition/latest/APIReference/Welcome.html) | Análisis de imágenes y vídeo: objetos, caras, texto y escenas | `rekognition` | `detect_labels` |
| [Textract](https://docs.aws.amazon.com/textract/latest/dg/API_Reference.html) | Extracción de texto y datos (tablas, formularios) de documentos escaneados | `textract` | `analyze_document`, `detect_document_text` (síncronos); `get_document_analysis` (recupera el resultado de un análisis asíncrono iniciado con `start_document_analysis`) |
| [Polly](https://docs.aws.amazon.com/polly/latest/dg/API_Reference.html) | Texto a voz | `polly` | `synthesize_speech`, `describe_voices`, `list_lexicons` |
| [Transcribe](https://docs.aws.amazon.com/transcribe/latest/APIReference/Welcome.html) | Voz a texto | `transcribe` | `start_transcription_job`, `get_transcription_job`, `list_transcription_jobs` |
| [Comprehend](https://docs.aws.amazon.com/comprehend/latest/APIReference/welcome.html) | Procesamiento del lenguaje natural: sentimiento, entidades, frases clave | `comprehend` | `detect_sentiment`, `detect_entities`, `detect_key_phrases` |
| [Lex](https://docs.aws.amazon.com/lexv2/latest/APIReference/welcome.html) | Interfaces conversacionales (chatbots) de voz y texto | `lexv2-runtime` | `recognize_text` |
| [Personalize](https://docs.aws.amazon.com/personalize/latest/dg/API_Reference.html) | Recomendaciones personalizadas | `personalize` (gestión) y `personalize-runtime` (inferencia) | `create_campaign`; `get_recommendations`, `get_personalized_ranking` |
| [Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/getting-started-api.html) | Inferencia con modelos fundacionales (FM) | `bedrock-runtime`; los agentes se invocan con `bedrock-agent-runtime` | `converse` (también `invoke_model`) |

### Amazon Rekognition

Para usarlo desde Python necesitas el AWS SDK for Python (**boto3**), que ya viene preinstalado en el Code Editor de SageMaker Studio. Primero subes las imágenes o vídeos a un bucket de S3 y después llamas a `detect_labels`. El parámetro `Image` indica el bucket y el archivo; `MaxLabels` fija el número máximo de etiquetas devueltas y `MinConfidence`, la confianza mínima exigida. La respuesta (`response['Labels']`) contiene las etiquetas detectadas con su puntuación de confianza:

```python
import boto3

client = boto3.client('rekognition')
response = client.detect_labels(
    Image={'S3Object': {'Bucket': 'your-bucket-name', 'Name': 'your-image.jpg'}},
    MaxLabels=10,
    MinConfidence=75
)
for label in response['Labels']:
    print(f"Label: {label['Name']}, Confidence: {label['Confidence']}")
```

Los ejemplos siguientes asumen `import boto3`.

### Amazon Textract

Extrae texto y datos de documentos escaneados. Las operaciones síncronas como `analyze_document` admiten imágenes y PDF de una sola página; los documentos multipágina se procesan de forma asíncrona (`start_document_analysis` + `get_document_analysis`).

```python
client = boto3.client('textract')
response = client.analyze_document(
    Document={'S3Object': {'Bucket': 'your-bucket-name', 'Name': 'your-document.pdf'}},
    FeatureTypes=['TABLES', 'FORMS']
)
for block in response['Blocks']:
    if block['BlockType'] == 'LINE':
        print(block['Text'])
```

### Amazon Polly

Convierte texto en voz realista:

```python
client = boto3.client('polly')
response = client.synthesize_speech(
    Text='Hello, world!',
    OutputFormat='mp3',
    VoiceId='Joanna'
)
with open('speech.mp3', 'wb') as file:
    file.write(response['AudioStream'].read())
```

### Amazon Transcribe

Convierte voz en texto. La transcripción es un trabajo asíncrono: se inicia y luego se consulta su estado hasta que termina.

```python
import time

client = boto3.client('transcribe')
client.start_transcription_job(
    TranscriptionJobName='TranscriptionJob',
    Media={'MediaFileUri': 's3://your-bucket-name/your-audio-file.mp3'},
    MediaFormat='mp3',
    LanguageCode='en-US'
)

# Esperar a que el trabajo termine y recuperar la transcripción
while True:
    status = client.get_transcription_job(TranscriptionJobName='TranscriptionJob')
    if status['TranscriptionJob']['TranscriptionJobStatus'] in ['COMPLETED', 'FAILED']:
        break
    time.sleep(5)  # pausa entre consultas para no saturar la API

print(status['TranscriptionJob']['Transcript']['TranscriptFileUri'])
```

### Amazon Comprehend

Análisis de sentimiento, reconocimiento de entidades y extracción de frases clave:

```python
client = boto3.client('comprehend')
response = client.detect_sentiment(
    Text='I love using Amazon Comprehend!',
    LanguageCode='en'
)
print(f"Sentiment: {response['Sentiment']}, Score: {response['SentimentScore']}")
```

### Amazon Lex

Sirve para crear interfaces conversacionales de voz y texto. `recognize_text` gestiona la interacción con tu bot:

```python
client = boto3.client('lexv2-runtime')
response = client.recognize_text(
    botId='YourBotId',
    botAliasId='YourBotAliasId',
    localeId='en_US',
    sessionId='User123',
    text='I would like to book a flight'
)
print(f"Bot Response: {response['messages'][0]['content']}")
```

### Amazon Personalize

A diferencia de los servicios anteriores, **Personalize no ofrece modelos preentrenados**: entrena un modelo personalizado con tus propios datos (usuarios, ítems e interacciones) y lo expone mediante una **campaña**. La gestión (p. ej., `create_campaign`) usa el cliente `personalize`, mientras que las recomendaciones se piden con el cliente **`personalize-runtime`**:

```python
client = boto3.client('personalize-runtime')
response = client.get_recommendations(
    campaignArn='your-campaign-arn',
    userId='User123'
)
for item in response['itemList']:
    print(f"Recommended Item: {item['itemId']}, Score: {item['score']}")
```

### Amazon Bedrock

Amazon Bedrock ofrece acceso seguro y escalable a modelos fundacionales (FM) mediante API, sin que tengas que gestionar la infraestructura que los aloja. Para la inferencia directa, el cliente `bedrock-runtime` ofrece la API `InvokeModel` y la **Converse API**. Además, puedes usar **agentes**.

#### Agentes

Un agente de Bedrock actúa como intermediario entre tu aplicación y el FM. Defines qué tareas puede realizar y cómo debe comportarse; el agente usa el FM para interpretar la petición, dividirla en pasos y ejecutar las acciones configuradas (por ejemplo, llamar a API de tu organización). El proceso consiste en crear el agente, configurar sus tareas y crear un **alias** que tu aplicación invoca (con el cliente `bedrock-agent-runtime`). Los agentes son la opción adecuada para flujos complejos de varios pasos que requieren orquestación.

#### Converse API

Está pensada para interacciones conversacionales: envías mensajes, cada uno con su rol, y recibes la respuesta del modelo, con el mismo formato sea cual sea el modelo usado. Es adecuada para chatbots, asistentes virtuales y atención al cliente, y es una alternativa sencilla cuando un agente resultaría demasiado complejo.

```python
client = boto3.client('bedrock-runtime')
response = client.converse(
    modelId='your-model-id',
    messages=[{
        'role': 'user',
        'content': [{'text': 'Create a list of ten bestselling books about Calculus'}]
    }]
)
print(f"Model Response: {response['output']['message']['content'][0]['text']}")
```

---

## Despliegue de tu propio modelo

Cuando has construido y ajustado tu propio modelo, el despliegue lo hace accesible para generar predicciones sobre datos nuevos, en tiempo real o por lotes. Hay dos enfoques:

| | Despliegue gestionado (SageMaker AI) | Despliegue no gestionado (EC2, ECS, EKS, Lambda) |
|---|---|---|
| Quién gestiona la infraestructura | SageMaker AI: aprovisionamiento, escalado, balanceo de carga, alta disponibilidad y tolerancia a fallos | Tú |
| Ventaja principal | Poca carga operativa; te centras en la lógica y el rendimiento de la aplicación | Control y personalización: hardware, dependencias, red e integración con sistemas existentes; ajuste fino de recursos y costes |
| Cuándo elegirlo | Producción con requisitos de alta disponibilidad y escalado sin esfuerzo | Configuraciones complejas, redes personalizadas o integración con sistemas existentes |

La elección del tipo y tamaño de infraestructura depende de cinco criterios: **escalabilidad, facilidad de gestión, necesidad de personalización, seguridad y coste**.

### Consideraciones para elegir la infraestructura

Como el objetivo del despliegue es obtener inferencias, conviene entender en qué se diferencian del entrenamiento a nivel de cómputo. La inferencia aplica el modelo entrenado a datos nuevos mediante un *forward pass*, igual que en el entrenamiento, pero **sin *backward pass***: no calcula errores ni actualiza los pesos (u otros parámetros internos) para minimizar la función de pérdida. De ahí salen las diferencias de la Tabla 6.1, que debes conocer para el examen.

**Tabla 6.1 · Cómputo de inferencia frente a cómputo de entrenamiento**

| Inferencia | Entrenamiento |
|---|---|
| Se ejecuta en endpoints de tiempo real individuales (salvo la inferencia por lotes) | Requiere alto paralelismo y procesamiento en lotes grandes para lograr más *throughput* |
| Menos cómputo | Más cómputo |
| Menos memoria | Más memoria |
| Se ejecuta en cualquier lugar (dispositivos *edge* y nube) | Se ejecuta en la nube |
| Se ejecuta todo el tiempo | Se ejecuta cuando hace falta (se entrena una vez y se reentrena con poca frecuencia) |
| Integrada en la pila de la aplicación de ML | Independiente |

En resumen, la infraestructura de entrenamiento busca gran potencia de cómputo para procesar muchos datos y optimizar el modelo, mientras que la de inferencia prioriza **baja latencia y eficiencia**.

---

## Despliegues gestionados en SageMaker AI

Alojar el modelo en SageMaker AI (**despliegue gestionado**) delega en AWS el aprovisionamiento, el escalado, el balanceo de carga y la monitorización del estado de la infraestructura, lo que reduce mucho la carga operativa. Además, se integra con **IAM** para el control de acceso y con **Amazon CloudWatch** para métricas y logs, en línea con los pilares del Well-Architected Framework.

Todas las opciones de despliegue de SageMaker AI se basan en **contenedores**, que encapsulan el modelo y sus dependencias. Así se garantiza un entorno coherente en todas las fases del ciclo de vida y se facilita pasar de desarrollo a producción. El otro elemento común es el **artefacto del modelo**, normalmente `model.tar.gz`, que contiene los parámetros aprendidos y los archivos necesarios para la inferencia. Al desplegar, SageMaker AI lo descarga de S3 y lo descomprime en las instancias que atienden las peticiones.

SageMaker AI ofrece cuatro opciones de despliegue gestionado:

| Opción | Latencia | Carga útil / duración | Patrón de tráfico | Infraestructura | Casos de uso |
|---|---|---|---|---|---|
| **Tiempo real** | Baja, inmediata | Pequeña | Continuo, interactivo | Endpoint con instancias siempre activas, autoescalable | Detección de fraude, recomendaciones, apps interactivas |
| **Serverless** | Baja, con posibles arranques en frío | Pequeña | Intermitente o impredecible | Endpoint sin instancias que elegir; escala según demanda (incluso a cero) | Tráfico variable con periodos de inactividad |
| **Asíncrona** | Diferida (casi tiempo real) | Grande, procesamiento largo | Peticiones pesadas que toleran retraso | Endpoint con cola interna; entrada y salida en S3; notificación opcional por SNS | Imágenes o documentos grandes, modelos complejos |
| **Por lotes** (*batch transform*) | No importa | Datasets completos | Offline, periódico | **Sin endpoint**: trabajo que procesa en paralelo y termina | *Scoring* de clientes, análisis offline, mantenimiento predictivo |

### Inferencia en tiempo real

Da predicciones inmediatas a cada petición entrante. Es la opción adecuada para aplicaciones que necesitan respuestas de baja latencia con cargas útiles pequeñas. Usa un artefacto del modelo en Amazon S3 y un contenedor de inferencia de Amazon Elastic Container Registry (ECR), servidos por una o varias instancias autoescalables.

Funcionamiento a alto nivel:

1. **Modelo.** Se registra en SageMaker AI el modelo, que combina el artefacto `model.tar.gz` de S3 (modelo serializado, parámetros aprendidos, archivos de configuración o scripts generados por el trabajo de entrenamiento) y la imagen del contenedor de inferencia en ECR. SageMaker AI ofrece contenedores para frameworks como TensorFlow y PyTorch y para sus algoritmos integrados (capítulo 4).
2. **Configuración del endpoint.** Define el tipo de instancia (p. ej., `ml.m5.large`), el número de instancias y el modelo que se despliega. Puedes ajustarla a tus requisitos de rendimiento y escalabilidad. El método `deploy()` del SDK crea el modelo, la configuración y el endpoint automáticamente.
3. **Endpoint HTTPS.** Al crear el endpoint a partir de la configuración, SageMaker AI aprovisiona las instancias, crea el contenedor a partir de la imagen de ECR (en un runtime Docker), descarga y descomprime el artefacto, y el contenedor carga (deserializa) el modelo. El endpoint queda expuesto como una URL única con HTTPS, que cifra la comunicación entre cliente y servidor.
4. **Autoescalado y balanceo de carga (opcional).** Con **Application Auto Scaling**, el número de instancias crece o decrece (*scale-out* / *scale-in*) según políticas o métricas. SageMaker AI reparte automáticamente las peticiones entre las instancias disponibles.
5. **Monitorización y logs.** CloudWatch muestra métricas como el número de invocaciones, la latencia y la tasa de errores; los logs sirven para depurar y analizar el rendimiento.
6. **Invocación.** Los clientes envían peticiones HTTPS POST con los datos de entrada (API `InvokeEndpoint` de SageMaker Runtime). El contenedor procesa la entrada, la pasa por el modelo y devuelve la predicción a la aplicación de ML, que la entrega al cliente.

El examen exige conocer dos variantes de la inferencia en tiempo real: los **endpoints multimodelo (MME)** y los **endpoints multicontenedor (MCE)**.

#### Endpoints multimodelo (MME)

Alojan **muchos modelos en un solo endpoint que comparten el mismo contenedor** (misma imagen, framework y entorno de ejecución), lo que optimiza el uso de recursos y evita pagar un endpoint por modelo. Son ideales cuando tienes **muchos modelos de uso poco frecuente**. Los modelos se guardan en S3 y cada petición indica cuál usar (parámetro `TargetModel`). SageMaker AI carga el modelo en memoria la primera vez que se invoca y, cuando falta memoria, descarga los menos usados.

Limitaciones:

- Todos los modelos deben usar **la misma imagen de contenedor**, lo que no sirve si necesitan frameworks o dependencias distintos.
- Los modelos comparten el contenedor y los recursos de la instancia, así que puede haber **contención de recursos** con mucha demanda.
- Invocar un modelo que no está en memoria añade la **latencia de su carga** (arranque en frío).

Aun así, los MME son una solución flexible y rentable para desplegar muchos modelos.

#### Endpoints multicontenedor (MCE)

Superan la limitación de los MME: permiten desplegar **contenedores con frameworks distintos** en un solo endpoint (**hasta 15 contenedores**). Todos los contenedores se ejecutan en las mismas instancias, lo que aprovecha mejor los recursos y simplifica pipelines de inferencia complejos.

Se crean con la API **`CreateModel`** (una API REST a la que pasas el nombre del modelo, los contenedores y el rol de ejecución; ver la [referencia](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateModel.html)), definiendo un modelo "polimórfico" con varias imágenes de contenedor (p. ej., Scikit-Learn, TensorFlow y PyTorch). Los contenedores pueden invocarse de dos formas:

- **Invocación en serie:** los contenedores se ejecutan en secuencia, como un pipeline.
- **Invocación directa:** cada contenedor se consume de forma independiente, lo que mejora la utilización del endpoint y optimiza costes. El cliente elige el contenedor indicando su *hostname* de destino (`TargetContainerHostname`) en la llamada `invoke_endpoint` de SageMaker Runtime.

Como en los MME, los contenedores comparten instancia, así que puede haber contención de recursos si requieren mucho cómputo.

> **Para el examen**, casos de uso típicos de los MCE:
>
> - Alojar modelos de **frameworks distintos**.
> - Alojar modelos del **mismo framework con algoritmos distintos**.
> - **Pruebas A/B** (p. ej., comparar versiones distintas de un framework).
>
> Si el flujo incluye varios modelos o pasos de preprocesamiento con frameworks y dependencias diferentes, un MCE es una opción válida.

### Inferencia serverless

Con la inferencia serverless ya no tienes que decidir cuántas instancias necesitas ni de qué tipo. SageMaker AI aprovisiona y escala automáticamente la capacidad de cómputo según la carga de trabajo: la amplía ante ráfagas de tráfico (*scale-out*) y la libera cuando baja la demanda (*scale-in*), incluso hasta cero cuando el endpoint está inactivo. Solo **pagas por el cómputo usado para procesar peticiones** (en función de la memoria configurada) **y por los datos procesados**, no por instancias.

Configuras dos parámetros: el **tamaño de memoria** y la **concurrencia máxima** del endpoint.

> - **Concurrencia máxima** (*max concurrency*): número máximo de peticiones simultáneas que el endpoint puede procesar. Limita el paralelismo y ayuda a gestionar los picos de tráfico.
> - **Concurrencia aprovisionada** (*provisioned concurrency*): mantiene siempre listo un número de entornos de ejecución. Evita los **arranques en frío** (*cold starts*) que sufre la primera petición tras un periodo de inactividad, con lo que el tiempo de respuesta es más bajo y predecible. Estos entornos se pagan aunque no reciban tráfico.

El despliegue es igual que en tiempo real: indicas el artefacto (`model.tar.gz`) y la URI de la imagen del contenedor, y SageMaker AI descarga y descomprime el modelo, lo inicializa y expone el endpoint.

La inferencia serverless es especialmente adecuada para **tráfico variable o impredecible**. En el ejemplo práctico del final del capítulo se usa la clase [`ServerlessInferenceConfig`](https://sagemaker.readthedocs.io/en/stable/api/inference/serverless.html#sagemaker.serverless.serverless_inference_config.ServerlessInferenceConfig) del SageMaker Python SDK para exponer con un endpoint serverless el mejor modelo XGBoost ajustado en el capítulo 5.

### Inferencia asíncrona

Está diseñada para **peticiones pesadas y de larga duración que no necesitan respuesta inmediata**. En lugar de devolver el resultado en la misma llamada, las peticiones se **encolan** y se procesan a medida que hay capacidad, lo que equilibra la carga sin saturar la infraestructura.

Flujo:

1. Creas un endpoint asíncrono, de forma similar a uno de tiempo real.
2. Dejas la carga útil de cada petición en S3 y envías la petición al endpoint (`InvokeEndpointAsync`) indicando su ubicación.
3. SageMaker AI encola la petición, la procesa y escribe el resultado en S3, desde donde lo recuperas cuando quieras.
4. Opcionalmente, recibes notificaciones de éxito o error con **Amazon SNS**.

Es la opción adecuada cuando la inferencia es costosa en recursos y tiempo y tolera respuestas diferidas: imágenes o documentos grandes, modelos complejos con tiempos de inferencia largos o transformaciones de datos extensas.

### Inferencia por lotes (batch transform)

Cuando crecen el volumen y el tamaño de las peticiones, la inferencia por lotes permite **procesar grandes datasets en bloque** sin requisitos de baja latencia.

> **Importante:** los trabajos de *batch transform* **no usan endpoints**. Procesan los datos en bloque, guardan los resultados en la ubicación indicada y los recursos solo existen mientras dura el trabajo.

Al crear un trabajo de *batch transform* indicas la ubicación de los datos de entrada, el modelo y la ubicación de salida. SageMaker AI procesa los datos en paralelo y deja los resultados en la salida indicada. Como en la inferencia asíncrona, esta arquitectura desacoplada apenas requiere supervisión.

Es adecuada cuando hay que procesar grandes volúmenes de datos y la inmediatez no es crítica: analítica offline, preprocesamiento de datos, predicciones periódicas sobre datasets completos, *scoring* de clientes para campañas de marketing o análisis de datos de sensores para mantenimiento predictivo.

---

## Despliegues no gestionados

No es obligatorio desplegar con SageMaker AI. Aunque ofrece una solución robusta y totalmente gestionada, no siempre es la mejor opción. En un **despliegue no gestionado**, el modelo no se aloja en SageMaker AI (aunque se haya entrenado allí) y **tú asumes la gestión de la infraestructura de despliegue**. Ganas flexibilidad y control a cambio del coste y la complejidad de gestionarla, incluso cuando usas servicios serverless como Fargate o Lambda.

Qué impulsa esta elección:

- **Control y personalización:** configuraciones de hardware concretas, dependencias de software propias y ajustes de red especializados.
- **Coste:** puedes elegir con precisión los tipos de instancia, usar instancias de spot y gestionar tú mismo las políticas de escalado.
- **Flujos complejos y necesidades de integración** con sistemas existentes.
- **Residencia de datos o de cómputo y cumplimiento normativo** (p. ej., GDPR, HIPAA): cuando necesitas implementar controles de seguridad propios sobre la infraestructura.

Los servicios de cómputo de AWS para estos despliegues son **Amazon EC2, ECS, EKS y AWS Lambda**.

### Amazon Elastic Compute Cloud (EC2)

EC2 proporciona servidores virtuales con **control total** del sistema operativo, la red y la seguridad, y te permite elegir el tipo de instancia que mejor se ajuste a la carga en rendimiento y coste. La decisión clave es **CPU o GPU**:

- **CPU:** ML tradicional, preprocesamiento de datos y aplicaciones sin procesamiento paralelo intensivo. Encaja en las primeras fases del ciclo de vida (limpieza de datos, ingeniería de características) y en inferencias que no exigen mucho cómputo.
- **GPU:** deep learning, procesamiento de imágenes y otras operaciones intensivas que aprovechan el paralelismo. Destacan en entrenamiento y evaluación con grandes datasets y algoritmos complejos, donde **reducen el tiempo de entrenamiento** (la GPU acelera el cálculo, pero no mejora por sí misma la precisión del modelo).

> Entrenar un modelo de deep learning suele requerir una GPU potente, pero la **inferencia** necesita mucho menos cómputo y a menudo basta con una GPU menor o incluso una CPU, según la latencia deseada. Desplegarlo en una GPU grande puede suponer **infrautilización y costes innecesarios**.

La Tabla 6.2 ayuda a elegir el tipo de instancia EC2 para inferencia. Los costes son aproximados porque varían por región y otros factores. Recuerda que el entrenamiento suele exigir mucha memoria, mientras que la inferencia busca **alto *throughput* y baja latencia**.

> **Para el examen:** las instancias **AWS Inferentia (Inf1, Inf2)** ofrecen alto rendimiento al menor coste para inferencia de deep learning e IA generativa. El **AWS Neuron SDK** se integra con frameworks como PyTorch y TensorFlow, de modo que puedes seguir usando tu código y tus flujos de trabajo sobre chips Inferentia. Más información: <https://aws.amazon.com/ec2/instance-types/inf2>.

**Tabla 6.2 · Tipos de instancia EC2 para inferencia**

| Tipo (categoría) | Caso de uso | Características clave | Coste aprox. | GPU | Arquitectura de cómputo |
|---|---|---|---|---|---|
| **t2** (uso general, rendimiento ampliable) | Desarrollo, pruebas, inferencia básica | CPU de bajo coste con capacidad de ráfaga | Bajo | No | x86-64 |
| **c5** (optimizada para cómputo) | Cargas de ML versátiles, inferencia básica | Alta proporción de CPU respecto a memoria | Moderado | No | x86-64 |
| **m5** (uso general) | Cargas mixtas, inferencia básica | CPU, memoria y red equilibradas | Moderado | No | x86-64 |
| **Inf1** (computación acelerada) | Inferencia de alto rendimiento y bajo coste | Chips AWS Inferentia | Moderado | No | AWS Inferentia |
| **Inf2** (computación acelerada) | Inferencia de alto rendimiento y bajo coste | Chips AWS Inferentia de segunda generación | Moderado | No | AWS Inferentia2 |
| **g4dn** (computación acelerada) | Inferencia de deep learning | GPU NVIDIA rentables | Alto | Sí | NVIDIA T4 Tensor Core |
| **g5** (computación acelerada) | Inferencia de deep learning | GPU NVIDIA de mayor rendimiento | Alto | Sí | NVIDIA A10G Tensor Core |
| **g6** (computación acelerada) | Inferencia de deep learning | GPU NVIDIA L4 | Alto | Sí | NVIDIA L4 Tensor Core |
| **p4** (computación acelerada) | Inferencia de alto rendimiento | GPU NVIDIA de gama alta | El más alto | Sí | NVIDIA A100 Tensor Core |
| **p5** (computación acelerada) | Inferencia de alto rendimiento | GPU NVIDIA de gama alta | El más alto | Sí | NVIDIA H100 Tensor Core |
| **Trn1** (computación acelerada) | Entrenamiento e inferencia de alto rendimiento (LLM e IA generativa) | Chips AWS Trainium | El más alto | No | AWS Trainium |
| **Trn2** (computación acelerada) | Entrenamiento e inferencia de alto rendimiento (LLM e IA generativa) | Chips AWS Trainium2 | El más alto | No | AWS Trainium2 |

### Amazon Elastic Container Service (ECS)

ECS ejecuta y gestiona aplicaciones en contenedores. El despliegue del modelo se describe con **definiciones de tarea** (*task definitions*), que especifican las imágenes Docker, los recursos y la configuración de red. Admite dos tipos de lanzamiento:

- **EC2:** control granular de la infraestructura subyacente; eliges los tipos de instancia para optimizar rendimiento y coste.
- **AWS Fargate:** abstrae la infraestructura (experiencia serverless); solo te ocupas de ejecutar los contenedores.

Se integra con CloudWatch (monitorización de contenedores), IAM (seguridad y control de acceso) y Amazon EFS (almacenamiento persistente). Además, ofrece **descubrimiento de servicios** (*service discovery*) para que los contenedores se comuniquen dentro de la misma red, algo útil en arquitecturas de microservicios.

### Amazon Elastic Kubernetes Service (EKS)

EKS usa **Kubernetes**, la plataforma de código abierto líder en orquestación de contenedores, para desplegar, gestionar y escalar modelos con las herramientas y API nativas de Kubernetes. AWS gestiona el plano de control de Kubernetes (alta disponibilidad, seguridad y rendimiento), y EKS se integra con S3, IAM y CloudWatch.

Te da acceso a todas las capacidades de Kubernetes, como complementos (*add-ons*) y herramientas de terceros (service meshes, logging, monitorización), autoescalado, actualizaciones progresivas (*rolling updates*) y autorreparación (*self-healing*). Es la opción natural para **organizaciones con infraestructura o experiencia previa en Kubernetes**, porque ofrece una experiencia de despliegue coherente tanto *on-premises* como en la nube.

### AWS Lambda

Lambda es un servicio de cómputo **serverless**: empaquetas el modelo en una función que **escala automáticamente** con las peticiones y **pagas solo por el tiempo de cómputo consumido**. Las funciones pueden dispararse desde servicios como API Gateway, S3 o DynamoDB.

- Maneja con facilidad **cargas muy variables** (picos repentinos, estacionalidad) sin intervención manual.
- Se integra con CloudWatch para métricas, logs, alarmas y notificaciones.
- Admite varios lenguajes, entre ellos Python. Las **Lambda Layers** permiten gestionar dependencias y compartir código entre funciones.
- Cada invocación es independiente (*stateless*).

Combinada con S3 (almacenamiento del modelo) y API Gateway (API REST), forma una solución de despliegue escalable y rentable.

| Servicio | Nivel de control | Encaja cuando… |
|---|---|---|
| EC2 | Máximo (SO, red, hardware) | Necesitas personalizar a fondo el entorno o el hardware |
| ECS | Contenedores; con EC2 o Fargate | Quieres contenedores gestionados de forma sencilla e integrada con AWS |
| EKS | Contenedores con Kubernetes | Ya usas Kubernetes o quieres su ecosistema |
| Lambda | Solo el código de la función (serverless) | El tráfico es variable y quieres pagar por uso |

### Optimización de modelos para dispositivos edge: Amazon SageMaker Neo

**SageMaker Neo** compila y ajusta modelos para que se ejecuten eficientemente en un hardware y software concretos, lo que lo hace ideal para dispositivos *edge*:

- Admite dispositivos con procesadores ARM, Intel, NVIDIA y otros.
- Admite frameworks como TensorFlow, TensorFlow Lite, PyTorch y el formato estándar ONNX (Open Neural Network Exchange).
- **Qué hace:** compila el modelo; optimiza su grafo de cómputo y genera un binario específico para el hardware de destino. No reentrena ni poda el modelo: técnicas como la poda o la cuantización se aplican aparte, antes de compilar.

Proceso: entrenas el modelo (en SageMaker AI o en otro sitio) y Neo lo compila para el dispositivo de destino, generando un binario optimizado que **reduce la latencia y el consumo de energía**. Eso lo hace adecuado para dispositivos de bajo consumo.

> Neo **no** gestiona la distribución ni las actualizaciones *over-the-air* (OTA) de modelos en los dispositivos. Para desplegar y actualizar modelos en flotas de dispositivos se usan otros servicios, como AWS IoT Greengrass. SageMaker Edge Manager, que cubría esta función, se retiró en 2024.

Dispositivos, instancias y versiones de frameworks admitidos: <https://docs.aws.amazon.com/sagemaker/latest/dg/neo-supported-devices-edge.html>.

El ejemplo del libro exporta un modelo TensorFlow preentrenado en formato SavedModel, lo sube a S3 y crea un trabajo de compilación de Neo para iPhone (Apple Core ML). SavedModel es un **directorio**, y Neo exige que `S3Uri` apunte a **un único archivo `.tar.gz`**: antes de subirlo hay que empaquetarlo en `model.tar.gz`.

```python
import boto3

sagemaker_client = boto3.client('sagemaker')
sagemaker_client.create_compilation_job(
    CompilationJobName='MyCompilationJob',
    RoleArn='<your-sagemaker-execution-role-arn>',
    InputConfig={
        'S3Uri': 's3://your-bucket/model.tar.gz',       # SavedModel empaquetado
        'DataInputConfig': '{"input": [1, 224, 224, 3]}',
        'Framework': 'TENSORFLOW'
    },
    OutputConfig={
        'S3OutputLocation': 's3://your-bucket/compiled-model',
        'TargetDevice': 'coreml'                        # Apple Core ML (iPhone)
    },
    StoppingCondition={'MaxRuntimeInSeconds': 3600}
)

# boto3 no incluye un waiter para trabajos de compilación: se consulta el estado
status = sagemaker_client.describe_compilation_job(
    CompilationJobName='MyCompilationJob'
)['CompilationJobStatus']
print(f'Compilation job status: {status}')
```

---

## Técnicas avanzadas de despliegue

SageMaker AI ofrece técnicas para que los modelos funcionen de forma óptima y fiable en producción:

- **Autoescalado** de endpoints con Application Auto Scaling, que ajusta los recursos a un tráfico variable.
- **Estrategias de despliegue** como **Blue/Green** y de prueba como **canary**, **shadow** y **A/B**, para pasar a nuevas versiones del modelo con la mínima interrupción.

### Autoescalado de endpoints

El autoescalado de un endpoint de SageMaker AI **escala horizontalmente** (*scale-in* / *scale-out*) las instancias que atienden las peticiones de inferencia. **Application Auto Scaling** ajusta el número de instancias según métricas como el uso de CPU o el número de peticiones, manteniendo el rendimiento con el coste bajo control. Se configura con el cliente `application-autoscaling` de boto3 y requiere dos componentes:

- **Objetivo escalable** (*scaling target*): fija los límites del escalado (p. ej., mínimo 1 instancia y máximo 10).
- **Política de escalado** (*scaling policy*): define la métrica que se vigila (p. ej., `SageMakerVariantInvocationsPerInstance` o `CPUUtilization`), su valor objetivo y cuándo escalar hacia dentro o hacia fuera.

**Ejemplo 1: escalado según invocaciones por instancia**

```python
import boto3

client = boto3.client('application-autoscaling')
resource_id = 'endpoint/your-endpoint-name/variant/AllTraffic'

# Registrar el objetivo escalable
client.register_scalable_target(
    ServiceNamespace='sagemaker',
    ResourceId=resource_id,
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    MinCapacity=1,
    MaxCapacity=10
)

# Crear la política de escalado
client.put_scaling_policy(
    PolicyName='ScalingPolicy',
    ServiceNamespace='sagemaker',
    ResourceId=resource_id,
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    PolicyType='TargetTrackingScaling',
    TargetTrackingScalingPolicyConfiguration={
        'TargetValue': 70.0,  # invocaciones por instancia y minuto
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'SageMakerVariantInvocationsPerInstance'
        },
        'ScaleInCooldown': 300,
        'ScaleOutCooldown': 300
    }
)
```

La política intenta mantener una media de **70 invocaciones por instancia y minuto**. Si la media sube de 70, añade instancias (*scale-out*); si baja, las reduce (*scale-in*). Así el endpoint se adapta de forma elástica al tráfico.

`ScaleInCooldown` y `ScaleOutCooldown` definen el **periodo de enfriamiento** (en segundos; aquí 300 s = 5 min) tras una acción de escalado. Durante ese periodo no se lanza otra acción del mismo tipo, lo que evita escalados rápidos y excesivos por fluctuaciones puntuales y da tiempo a que el sistema se estabilice.

**Ejemplo 2: escalado según el uso medio de CPU.** Se registra el objetivo escalable igual que antes y se cambia la política. `CPUUtilization` no es una métrica predefinida de Application Auto Scaling, así que se indica como **métrica personalizada** (`CustomizedMetricSpecification`):

```python
client.put_scaling_policy(
    PolicyName='CPUScalingPolicy',
    ServiceNamespace='sagemaker',
    ResourceId=resource_id,
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    PolicyType='TargetTrackingScaling',
    TargetTrackingScalingPolicyConfiguration={
        'TargetValue': 50.0,  # % medio de uso de CPU
        'CustomizedMetricSpecification': {
            'MetricName': 'CPUUtilization',
            'Namespace': '/aws/sagemaker/Endpoints',  # métricas de instancia del endpoint
            'Dimensions': [
                {'Name': 'EndpointName', 'Value': 'your-endpoint-name'},
                {'Name': 'VariantName', 'Value': 'AllTraffic'}
            ],
            'Statistic': 'Average',
            'Unit': 'Percent'
        },
        'ScaleInCooldown': 300,
        'ScaleOutCooldown': 300
    }
)
```

La política mantiene un **uso medio de CPU del 50 %**: si lo supera, añade instancias; si baja, las reduce. Los periodos de enfriamiento cumplen la misma función que en el ejemplo anterior. Ambos ejemplos usan **escalado de seguimiento de objetivo** (*target tracking*).

> **Para el examen**, otros tipos de escalado:
>
> - **Escalado por pasos** (*step scaling*): cuando una alarma supera un umbral, ajusta el número de instancias según escalones predefinidos (límite inferior, límite superior y cuánto escalar en cada uno).
> - **Escalado programado** (*scheduled scaling*): ajusta el número de instancias según un calendario.

Application Auto Scaling es totalmente compatible con **MME y MCE**. En los MCE, ten cuidado: si escalas con `InvocationsPerInstance`, los modelos de todos los contenedores deben tener un uso de CPU y una latencia por petición similares. Si el tráfico pasa de un modelo con poco uso de CPU a otro con mucho, pero el volumen total de invocaciones no cambia, el endpoint no escalará y no habrá instancias suficientes para el modelo más exigente.

**Variantes de producción y `AllTraffic`.** El `ResourceId` identifica una **variante de producción** del endpoint: `endpoint/<nombre>/variant/<variante>`. Una variante es una versión o configuración del modelo desplegada en el endpoint, con su propio tipo y número de instancias, y un endpoint puede tener varias. **`AllTraffic` es el nombre que el SageMaker Python SDK da por defecto a la variante** cuando despliegas un único modelo; no significa "todo el tráfico del endpoint". El autoescalado se configura **por variante**: en un endpoint con varias variantes (p. ej., `variant/VersionA`), cada una se registra y escala por separado. Esto resulta útil en las estrategias de la sección siguiente.

### Estrategias de despliegue y prueba

Toda estrategia de despliegue busca **minimizar el tiempo de inactividad** para los consumidores del modelo. Esto importa porque los modelos se actualizan y reevalúan con frecuencia: una versión nueva puede salir de reentrenar con datos más recientes, del ajuste de hiperparámetros (p. ej., con SageMaker AI AMT), de una mejor selección de características o de usar mejores instancias y contenedores de inferencia.

Con las **variantes de producción** de SageMaker AI puedes comparar modelos y elegir el mejor candidato. En un **endpoint multivariante** puedes repartir las invocaciones entre variantes asignando a cada una un **peso de tráfico**, o invocar directamente una variante concreta en cada petición (parámetro `TargetVariant`). Esta es la base de las **pruebas A/B**.

#### Despliegue Blue/Green

Reduce el riesgo y el tiempo de inactividad al desplegar una versión nueva. En SageMaker AI se implementa actualizando el endpoint (API `UpdateEndpoint`) con una **nueva configuración de endpoint** que contiene el nuevo modelo y una **configuración de despliegue** (`DeploymentConfig`); AWS llama a esta funcionalidad *deployment guardrails*. SageMaker AI aprovisiona la nueva flota y desplaza el tráfico de forma controlada, sin interrumpir el servicio.

Términos que debes conocer:

- **Flota azul** (*blue fleet*): infraestructura con la versión actual del modelo, que atiende el tráfico en producción.
- **Flota verde** (*green fleet*): infraestructura con la versión nueva, creada a partir de la nueva configuración del endpoint.
- **Periodo de *baking*** (*baking period*): intervalo que empieza cuando la flota verde recibe tráfico real. Durante ese tiempo, las **alarmas de CloudWatch** vigilan sus métricas para detectar problemas antes de enviarle más tráfico.
- **Modo de desplazamiento de tráfico** (*traffic shifting mode*): patrón con el que el endpoint reparte las peticiones entre las flotas azul y verde.
- **Rollback automático** (*auto-rollback*): las alarmas de CloudWatch con las que SageMaker AI vigila la flota verde. Si alguna se dispara, el 100 % del tráfico vuelve a la flota azul.

> **Recuerda:** las **alarmas de CloudWatch** son imprescindibles en Blue/Green: sin ellas no hay vigilancia durante el *baking* ni rollback automático.

El examen exige conocer tres modos de desplazamiento de tráfico: **All At Once**, **Canary** y **Linear**.

##### All At Once

SageMaker AI envía **el 100 % del tráfico a la flota verde** en un solo paso, y en ese momento empieza el periodo de *baking*. Si ninguna alarma se dispara, elimina la flota azul; si salta alguna, hace rollback automático y devuelve el 100 % del tráfico a la flota azul.

Ejemplo: `Model_v1` atiende el tráfico en producción desde la flota azul (los clientes lo invocan con la API [`invoke_endpoint`](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sagemaker-runtime/client/invoke_endpoint.html)). Para desplegar `Model_v2` se crea una nueva configuración de endpoint con ese modelo y se actualiza el endpoint. Durante el despliegue coexisten la flota azul (`Model_v1`) y la verde (`Model_v2`), y todo el tráfico pasa a `Model_v2` ([sintaxis de `update_endpoint`](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sagemaker/client/update_endpoint.html)):

```python
import boto3

client = boto3.client("sagemaker")
response = client.update_endpoint(
    EndpointName="<your-endpoint-name>",
    EndpointConfigName="<your-new-config-name>",   # configuración con Model_v2
    DeploymentConfig={
        "BlueGreenUpdatePolicy": {
            "TrafficRoutingConfiguration": {
                "Type": "ALL_AT_ONCE"
            },
            "TerminationWaitInSeconds": 600,
            "MaximumExecutionTimeoutInSeconds": 1800
        },
        "AutoRollbackConfiguration": {
            "Alarms": [{"AlarmName": "<your-cw-alarm>"}]
        }
    }
)
```

- `MaximumExecutionTimeoutInSeconds`: tiempo máximo que puede durar el despliegue antes de agotar el tiempo de espera (aquí 30 min).
- `TerminationWaitInSeconds`: tiempo que SageMaker AI espera, con la flota verde ya activa y tras el último *baking*, antes de terminar las instancias de la flota azul (aquí 10 min).

En el mejor caso no se dispara ninguna alarma y se termina la flota azul; si salta alguna, se hace rollback automático.

##### Canary

El desplazamiento es más gradual y se hace **en dos pasos**. Primero se envía una pequeña parte del tráfico a la flota verde (la **prueba canary**) mientras el resto sigue en la azul. Si durante el *baking* no salta ninguna alarma, el tráfico restante pasa a la flota verde. Los problemas se detectan pronto y afectan solo a un pequeño subconjunto de usuarios.

La llamada es la misma que en All At Once; solo cambia `TrafficRoutingConfiguration`:

```python
"TrafficRoutingConfiguration": {
    "Type": "CANARY",
    "CanarySize": {
        "Type": "CAPACITY_PERCENT",
        "Value": 30
    },
    "WaitIntervalInSeconds": 600
}
```

- `CanarySize` define el tamaño del canario: aquí se activa el **30 % de la capacidad de la flota verde**, que recibe la parte proporcional del tráfico. También puede expresarse como número de instancias (`INSTANCE_COUNT`).
- El tamaño del canario debe ser **igual o inferior al 50 % de la capacidad de la flota verde**.
- `WaitIntervalInSeconds` es la duración del *baking* del canario: aquí, 10 minutos después del primer desplazamiento se hace el segundo y último.

**Canary frente a All At Once.** All At Once mueve el 100 % del tráfico en un paso: es más rápido, pero más arriesgado, porque cualquier problema no detectado afecta a todos los usuarios de inmediato. Por eso depende mucho de las pruebas previas y de un buen plan de rollback. Canary es más seguro y controlado. La diferencia está en la **gestión del riesgo y la velocidad de transición**, y la elección depende de tu tolerancia al riesgo y de lo crítica que sea una experiencia de usuario ininterrumpida.

##### Linear

Es el modo con el **control más granular**. En lugar de uno (All At Once) o dos pasos (Canary), desplaza el tráfico de la flota azul a la verde **en incrementos pequeños e iguales** durante un periodo determinado. El tamaño de cada incremento se indica en número de instancias o en porcentaje (**10–50 %**) de la capacidad de la flota verde. Cada paso tiene su propio *baking*: si termina sin alarmas, la flota verde recibe más tráfico; si salta alguna en cualquier paso, todo el tráfico vuelve de inmediato a la flota azul.

```python
"TrafficRoutingConfiguration": {
    "Type": "LINEAR",
    "LinearStepSize": {
        "Type": "CAPACITY_PERCENT",
        "Value": 20
    },
    "WaitIntervalInSeconds": 300
}
# en BlueGreenUpdatePolicy: TerminationWaitInSeconds=300, MaximumExecutionTimeoutInSeconds=3600
```

El tráfico hacia la flota verde crece en incrementos del **20 %**, con 5 minutos de *baking* entre pasos (`WaitIntervalInSeconds=300`). Al llegar al 100 % sin alarmas, SageMaker AI espera otros 5 minutos (`TerminationWaitInSeconds=300`) y termina la flota azul. Si salta una alarma en cualquiera de los cinco pasos (0–20, 20–40, 40–60, 60–80, 80–100), el 100 % del tráfico vuelve a la flota azul.

Linear ofrece la **mayor reducción del riesgo**, a costa de despliegues potencialmente más largos. Es ideal para despliegues críticos, donde la estabilidad es esencial.

| Modo | Pasos | Parámetros de tamaño | Riesgo / velocidad |
|---|---|---|---|
| **All At Once** | 1 (100 %) | — | El más rápido y el más arriesgado |
| **Canary** | 2 (canario + resto) | `CanarySize` (≤ 50 % de la flota verde) + `WaitIntervalInSeconds` | Riesgo bajo |
| **Linear** | N incrementos iguales | `LinearStepSize` (10–50 %) + `WaitIntervalInSeconds` | El riesgo más bajo y el más lento |

---

## Orquestación de flujos de trabajo de ML

Hasta aquí has visto cómo ingerir datos, crear características, elegir un algoritmo, entrenar, optimizar y desplegar el modelo. Hacer todos estos pasos manualmente es lento y propenso a errores. SageMaker AI ofrece un entorno integrado para automatizar el flujo completo, y su herramienta central para ello es **Amazon SageMaker Pipelines**.

### Amazon SageMaker Pipelines

**MLOps** (operacionalización de modelos de ML) une el desarrollo de modelos con la producción: integración, despliegue, monitorización y mantenimiento. La **orquestación** es la automatización y gestión del ciclo de vida completo de ML. SageMaker Pipelines ofrece un marco estructurado para definir, programar y monitorizar flujos de ML de forma **coherente y repetible**, lo que aumenta la fiabilidad y reduce el tiempo de salida al mercado.

Ventajas frente a otras soluciones de flujos de trabajo de AWS:

- **Infraestructura serverless con autoescalado:** no gestionas la infraestructura de orquestación; SageMaker AI la aprovisiona, escala y apaga según la carga.
- **Varias interfaces:** editor visual de SageMaker Studio (arrastrar y soltar), SDK, API o definición en JSON. El SDK forma parte del SageMaker Python SDK ([documentación de Pipelines](https://sagemaker.readthedocs.io/en/stable/workflows/pipelines/index.html)).
- **Integración** con todas las funcionalidades de SageMaker AI y con otros servicios de AWS para automatizar procesamiento de datos, entrenamiento, *fine-tuning*, evaluación, despliegue y monitorización.
- **Coste:** Pipelines no tiene coste propio; solo pagas por los trabajos que orquesta (procesamiento, entrenamiento, etc.).

Antes de ver Pipelines en acción, conviene repasar tres conceptos clave de MLOps: repositorio de código, CI/CD y orquestación.

### Repositorio de código y control de versiones

Un **repositorio de código** (*repo*) es el lugar central donde el equipo almacena, comparte y gestiona el código y los archivos relacionados. Se aloja en plataformas como GitHub, GitLab o Bitbucket, que facilitan el desarrollo colaborativo: varias personas trabajan a la vez en el mismo código, siguen los cambios y contribuyen al proyecto. Con SageMaker Pipelines, el repositorio organiza los scripts, notebooks y demás artefactos necesarios para construir, entrenar y desplegar modelos.

El **control de versiones** registra los cambios de los archivos a lo largo del tiempo. **Git** es el sistema más usado: guarda instantáneas del código llamadas **commits**, cada una con sus cambios y metadatos (autor y fecha). Permite volver a versiones anteriores, comparar commits y fusionar contribuciones. En ML es esencial por la naturaleza iterativa del desarrollo, porque permite revisar o revertir los cambios en scripts de preprocesamiento, código de entrenamiento o configuraciones de despliegue, con lo que el flujo sigue siendo repetible y transparente.

### Amazon SageMaker Model Registry

El artefacto de un modelo es el resultado de entrenarlo y ajustarlo, así que hay que gestionarlo durante todo su ciclo de vida. **SageMaker Model Registry**, una funcionalidad totalmente gestionada de SageMaker AI, cataloga y organiza los modelos:

- Lleva el control de las **versiones** de cada modelo, desde el desarrollo hasta la producción.
- Centraliza **metadatos, métricas de rendimiento e historial** de cada versión.
- Asigna a cada versión un **estado de aprobación** (pendiente, aprobado o rechazado); aprobar una versión puede disparar su **despliegue automático** en un pipeline de CI/CD.
- Facilita la colaboración, la **gobernanza y el cumplimiento normativo** con un registro auditable de cambios y despliegues.

Se integra con SageMaker Pipelines para automatizar el registro, el versionado y la gestión del ciclo de vida de los modelos, de modo que queden versionados, sean reproducibles y puedan recuperarse con facilidad.

### CI/CD e infraestructura como código

Un pipeline de **CI/CD** automatiza la integración de cambios de código, la construcción de la aplicación, la ejecución de pruebas y el despliegue en producción. Servicios de AWS:

| Servicio | Función |
|---|---|
| **AWS CodeArtifact** | Repositorio gestionado de artefactos: almacenar, publicar y compartir paquetes de software de forma segura |
| **AWS CodeBuild** | Servicio de compilación gestionado: compila el código, ejecuta pruebas y genera paquetes listos para desplegar |
| **AWS CodeDeploy** | Automatiza el despliegue de aplicaciones en EC2, Lambda y servidores *on-premises* |
| **AWS CodePipeline** | Servicio de CI/CD que modela, visualiza y automatiza los pasos para publicar el software |

Como la infraestructura moderna está virtualizada, gestionarla a mano es poco práctico y propenso a errores. La **infraestructura como código (IaC)** define la infraestructura en archivos de configuración legibles por máquina, de forma declarativa, lo que la hace **repetible, coherente y versionable** como el código de la aplicación:

- **AWS CloudFormation:** describe y aprovisiona los recursos con plantillas de texto (**JSON o YAML**) de forma automatizada y segura, en todas las regiones y cuentas.
- **Terraform:** herramienta de IaC de HashiCorp con un flujo de CLI común para gestionar cientos de servicios en la nube. Usa su propio lenguaje, **HCL** (HashiCorp Configuration Language).
- **AWS CDK (Cloud Development Kit):** CloudFormation y Terraform obligan a usar JSON, YAML o HCL. CDK permite definir la infraestructura con lenguajes de programación (**JavaScript, TypeScript, Python, Java, C# y Go**), y el código CDK se convierte en plantillas de CloudFormation.

### Orquestación MLOps: AWS Step Functions y Amazon MWAA

MLOps aplica los principios de DevOps al ML (integración, entrega y despliegue continuos). Además de SageMaker Pipelines, el examen exige conocer otras dos herramientas de orquestación.

**AWS Step Functions** es un servicio **serverless** que coordina varios servicios de AWS en flujos de trabajo escalables. Los flujos se definen como **máquinas de estados** compuestas por tareas y se diseñan y monitorizan desde una **interfaz visual**. Step Functions garantiza que cada paso se ejecute en el orden correcto, **gestiona los errores** y admite **flujos paralelos a gran escala**. Con SageMaker AI puede orquestar todo el ciclo de vida: preprocesamiento, ingeniería de características, entrenamiento, ajuste y despliegue.

**Amazon Managed Workflows for Apache Airflow (MWAA)** es un servicio gestionado de **Apache Airflow**, la plataforma de código abierto para definir por programación, programar y monitorizar flujos de trabajo, especialmente pipelines de datos complejos. AWS se encarga del aprovisionamiento, el escalado y el mantenimiento de los entornos de Airflow, que se integran con S3, Redshift y EMR. En MLOps, puede orquestar la extracción, transformación y carga (ETL), el entrenamiento, la evaluación y el despliegue.

### Cómo elegir la herramienta de orquestación

Depende de la complejidad de los flujos y de la familiaridad del equipo con cada herramienta:

| Herramienta | Tipo | Ideal para |
|---|---|---|
| **SageMaker Pipelines** | Específica para ML | Definir, visualizar y automatizar flujos de ML de principio a fin, con integración nativa con el ecosistema de SageMaker AI |
| **AWS Step Functions** | Orquestación general, serverless | Pipelines de principio a fin que integran servicios de AWS, con interfaz visual y mínima carga operativa; equipos que buscan simplicidad |
| **Amazon MWAA** | Orquestación general, Airflow gestionado | Flujos complejos y personalizables con un rico ecosistema de plugins; ingeniería de datos y equipos que ya usan Airflow |

En el examen, **SageMaker Pipelines** es la herramienta principal, y es en la que se centran las secciones siguientes.

---

## Automatizar la construcción y el despliegue de modelos con SageMaker Pipelines

Automatizar con SageMaker Pipelines consiste en crear una secuencia de pasos bien definidos que forman el flujo de ML. Su gran beneficio es la **repetibilidad**: convierte tareas manuales en procesos que se pueden recrear de forma fiable y producen resultados coherentes. Esto es clave para validar modelos, experimentar y que otros miembros del equipo reproduzcan los resultados, dada la naturaleza iterativa del ML. Además, libera tiempo para tareas de más valor y reduce los errores humanos. Cada paso es como una pieza de LEGO: al ensamblar las piezas se obtiene el pipeline completo.

Como cada componente del flujo es un paso independiente, se puede desarrollar y optimizar por separado. Después los pasos se combinan en un objeto `Pipeline`, que forma un **grafo dirigido acíclico (DAG)**: el orden de ejecución lo determinan las **dependencias entre pasos**, no el orden de la lista, y los pasos sin dependencias entre sí pueden ejecutarse en paralelo. El pipeline puede ejecutarse **bajo demanda, de forma programada o en respuesta a eventos**, de modo que los modelos se actualizan a medida que llegan datos nuevos.

A continuación se diseña un pipeline sencillo.

### 1. Definir los pasos del flujo

| Componente | Qué hace |
|---|---|
| Preprocesamiento de datos | Limpiar y transformar los datos en bruto |
| Ingeniería de características | Enriquecer los datos para mejorar la precisión del modelo |
| Entrenamiento | Construir el modelo con un algoritmo |
| Evaluación | Medir el rendimiento del modelo |
| Despliegue | Llevar el modelo entrenado a producción |

Cada componente se corresponde con uno o varios pasos del pipeline.

### 2. Crear y configurar los pasos

El SageMaker Python SDK ofrece una clase para cada tipo de paso, así que no hay que "reinventar la rueda":

| Clase | Qué ejecuta | Uso en el ejemplo |
|---|---|---|
| `ProcessingStep` | Un trabajo de SageMaker Processing (script propio) | Preprocesamiento de datos. También sirve para ingeniería de características o evaluación |
| `TransformStep` | Un trabajo de *batch transform* (aplica un modelo existente a un dataset completo) | Ingeniería de características con un modelo de transformación ya creado |
| `TrainingStep` | Un trabajo de entrenamiento | Entrenamiento del modelo |
| `ModelStep` | Crea el modelo en SageMaker AI (`model.create()`) o lo registra en Model Registry (`model.register()`) | Crear el modelo desplegable |

> **Ojo:** `ModelStep` **no crea un endpoint**. Pipelines no tiene un paso nativo de despliegue: el endpoint se crea después, por ejemplo con un paso que invoque una función Lambda o con un pipeline de CI/CD que se dispare al aprobar el modelo en Model Registry.
>
> Los fragmentos de este capítulo usan la API **v2** del SageMaker Python SDK. En la v3 estos módulos cambiaron, así que, para ejecutarlos tal cual, necesitas `sagemaker<3`.

Primero se crea una sesión de pipeline:

```python
from sagemaker.workflow.pipeline_context import PipelineSession

pipeline_session = PipelineSession()
```

**Paso de preprocesamiento:**

```python
from sagemaker.processing import ScriptProcessor
from sagemaker.workflow.steps import ProcessingStep

processor = ScriptProcessor(
    image_uri='your_image_uri',
    command=['python3'],          # obligatorio: comando que ejecuta el script
    role='your_iam_role',
    instance_count=1,
    instance_type='ml.m5.large',
    sagemaker_session=pipeline_session
)

processing_step = ProcessingStep(
    name="DataPreprocessing",
    processor=processor,
    inputs=[...],
    outputs=[...],
    code="preprocessing_script.py"
)
```

**Paso de ingeniería de características** (*batch transform* con un modelo de transformación existente):

```python
from sagemaker.transformer import Transformer
from sagemaker.inputs import TransformInput
from sagemaker.workflow.steps import TransformStep

transformer = Transformer(
    model_name='your_model_name',
    instance_count=1,
    instance_type='ml.m5.large',
    output_path='s3://your-bucket/transform-output',
    sagemaker_session=pipeline_session
)

feature_engineering_step = TransformStep(
    name="FeatureEngineering",
    transformer=transformer,
    inputs=TransformInput(data='s3://your-bucket/processed-data')
)
# Ejecutar después del preprocesamiento
feature_engineering_step.add_depends_on([processing_step])
```

**Paso de entrenamiento:**

```python
from sagemaker.estimator import Estimator
from sagemaker.workflow.steps import TrainingStep

estimator = Estimator(
    image_uri='your_image_uri',
    role='your_iam_role',
    instance_count=1,
    instance_type='ml.m5.large',
    hyperparameters={'max_depth': 5, 'eta': 0.2},
    sagemaker_session=pipeline_session
)

training_step = TrainingStep(
    name="ModelTraining",
    estimator=estimator,
    inputs={
        'train': 's3://your-bucket/train-data',
        'validation': 's3://your-bucket/validation-data'
    }
)
# Ejecutar después de la ingeniería de características
training_step.add_depends_on([feature_engineering_step])
```

**Paso de creación del modelo:**

```python
from sagemaker.model import Model
from sagemaker.workflow.model_step import ModelStep

model = Model(
    image_uri='your_inference_image_uri',
    model_data='s3://your-bucket/model.tar.gz',
    role='your_iam_role',
    entry_point='inference_script.py',
    sagemaker_session=pipeline_session
)

model_step = ModelStep(
    name="CreateModel",
    step_args=model.create(instance_type='ml.m5.large'),
    depends_on=[training_step]    # ejecutar después del entrenamiento
)
```

### 3. Definir el pipeline

Los pasos se combinan en un objeto `Pipeline`. Antes de ejecutarlo hay que **crearlo o actualizarlo en SageMaker AI** con `upsert()`, que recibe el rol IAM con el que se ejecutará:

```python
from sagemaker.workflow.pipeline import Pipeline

pipeline = Pipeline(
    name="MyMLPipeline",
    steps=[processing_step, feature_engineering_step, training_step, model_step]
)
pipeline.upsert(role_arn='your_iam_role')
```

**Gestión de errores.** Si un paso falla, la ejecución se detiene y queda marcada como fallida. Las notificaciones al equipo no son automáticas: se configuran, por ejemplo, con reglas de Amazon EventBridge sobre los cambios de estado de la ejecución que publiquen en Amazon SNS.

### 4. Configurar disparadores y programación

La ejecución puede automatizarse con **Amazon EventBridge**, ya sea en respuesta a eventos o de forma programada. Esta regla lanza `MyMLPipeline` cuando se sube un objeto nuevo a S3. El destino es el **ARN del pipeline de SageMaker** e incluye un **rol IAM** que permita a EventBridge iniciarlo:

```python
import boto3

client = boto3.client('events')

event_pattern = '''{
  "source": ["aws.s3"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {"eventSource": ["s3.amazonaws.com"], "eventName": ["PutObject"]}
}'''

client.put_rule(
    Name='MyMLPipelineRule',
    EventPattern=event_pattern,
    State='ENABLED',
    Description='Rule to trigger MyMLPipeline when a new object is uploaded to S3'
)

client.put_targets(
    Rule='MyMLPipelineRule',
    Targets=[{
        'Id': '1',
        'Arn': 'arn:aws:sagemaker:us-east-1:123456789012:pipeline/mymlpipeline',
        'RoleArn': 'arn:aws:iam::123456789012:role/EventBridgeStartPipelineRole'
    }]
)
```

Este patrón se basa en eventos de CloudTrail, así que requiere un *trail* que registre los eventos de datos de S3 del bucket.

### 5. Ejecutar el pipeline

También puedes lanzar la ejecución manualmente con el SDK. Los pasos sin dependencias entre sí se ejecutan en paralelo, lo que reduce el tiempo total:

```python
execution = pipeline.start()
```

### Consideraciones clave

- **Escalabilidad:** Pipelines gestiona y escala automáticamente la infraestructura necesaria.
- **Monitorización y logs:** son **responsabilidad del ingeniero de ML**, que debe seguir el rendimiento del pipeline y diagnosticar problemas.
- **Gestión de errores:** un fallo detiene la ejecución; hay que configurar las alertas para que el equipo actúe a tiempo.
- **Concurrencia:** los pasos independientes se ejecutan a la vez, lo que aprovecha mejor los recursos y acorta la ejecución.

---

## Ejemplo práctico: despliegue serverless del mejor modelo

Este ejemplo amplía el ejemplo de ajuste de hiperparámetros del capítulo 5: evaluar XGBoost sobre el dataset **Digits** con **SageMaker AI Automatic Model Tuning (AMT)** y estrategia **bayesiana**. Aquí se expone el mejor modelo (el de mayor exactitud) con un **endpoint de inferencia serverless** y se prueba con una petición. Para ahorrar coste y tiempo se usan 3 trabajos de ajuste en lugar de 20. Las peticiones son representaciones en CSV de dígitos manuscritos, y el modelo predice la clase del dígito. El autor ejecutó el programa en el Code Editor de SageMaker Studio con el SageMaker Python SDK v2.

La primera parte del programa, que repite el capítulo 5, hace lo siguiente:

- Carga Digits y lo divide en entrenamiento y validación (80/20).
- Guarda ambos conjuntos en CSV con la etiqueta en la primera columna y los sube a S3.
- Define un estimador con el contenedor XGBoost `1.2-1` (`objective='multi:softmax'`, `num_class=10`, `ml.m5.large`).
- Lanza un `HyperparameterTuner` bayesiano que maximiza `validation:accuracy` ajustando `alpha`, `eta`, `min_child_weight` y `max_depth`, con `max_jobs=3` y `max_parallel_jobs=3`.
- Obtiene el mejor trabajo con `tuner.best_training_job()` y guarda un gráfico de los resultados del ajuste.

La parte de despliegue es la siguiente:

```python
from sagemaker.serverless import ServerlessInferenceConfig
from sagemaker.model import Model
from sagemaker.serializers import CSVSerializer
from sagemaker.deserializers import JSONDeserializer

# ... datos, estimador y ajuste como en el capítulo 5 ...
best_training_job_name = tuner.best_training_job()

# Configuración serverless: memoria y concurrencia máxima
serverless_config = ServerlessInferenceConfig(
    memory_size_in_mb=2048,
    max_concurrency=5
)

# Mejor modelo: artefacto del mejor trabajo + imagen de XGBoost
best_model = Model(
    image_uri=xgb_image,
    model_data=f's3://{bucket_name}/{prefix}/output/{best_training_job_name}/output/model.tar.gz',
    role=role,
    sagemaker_session=session
)

# Despliegue como endpoint serverless con un nombre único
predictor = best_model.deploy(
    serverless_inference_config=serverless_config,
    endpoint_name='xgboost-digits-serverless-endpoint',
    serializer=CSVSerializer(),
    deserializer=JSONDeserializer()
)

# Inferencia de ejemplo con la primera muestra de prueba, codificada como CSV
test_sample_csv = ','.join(map(str, X_test[0]))
result = predictor.predict(test_sample_csv)
print('Prediction for the first test sample:', result)
```

Cómo despliega el modelo:

1. **`ServerlessInferenceConfig`** fija el tamaño de memoria y la concurrencia máxima. SageMaker AI gestiona los recursos de cómputo de forma dinámica, sin instancias dedicadas.
2. **`Model`** apunta al artefacto `model.tar.gz` del mejor trabajo de entrenamiento en S3 y a la URI de la imagen de XGBoost (`xgboost:1.2-1`), compatible con el formato serializado del modelo.
3. **`deploy()`** orquesta el despliegue: SageMaker AI descarga y descomprime `model.tar.gz` en el contenedor, y el contenedor carga el modelo XGBoost en memoria, listo para atender peticiones.
4. **Serialización:** el endpoint espera CSV, así que la entrada se envía con `CSVSerializer` y la respuesta se lee con `JSONDeserializer`. El endpoint lleva un nombre único para evitar conflictos.
5. **Inferencia:** se envía una muestra de prueba como cadena CSV y se recibe la clase predicha.

**Resultados:**

- El modelo predice correctamente la muestra de prueba, que corresponde al dígito **6**.
- En SageMaker Studio, el endpoint aparece con estado **InService**, junto con su modelo, su variante, la concurrencia máxima y el tamaño de memoria configurados en el código.
- Studio permite **probar el endpoint desde la interfaz** enviando la muestra en CSV.
- El gráfico del ajuste muestra los hiperparámetros del modelo elegido por la estrategia bayesiana.

En resumen, se ha desplegado un modelo por programación en un endpoint serverless que aprovecha los contenedores gestionados de SageMaker AI y atiende las peticiones de inferencia de forma eficiente y escalable, sin gestión manual de recursos.

---

## Resumen

- **Servicios de IA de AWS:** Rekognition, Textract, Polly, Transcribe, Translate, Comprehend y Lex ofrecen modelos preentrenados; Personalize entrena modelos con tus datos. Todos se consumen mediante API y sus modelos viven en cuentas gestionadas por AWS, así que tu acceso a ellos es limitado. Se integran rápido y sin gestionar modelos. **Amazon Bedrock** da acceso a FM mediante `InvokeModel` y la **Converse API** (interacciones conversacionales) o mediante **agentes** (orquestación de tareas de varios pasos).
- **Despliegue gestionado en SageMaker AI:** **tiempo real** (baja latencia), **serverless** (escalado automático para tráfico intermitente, como en el ejemplo práctico), **asíncrona** (cargas grandes y procesamiento largo) y **por lotes** (grandes datasets sin endpoint).
- **Despliegue no gestionado:** EC2, ECS, EKS, Lambda o servidores *on-premises*. Requiere más esfuerzo de configuración, escalado y mantenimiento, a cambio de más control y personalización.
- **SageMaker Neo** compila modelos para hardware concreto (*edge*), lo que reduce la latencia y mejora el rendimiento.
- **Autoescalado** y **Blue/Green** (All At Once, Canary, Linear) permiten actualizar modelos de forma segura y controlada.
- **Orquestación:** SageMaker Pipelines automatiza el flujo desde la preparación de datos hasta el despliegue; Step Functions y MWAA complementan la orquestación con flujos complejos y lógica condicional. El resultado son procesos de ML repetibles y mantenibles.

---

## Puntos clave para el examen

- **Infraestructura de inferencia frente a entrenamiento:** la inferencia suele requerir mucho menos cómputo y memoria, porque solo aplica el modelo entrenado a datos nuevos. El entrenamiento, en cambio, aprende y optimiza los parámetros sobre grandes datasets.
- **Las cuatro opciones gestionadas de SageMaker AI:**
  - **Tiempo real:** endpoints de baja latencia para aplicaciones interactivas.
  - **Serverless:** escala según el tráfico; ideal para patrones intermitentes o impredecibles.
  - **Asíncrona:** cargas grandes y tiempos de procesamiento largos.
  - **Por lotes:** grandes datasets sin necesidad de tiempo real.
- **MME frente a MCE:**
  - **MME:** muchos modelos con poco tráfico que comparten el **mismo contenedor** en un endpoint, para ahorrar costes.
  - **MCE:** modelos con **dependencias o frameworks distintos**, cada uno en su contenedor (hasta 15), en un endpoint. Los contenedores aíslan dependencias, pero comparten las instancias y escalan juntos.
- **CPU frente a GPU:**
  - **GPU:** trabajos intensivos, como entrenar deep learning con muchos datos.
  - **CPU:** tareas sin paralelismo masivo, como preprocesamiento, ingeniería de características, entrenamientos pequeños e inferencia con poca demanda de cómputo.
  - **Inferentia (Inf1/Inf2):** inferencia de deep learning al menor coste.
- **Métricas de autoescalado:**
  - `SageMakerVariantInvocationsPerInstance`: invocaciones medias por instancia y minuto.
  - `CPUUtilization`: % de CPU.
  - `ModelLatency`: tiempo de respuesta del modelo.
  - `GPUUtilization`: % de GPU.
  - Tipos de escalado: seguimiento de objetivo, por pasos y programado.
- **Modos de Blue/Green:**
  - **All At Once:** mucha confianza en la versión nueva y necesidad de una transición rápida.
  - **Canary:** minimizar el riesgo probando con una pequeña parte del tráfico antes del despliegue completo.
  - **Linear:** desplazamiento controlado paso a paso con monitorización en cada incremento.
  - Todos dependen de **alarmas de CloudWatch** para el rollback automático.
- **Servicios de orquestación:**
  - **SageMaker Pipelines:** flujos de ML de principio a fin.
  - **Step Functions:** coordinar flujos de varios pasos con servicios de AWS, lógica condicional y recuperación de errores.
  - **MWAA:** Airflow gestionado para ETL o pipelines de ML complejos con poca carga operativa.

---

## Preguntas de repaso

1. ¿Qué tipo de instancia elegirías para un endpoint multimodelo de SageMaker AI que aloja varios modelos de deep learning entrenados con grandes datasets, para optimizar coste y rendimiento?
   - A. t2.medium
   - B. m5.xlarge
   - C. p3.8xlarge
   - D. c5.4xlarge

2. Al configurar un endpoint de SageMaker AI, ¿en qué circunstancias elegirías un despliegue gestionado en lugar de uno no gestionado?
   - A. Cuando necesitas control total de la infraestructura de despliegue.
   - B. Cuando necesitas escalabilidad sin fricciones y asignación de recursos gestionada.
   - C. Cuando despliegas modelos en entornos ajenos a AWS.
   - D. Cuando necesitas personalizar a fondo la configuración del despliegue.

3. ¿Qué patrón de desplazamiento de tráfico es más adecuado para desplegar una actualización crítica de un modelo con el mínimo riesgo y una monitorización exhaustiva antes del despliegue completo?
   - A. All At Once
   - B. Linear
   - C. Canary
   - D. Partial

4. ¿Cuál es la principal ventaja de usar SageMaker Neo para desplegar modelos en un entorno IoT?
   - A. Reducir el coste de entrenar modelos.
   - B. Garantizar una alta precisión de las predicciones.
   - C. Optimizar los modelos para hardware diverso y permitir despliegues eficientes en el *edge*.
   - D. Acelerar el preprocesamiento de datos.

5. ¿Qué servicio de orquestación usarías para automatizar flujos de ML de principio a fin con la mínima intervención manual?
   - A. Amazon SageMaker Pipelines
   - B. Amazon Managed Workflows for Apache Airflow (MWAA)
   - C. AWS Step Functions
   - D. Amazon SageMaker Clarify

6. ¿Qué tipo de instancia EC2 elegirías para una carga de inferencia que requiere alto *throughput* y baja latencia con modelos de ML propios?
   - A. t3.medium
   - B. r5.4xlarge
   - C. g4dn.xlarge
   - D. inf1.2xlarge

7. ¿Cuándo elegirías SageMaker Serverless Inference como opción de despliegue gestionado?
   - A. Cuando necesitas predicciones en tiempo real de baja latencia.
   - B. Cuando tienes patrones de tráfico intermitentes o impredecibles.
   - C. Cuando necesitas procesamiento por lotes de alto *throughput*.
   - D. Cuando necesitas inferencia sobre cargas grandes con tiempos de procesamiento largos.

8. ¿Cuándo elegirías AWS Step Functions en lugar de SageMaker Pipelines y MWAA para orquestar flujos de ML?
   - A. Cuando necesitas automatizar tareas ETL sencillas.
   - B. Cuando necesitas un servicio gestionado para orquestar flujos de ML complejos con ramificación condicional y gestión de errores.
   - C. Cuando necesitas plantillas predefinidas para proyectos de ML.
   - D. Cuando necesitas detectar sesgos y explicar las predicciones del modelo.

9. ¿Qué funcionalidades clave ofrece SageMaker Model Registry y cuándo deberías usarlo en un pipeline de CI/CD?
   - A. Ofrece monitorización de modelos y detección de sesgos; úsalo para monitorizar continuamente el rendimiento en producción.
   - B. Ofrece versionado de modelos, flujos de aprobación y automatización del despliegue; úsalo para gestionar y desplegar modelos de forma eficiente en un pipeline de CI/CD.
   - C. Simplifica el preprocesamiento de datos y la ingeniería de características; úsalo para preparar datasets de entrenamiento.
   - D. Se integra con AWS Step Functions para orquestar flujos de ML complejos; úsalo para coordinar varias tareas de ML.

10. ¿Cuándo deberías usar la Converse API de Amazon Bedrock?
    - A. Para gestionar y desplegar modelos de ML.
    - B. Para tareas automatizadas de comprensión y generación de lenguaje natural en tiempo real.
    - C. Para preprocesamiento de datos e ingeniería de características.
    - D. Para optimizar modelos para distintas plataformas de hardware.
