---
tema: "Capítulo 4 — Selección de modelos (versión explicada: AWS e infraestructura)"
fuente: Guia Oficial/04_model_selection.txt
guia-mla-c01: [Dominio 2, 2.1 Choose a modeling approach, 2.2 Train and refine models]
perfil-lector: profesional de datos/estadística sin experiencia administrando infraestructura
alcance-explicaciones: solo AWS e infraestructura; los conceptos de ML y estadística se conservan sin explicación añadida
versiones-codigo:
  python: "3.12.6"
  scikit-learn: "1.5.2"
  xgboost: "3.0.5"
  numpy: "2.0.2"
  sagemaker-sdk-del-libro: "2.x"
  sagemaker-sdk-instalado-localmente: "3.22.1"
verificado: 2026-09-25
tags:
  [
    aws,
    mla-c01,
    mla-c02,
    seleccion-de-modelos,
    servicios-de-ia,
    rekognition,
    textract,
    polly,
    transcribe,
    translate,
    comprehend,
    lex,
    personalize,
    bedrock,
    sagemaker,
    algoritmos-integrados,
  ]
---

> [!info] Cómo leer esta versión
>
> - **Qué es.** Traducción íntegra al español del capítulo 4 de la guía oficial de estudio, con los mismos encabezados, ejemplos, fragmentos de código y afirmaciones. No se eliminó nada. Los encabezados conservan entre paréntesis el título original en inglés, porque el examen se presenta en inglés.
> - **Qué se añadió.** Explicaciones de los términos de AWS y de infraestructura integradas en el texto, aclaraciones de razonamientos que el libro da por sentados, **notas de precisión** (recuadros amarillos) cuando el original es impreciso o está desactualizado, la sección «Escenarios donde este servicio es la opción obligada» y un glosario al final.
> - **Qué no se añadió.** Las explicaciones nuevas se limitan a AWS e infraestructura: cómo se configura cada servicio, qué recursos usa, dónde guarda los datos, qué permisos necesita, cómo escala y qué administra AWS frente a lo que administras tú. Los conceptos de ML y estadística (regresión, SVM, k-NN, árboles, PCA, métricas, sobreajuste, etc.) aparecen tal como los presenta el libro, sin desarrollo adicional.
> - **Figuras y fórmulas.** El `.txt` no incluye las imágenes, así que solo quedan los pies de figura. Cuando la figura mostraba una salida de texto de un programa que se puede ejecutar sin AWS, aquí va la salida real obtenida con las versiones del encabezado. Varias fórmulas se perdieron en la extracción del PDF y aparecen como huecos; se reconstruyeron en su forma estándar y se marcan como reconstruidas.
> - **Código.** Se corrigieron defectos de la extracción del PDF (líneas partidas a la mitad, cadenas cortadas). La lógica no se modificó. Cuando un fragmento de SageMaker no funciona tal cual, se explica en una nota y se da la corrección.
> - **Cifras y estado de los servicios.** Se verificaron en la documentación de AWS el 25 de septiembre de 2026. Cambian con frecuencia: confírmalos en la documentación oficial vigente antes de usarlos en una decisión real.

> [!warning] Cambios en AWS posteriores a la edición del libro (verificado el 25-09-2026)
>
> - **Amazon SageMaker** pasó a llamarse **Amazon SageMaker AI**; el propio libro lo advierte más adelante. En este texto se usa el nombre del libro.
> - **SDK de SageMaker para Python v3.** Todos los fragmentos de SageMaker del libro usan la versión 2 del SDK (`sagemaker.estimator.Estimator`, `sagemaker.image_uris`, `sagemaker.Session`, `get_execution_role`). En la versión 3, que es la que instala hoy `pip install sagemaker` (en esta máquina está la 3.22.1), esos módulos ya no existen: el entrenamiento se hace con `ModelTrainer` (paquete `sagemaker.train`) y el despliegue con `ModelBuilder` (`sagemaker.serve`). Para ejecutar los fragmentos del libro sin reescribirlos, crea un entorno virtual aparte con `pip install "sagemaker>=2,<3"`.
> - **Amazon Bedrock.** Desde octubre de 2025 los modelos serverless se habilitan automáticamente y ya no hay que «solicitar acceso» modelo por modelo, salvo un formulario único de caso de uso para los modelos de Anthropic. Los modelos **Titan Text G1** (Premier, Express y Lite) que recomienda el libro ya no aparecen en el catálogo. **Nova Canvas**, el modelo del ejemplo práctico, está en estado _Legacy_ desde el 30-03-2026 (los clientes nuevos ya no pueden usarlo) y llega a su fin de vida el **30-09-2026**.
> - **Amazon Comprehend.** El modelado de temas, la detección de eventos y la clasificación de seguridad de prompts no admiten clientes nuevos desde el 30-04-2026.
> - **Amazon Rekognition.** Las funciones _Streaming Events_ (alertas para hogares conectados) y moderación de contenido de imágenes por lotes no admiten clientes nuevos desde el 30-04-2026.
> - **Amazon Lex V1** llegó a su fin de vida el 15-09-2025. Todo lo que se diga de Lex aplica hoy a **Lex V2**.
> - **SageMaker Studio.** En la versión actual de Studio, los espacios de JupyterLab y Code Editor guardan el directorio de trabajo en un volumen **EBS**, no en EFS como dice el libro (ver la nota en la sección de Bedrock).

> [!tip] Relevancia para el examen (MLA-C01 frente a MLA-C02)
> El último día para presentar el MLA-C01 en inglés es el **28-09-2026**; el MLA-C02 entra en beta el 29-09-2026. Los servicios de IA y los algoritmos integrados de este capítulo siguen siendo materia de ambos exámenes. Dos matices para el C02: ya no incluye la configuración de datos de entrenamiento en EFS o FSx, que el libro menciona varias veces, y exige mucho más de IA generativa (Knowledge Bases, agentes, Guardrails, RAG, evaluación de modelos) de lo que cubre aquí la sección de Bedrock.

**Capítulo 4**

# Selección de modelos (_Model Selection_)

> LOS OBJETIVOS DEL EXAMEN AWS CERTIFIED MACHINE LEARNING (ML) ENGINEER – ASSOCIATE QUE CUBRE ESTE CAPÍTULO PUEDEN INCLUIR, ENTRE OTROS, LOS SIGUIENTES:
>
> ✔ Dominio 2: Desarrollo de modelos de ML
>
> - 2.1 Elegir un enfoque de modelado
> - 2.2 Entrenar y refinar modelos

En el capítulo anterior exploramos los pasos importantes de la transformación de datos y la ingeniería de características, que sientan las bases de aplicaciones eficaces de machine learning (ML). En este capítulo pasamos a la emocionante fase de seleccionar un modelo en función de tu problema de ML.

Con referencia al ciclo de vida de ML (Figura 4.1), en este capítulo nos centramos en elegir un algoritmo de ML (o un servicio de IA) adecuado y adaptado a tus requisitos específicos de ML.

_Figura 4.1 El ciclo de vida de ML._

Aprenderás a seleccionar el algoritmo de ML más adecuado para tu caso de uso específico y a implementarlo eficazmente para iniciar el proceso de desarrollo del modelo. Recuerda que el desarrollo de un modelo es un proceso iterativo por naturaleza: implica refinarlo y evaluarlo continuamente para asegurar el mejor rendimiento y la mayor exactitud posibles. Al dominar este enfoque iterativo, estarás bien preparado para abordar desafíos de datos complejos y lograr resultados de impacto.

Este capítulo te guiará por el intrincado pero fascinante proceso de transformar datos crudos en potentes modelos predictivos que pueden impulsar _insights_ y decisiones accionables.

La selección de un algoritmo de ML apropiado es un paso crucial del desarrollo del modelo. Esta decisión depende de una comprensión clara de tu caso de uso, de la naturaleza de tus datos y del problema específico que buscas resolver. Ya sea que enfrentes una tarea de clasificación, un problema de regresión o una necesidad de clustering, elegir el algoritmo correcto implica equilibrar factores como la interpretabilidad, la exactitud, la eficiencia computacional, la escalabilidad y el costo. En AWS, los tres últimos factores se traducen en preguntas concretas de infraestructura. La **escalabilidad** pregunta si el algoritmo puede entrenar en varias máquinas a la vez o solo en una, si puede leer los datos en flujo continuo desde el almacenamiento o necesita copiarlos antes a disco, y si el servicio que sirve el modelo puede añadir máquinas cuando sube el tráfico. La **eficiencia computacional** y el **costo** se traducen en qué tipo de instancia necesitas (solo CPU o con GPU), cuántas y durante cuánto tiempo las pagas. Profundizaremos en varios tipos de algoritmos, con pautas sobre cuándo usar cada uno y cómo ajustarlos a tus requisitos particulares.

Siguiendo el ciclo de vida de ML, una vez seleccionado el algoritmo comienza el entrenamiento. Aquí cubriremos los pasos básicos para alimentar datos al algoritmo. En SageMaker, «alimentar datos al algoritmo» significa, en concreto, dejar los datos en un bucket de S3 (o en un sistema de archivos como EFS o FSx for Lustre) e indicarle al trabajo de entrenamiento dónde están; se verá con detalle en la sección de algoritmos integrados. En el próximo capítulo aprenderás a ajustar los hiperparámetros y a refinar iterativamente el modelo para mejorar su rendimiento. También hablaremos de técnicas para evitar trampas comunes como el sobreajuste (_overfitting_) y el subajuste (_underfitting_), de modo que tu modelo generalice bien a datos nuevos, no vistos.

Al final de este capítulo tendrás una comprensión integral de la fase de selección de modelos, con el conocimiento para elegir el algoritmo o el servicio de IA más apropiado para tu problema de ML y la confianza para implementar tus decisiones eficazmente.

## Comprender los servicios de IA de AWS (_Understanding AWS AI Services_)

Los servicios de IA de AWS son modelos de ML potentes, preconstruidos y preentrenados que ayudan a desarrolladores e ingenieros de ML a integrar la IA en sus aplicaciones de forma fluida, sin requerir un conocimiento profundo de ML. Estos servicios cubren una amplia gama de capacidades de IA, como visión por computadora, procesamiento de lenguaje natural (NLP), reconocimiento de voz, IA generativa y analítica predictiva.

Que sean «preentrenados» tiene una consecuencia de infraestructura importante: no hay trabajo de entrenamiento que lanzar ni servidor que administrar. AWS ya tiene el modelo desplegado detrás de una **API**, un punto de acceso en internet (un _endpoint_ HTTPS regional, como `rekognition.us-east-1.amazonaws.com`) al que tu aplicación envía peticiones y del que recibe respuestas en JSON. Cada petición va firmada con credenciales de **IAM** (_Identity and Access Management_), el servicio de AWS donde se define qué identidad puede hacer qué acción sobre qué recurso; por ejemplo, una política que permita `rekognition:DetectLabels` pero no `rekognition:IndexFaces`. En la práctica no se construyen las peticiones a mano: se usa un **SDK** (_software development kit_), una biblioteca que firma y envía las llamadas por ti. En Python es **boto3**, y llamar a Rekognition se reduce a `boto3.client("rekognition").detect_labels(...)`.

Por ejemplo, Amazon Rekognition permite analizar imágenes y videos para identificar objetos, personas, texto, escenas y actividades. Por su parte, Amazon Comprehend realiza tareas de NLP como el análisis de sentimiento y el reconocimiento de entidades, lo que permite a las empresas obtener información a partir de datos de texto no estructurados.

Una de las ventajas clave de los servicios de IA de AWS es su facilidad de uso y su escalabilidad. Vienen con APIs totalmente administradas (_fully managed_), que los desarrolladores pueden llamar desde sus aplicaciones para aprovechar capacidades sofisticadas de ML con un esfuerzo mínimo. **Fully managed** significa que AWS opera toda la infraestructura que hay detrás: los servidores, las GPU, los parches de seguridad, la alta disponibilidad y el escalado. Tú no eliges tipos de instancia ni pagas por máquinas encendidas, sino por uso (por imagen analizada, por carácter traducido, por minuto de audio transcrito). Estos servicios también están diseñados para manejar el procesamiento de datos a gran escala, asegurando que las aplicaciones puedan cumplir requisitos de alto rendimiento. Esa escala tiene dos matices prácticos que el libro no menciona. Primero, cada cuenta tiene **cuotas** (_service quotas_), límites como el número máximo de transacciones por segundo (TPS) por API y región; muchas se pueden ampliar con una solicitud en la consola de Service Quotas, pero hay que planificarlo antes de un lanzamiento. Segundo, para volúmenes grandes casi todos estos servicios ofrecen, además de la API **síncrona** (envías la petición y esperas la respuesta en la misma conexión, en milisegundos o segundos), una API **asíncrona** o de trabajos por lotes: le indicas una ubicación en S3 con miles de archivos, el servicio los procesa en segundo plano, deja los resultados en otra ubicación de S3 y te avisa al terminar.

Además, los servicios de IA de AWS se integran de forma transparente con otras ofertas de AWS, lo que permite construir flujos de trabajo de ML de extremo a extremo, desde la recolección y el procesamiento de datos hasta el entrenamiento y el despliegue de modelos. Este enfoque integral ayuda a agilizar el desarrollo y acelera la adopción de la IA en diversas industrias. En concreto, «integrarse» significa tres cosas. Los permisos se controlan con IAM, igual que en cualquier otro servicio. Los datos de entrada y salida viven normalmente en **Amazon S3**, el almacenamiento de objetos de AWS, donde cada archivo se guarda como un objeto dentro de un contenedor llamado _bucket_. Y los servicios se encadenan por eventos: por ejemplo, subir un PDF a un bucket dispara una función de **AWS Lambda** (código que AWS ejecuta bajo demanda, sin servidor que administrar, y que cobra por milisegundo de ejecución), esa función llama a Textract y el resultado termina en S3, listo para consultarse con Amazon Athena.

> [!warning] Uso de tus datos por los servicios de IA
> El libro no lo menciona, pero es una pregunta que hará cualquier área legal. Algunos servicios de IA de AWS (entre ellos Comprehend, Lex, Polly, Rekognition, Textract, Transcribe y Translate) pueden usar el contenido que procesan para mejorar el servicio, salvo que la organización lo desactive con una **política de exclusión de servicios de IA** (_AI services opt-out policy_) de AWS Organizations, que aplica a todas las cuentas de la organización. Amazon Bedrock, en cambio, no usa las peticiones ni las respuestas para entrenar modelos. Revisa la lista vigente de servicios cubiertos en la documentación de AWS Organizations.

Para el examen, necesitas saber qué servicios de IA de AWS resuelven distintos casos de uso de IA y ML, como el análisis de texto, el reconocimiento de imágenes y video, la síntesis y el reconocimiento de voz, la traducción, las recomendaciones personalizadas, el procesamiento de documentos y la IA generativa. Esto incluye entender servicios como Amazon Comprehend para el análisis de texto, Amazon Rekognition para el reconocimiento de imágenes y video, Amazon Polly para la síntesis de voz, Amazon Transcribe para el reconocimiento de voz, Amazon Translate para la traducción, Amazon Personalize para las recomendaciones, Amazon Textract para el procesamiento de documentos y Amazon Bedrock para la IA generativa. Conocer los casos de uso y las aplicaciones específicas de cada servicio es clave para aprovechar eficazmente las capacidades de IA de AWS en distintos escenarios.

### Visión (_Vision_)

AWS ofrece herramientas potentes para el análisis de contenido visual mediante servicios como Amazon Rekognition y Amazon Textract. Amazon Rekognition ayuda a analizar imágenes y videos para identificar objetos, rostros y texto, con capacidades que potencian aplicaciones como la seguridad y el análisis de medios. Amazon Textract destaca en la extracción de texto y datos de documentos escaneados, lo que permite automatizar el procesamiento de documentos. Juntos, estos servicios permiten a las empresas obtener información valiosa de sus datos visuales de forma eficiente y eficaz.

#### Amazon Rekognition

Amazon Rekognition es un servicio versátil de análisis de imágenes y video impulsado por algoritmos de deep learning, diseñado para identificar una amplia gama de objetos, personas, texto, escenas y actividades dentro de medios visuales. Una de sus funciones más destacadas es el análisis facial, que incluye la detección, la comparación y el reconocimiento de rostros. Esto permite a los desarrolladores crear aplicaciones capaces de reconocer rostros individuales en tiempo real o en medios almacenados. Además, Amazon Rekognition puede detectar texto dentro de imágenes y videos, lo que permite un análisis y una extracción completos de la información textual.

La diferencia entre «en tiempo real» y «en medios almacenados» corresponde a tres modos de uso con infraestructura distinta. Las **imágenes** se analizan con llamadas síncronas: envías la imagen (en bytes o como referencia a un objeto de S3) y recibes el resultado en la misma respuesta. Los **videos almacenados** en S3 se analizan de forma asíncrona: inicias un trabajo (p. ej., `StartLabelDetection`), Rekognition procesa el video en segundo plano y avisa al terminar publicando un mensaje en **Amazon SNS** (_Simple Notification Service_, un servicio de mensajería que reenvía notificaciones a colas, funciones Lambda o correos). Los **videos en vivo**, como los de una cámara, no se envían a Rekognition directamente: la cámara transmite a **Amazon Kinesis Video Streams**, un servicio que recibe y almacena flujos de video, y un **procesador de flujo** (_stream processor_) de Rekognition lee ese flujo y emite los resultados. El reconocimiento facial, en particular, se apoya en **colecciones** (_face collections_): contenedores del lado de AWS donde indexas los rostros conocidos (`IndexFaces`) para después buscar coincidencias (`SearchFacesByImage`). Rekognition no guarda la foto en la colección, sino un vector numérico de rasgos faciales por cada rostro.

El servicio es fácil de usar: ofrece una API sencilla que permite a los desarrolladores añadir funciones de análisis de imágenes y video a sus aplicaciones sin necesitar un conocimiento extenso de ML. Su escalabilidad le permite manejar grandes volúmenes de datos de forma eficiente, lo que lo hace adecuado tanto para proyectos pequeños como para soluciones empresariales. Al aprovechar Amazon Rekognition, los desarrolladores pueden crear aplicaciones sofisticadas que analicen contenido visual para extraer información valiosa, automatizar procesos y mejorar la experiencia de usuario.

##### Casos de uso (_Use Cases_)

Amazon Rekognition se usa en diversas aplicaciones críticas de distintas industrias. Ayuda a detectar contenido inapropiado en imágenes y videos, asegurando que los medios de las plataformas sean seguros y cumplan las normas. También se usa para verificar identidades en línea, lo que mejora la seguridad y la validación de usuarios en los servicios digitales. Las empresas de medios pueden usarlo para agilizar su análisis, etiquetando y categorizando automáticamente el contenido, lo que facilita gestionar y recuperar activos específicos. Además, Amazon Rekognition puede enviar alertas inteligentes a hogares conectados, reconociendo rostros u objetos específicos y avisando a los propietarios de actividades inusuales, lo que mejora los sistemas de seguridad y automatización del hogar. Estos casos de uso muestran la versatilidad de Amazon Rekognition y su papel significativo en la transformación del uso de los datos visuales en diversos sectores.

La verificación de identidad en línea combina dos funciones que conviene conocer por nombre. `CompareFaces` compara el rostro de una selfie con el de la foto de una identificación. **Face Liveness** comprueba que frente a la cámara hay una persona real y presente, y no una foto impresa, una pantalla o una máscara; se integra en la aplicación web o móvil con un componente de interfaz de **AWS Amplify**, el conjunto de bibliotecas de AWS para aplicaciones de cliente.

> [!warning] Estado de dos casos de uso (verificado el 25-09-2026)
> Las alertas para hogares conectados que describe el libro corresponden a la función **Rekognition Streaming Video Events**, y la moderación de contenido tiene una variante de **moderación de imágenes por lotes**. Ambas funciones pasaron a mantenimiento y no admiten clientes nuevos desde el 30-04-2026. La moderación de imágenes individuales con `DetectModerationLabels` no está afectada.

#### Amazon Textract

Amazon Textract es un servicio de IA sofisticado, diseñado para extraer texto, formularios, tablas y otros datos de documentos escaneados. A diferencia de las tecnologías tradicionales de reconocimiento óptico de caracteres (OCR), Amazon Textract usa ML para entender la disposición y la estructura de los documentos, capturando las relaciones semánticas entre sus distintos elementos. Esta capacidad avanzada le permite interpretar con exactitud información compleja, lo que lo hace especialmente útil para procesar grandes volúmenes de documentos de forma rápida y eficaz, transformando datos no estructurados en formatos estructurados que se pueden analizar y usar con eficiencia.

Una de las funciones clave de Amazon Textract es su capacidad de reconocer y extraer datos de tablas y formularios dentro de los documentos, entendiendo el contexto y las relaciones entre los distintos datos. Esto va más allá de simplemente leer texto: Amazon Textract puede distinguir entre encabezados, filas y columnas de una tabla, asegurando que los datos extraídos mantengan su estructura y su significado originales. Esta comprensión semántica es crucial para aplicaciones que requieren una extracción e interpretación precisas, como los informes financieros, la documentación legal y los expedientes médicos, donde el contexto de los datos es tan importante como los datos mismos.

En la API esto se traduce en operaciones distintas según el documento. `DetectDocumentText` solo extrae líneas y palabras. `AnalyzeDocument` añade, según las opciones que actives, pares clave-valor de formularios (`FORMS`), tablas (`TABLES`), respuestas a preguntas en lenguaje natural sobre el documento (`QUERIES`), firmas y estructura de página (`LAYOUT`). Hay además APIs especializadas: `AnalyzeExpense` para facturas y recibos, y `AnalyzeID` para documentos de identidad. Como en Rekognition, el modo depende del tamaño: las imágenes y los documentos de una página se procesan de forma síncrona, mientras que los PDF o TIFF de varias páginas se dejan en S3 y se procesan con las variantes asíncronas (`StartDocumentAnalysis`), que notifican el final por SNS.

Amazon Textract también se integra de forma transparente con otros servicios de AWS, lo que permite construir canalizaciones (_pipelines_) completas de procesamiento de datos. Por ejemplo, los datos extraídos pueden almacenarse en Amazon S3, analizarse con Amazon Athena o alimentar a Amazon SageMaker para aplicaciones de ML. **Amazon Athena** es un motor SQL _serverless_: consulta directamente archivos guardados en S3, sin cargarlos en una base de datos, y cobra por los datos que escanea cada consulta. **Serverless** significa que no aprovisionas ni administras servidores; el servicio asigna cómputo cuando lo necesitas y cobra por uso. Esta flexibilidad permite a las organizaciones usar Amazon Textract como parte de una estrategia de datos más amplia, mejorando su capacidad de obtener información y tomar decisiones a partir de sus documentos. Al automatizar la extracción y el procesamiento de la información y preservar las relaciones semánticas, Amazon Textract ayuda a reducir el esfuerzo manual, minimizar los errores y aumentar la eficiencia operativa.

##### Casos de uso (_Use Cases_)

Amazon Textract se usa ampliamente en diversas industrias para agilizar el procesamiento de documentos y la extracción de datos. En el sector financiero, ayuda a automatizar la extracción de datos de facturas, recibos y documentos fiscales, lo que permite informes financieros más rápidos y exactos. En salud, se usa para digitalizar expedientes de pacientes, facilitando la recuperación y el análisis de la información médica. Los despachos jurídicos pueden aprovecharlo para procesar grandes volúmenes de contratos y documentos legales, asegurando que la información crítica se capture con exactitud y quede accesible. Además, en el sector público, Amazon Textract puede ayudar a gestionar y analizar formularios y solicitudes gubernamentales, mejorando la eficiencia de los procesos administrativos. Estos casos de uso destacan la adaptabilidad de Amazon Textract y su impacto significativo en la automatización de la extracción y el procesamiento de datos en industrias muy diversas.

### Voz (_Speech_)

Los servicios de IA de AWS ofrecen capacidades completas de voz, diseñadas para transformar la forma en que las aplicaciones interactúan con los usuarios mediante la voz. Estos servicios proporcionan herramientas tanto para generar voz a partir de texto como para convertir el lenguaje hablado en texto, lo que permite experiencias de usuario más naturales y accesibles. Para los casos de uso de voz, los dos servicios principales son Amazon Polly, que convierte texto en voz de sonido natural, y Amazon Transcribe, que convierte el lenguaje hablado en texto preciso.

#### Amazon Polly

Amazon Polly es un servicio de conversión de texto a voz (_text-to-speech_) que usa tecnologías avanzadas de deep learning para convertir texto escrito en voz de sonido natural. Ofrece una amplia variedad de voces realistas en múltiples idiomas y admite distintos acentos, lo que da a los desarrolladores flexibilidad para crear experiencias atractivas y localizadas para sus usuarios. Ya sea para sistemas de respuesta de voz interactiva (IVR), aplicaciones de lectura de noticias o audiolibros, Amazon Polly mejora el compromiso de los usuarios al hacer las aplicaciones más accesibles e interactivas. Un **IVR** (_interactive voice response_) es el sistema telefónico automático que atiende una llamada con mensajes grabados o sintetizados («para saldos, marque 1») y enruta al cliente según lo que marca o dice. Polly aporta la voz de esos mensajes, lo que permite cambiar un texto sin volver a grabar a un locutor.

Amazon Polly destaca por ofrecer síntesis de voz rápida y en tiempo real, con una latencia mínima para aplicaciones que requieren respuestas inmediatas. La **latencia** es el tiempo que transcurre entre que envías la petición y empiezas a recibir la respuesta; en una llamada telefónica, unos cientos de milisegundos de silencio ya se perciben como un fallo. Polly la reduce devolviendo el audio en flujo continuo (_streaming_) mientras lo genera, con la operación síncrona `SynthesizeSpeech`. Para textos largos, como el capítulo de un audiolibro, existe la operación asíncrona `StartSpeechSynthesisTask`, que deja el archivo de audio en un bucket de S3. Polly ofrece además varios «motores» de voz (estándar, neuronal, de formato largo y generativo) que difieren en naturalidad, precio y disponibilidad por idioma y región. Amazon Polly también admite el lenguaje de marcado de síntesis de voz (**SSML**, _Speech Synthesis Markup Language_), que permite a los desarrolladores controlar aspectos como el tono, la velocidad y la pronunciación de la voz para lograr una experiencia auditiva más ajustada. SSML es un estándar del W3C basado en XML, con etiquetas parecidas a las de HTML: en lugar de texto plano envías, por ejemplo, `<speak>Su saldo es <prosody rate="slow">1 250 pesos</prosody><break time="500ms"/></speak>`. Para pronunciaciones recurrentes (nombres de marca, siglas, fármacos) Polly admite además **léxicos** (_lexicons_), diccionarios de pronunciación que se cargan una vez en la cuenta y se aplican a todas las peticiones. Con Amazon Polly puedes crear de forma sencilla y eficiente interacciones de voz dinámicas y naturales.

##### Casos de uso (_Use Cases_)

Amazon Polly es un servicio versátil de texto a voz con una amplia variedad de casos de uso. Es ideal para crear interacciones de voz atractivas en sistemas IVR, asegurando que los clientes reciban respuestas claras y naturales. Polly también es perfecto para generar audiolibros, ofreciendo a los lectores una experiencia inmersiva con una narración realista. Además, mejora la accesibilidad al convertir contenido textual en voz para usuarios con discapacidad visual, haciendo la información más accesible. En las plataformas de e-learning, Polly puede dar vida al contenido educativo con una narración dinámica, lo que mejora la comprensión y la retención. En general, Amazon Polly es una herramienta potente para cualquier aplicación que se beneficie de una voz de alta calidad y sonido natural.

#### Amazon Transcribe

Amazon Transcribe es un potente servicio de reconocimiento automático de voz (ASR, _automatic speech recognition_) que convierte el lenguaje hablado en texto escrito. Está diseñado para producir transcripciones de alta exactitud de diversos formatos de audio y video, lo que lo convierte en una herramienta esencial para generar texto consultable a partir de grabaciones. Ya sea para crear subtítulos de contenido de video, generar transcripciones de pódcasts y entrevistas o habilitar subtítulos en tiempo real para eventos en vivo, Amazon Transcribe mejora la accesibilidad y la capacidad de búsqueda. También admite múltiples idiomas y dialectos, lo que asegura una amplia aplicabilidad en distintas regiones y contextos. Con funciones como la identificación de hablantes, la restitución de la puntuación y los vocabularios personalizados, Amazon Transcribe ayuda a los usuarios a obtener transcripciones exactas y completas para aplicaciones diversas.

Igual que en los servicios anteriores, hay dos formas de usarlo con infraestructura distinta. En la **transcripción por lotes**, el archivo de audio o video ya está en S3; inicias un trabajo (`StartTranscriptionJob`) y Transcribe deja el resultado como un archivo JSON en S3, con cada palabra, su marca de tiempo, su nivel de confianza y el hablante. En la **transcripción en streaming**, tu aplicación abre una conexión persistente con el servicio (sobre HTTP/2 o **WebSocket**, un protocolo que mantiene abierta una conexión bidireccional para enviar audio y recibir texto a la vez) y recibe el texto parcial mientras la persona habla; así funcionan los subtítulos en vivo. Los **vocabularios personalizados** son listas de términos que cargas en la cuenta (nombres de productos, apellidos, jerga del sector) para que el servicio los reconozca. Transcribe tiene además variantes especializadas: **Call Analytics**, para llamadas de centros de contacto, y **Transcribe Medical**, para dictado clínico.

##### Casos de uso (_Use Cases_)

Amazon Transcribe atiende una amplia gama de aplicaciones. Es perfecto para crear subtítulos exactos para contenido de video, lo que hace los medios más accesibles y atractivos. Los creadores de pódcasts y los entrevistadores pueden usarlo para generar transcripciones, lo que facilita el archivo del contenido y su búsqueda. En entornos en vivo, Transcribe puede proporcionar subtítulos en tiempo real, mejorando la accesibilidad para audiencias con discapacidad auditiva. También es valioso para los centros de atención telefónica, donde puede transcribir conversaciones para mejorar el análisis y el cumplimiento normativo. En ese contexto el **cumplimiento normativo** (_compliance_) suele exigir dos cosas que Transcribe resuelve con configuración: redactar (ocultar) datos personales como números de tarjeta en la transcripción, y separar los canales de audio de agente y cliente para saber quién dijo qué. Al convertir audio en texto de forma eficiente, Amazon Transcribe mejora la usabilidad y la accesibilidad del contenido hablado en industrias muy diversas.

### Lenguaje (_Language_)

Los servicios de IA de lenguaje de AWS están diseñados para mejorar y simplificar una amplia gama de tareas de NLP, con herramientas para entender, generar y traducir lenguaje humano. Estos servicios permiten a los desarrolladores crear aplicaciones que interactúen con los usuarios de forma más intuitiva y natural, automaticen el análisis de texto y admitan la comunicación multilingüe. En particular, Amazon Translate ofrece traducción en tiempo real y Amazon Comprehend proporciona información mediante el análisis de texto, incluida la detección de sentimiento y el reconocimiento de entidades.

#### Amazon Translate

Amazon Translate es un servicio de traducción automática neuronal que ofrece traducciones rápidas, de alta calidad y a precio asequible. Admite decenas de idiomas, lo que permite a empresas y desarrolladores traducir grandes volúmenes de texto con eficiencia e integrar la traducción directamente en sus aplicaciones. Amazon Translate puede manejar traducción en tiempo real para aplicaciones como sitios web y apps móviles, lo que facilita que los usuarios de todo el mundo interactúen con el contenido en su idioma nativo. El servicio también admite traducción por lotes, útil para traducir rápidamente documentos o datasets grandes.

Las dos modalidades funcionan como en Transcribe. La traducción en tiempo real es una llamada síncrona (`TranslateText` para texto, `TranslateDocument` para un documento pequeño) cuya respuesta llega en la misma conexión. La traducción por lotes (`StartTextTranslationJob`) lee una carpeta de S3 y escribe las traducciones en otra. Para esto último, Translate necesita un **rol de acceso a datos**: un rol de IAM que tú creas y que el servicio «asume» temporalmente para leer y escribir en tus buckets. Un **rol de IAM** es una identidad con permisos que no pertenece a una persona, sino que la adoptan temporalmente usuarios o servicios; así Translate obtiene acceso a tus datos sin que tengas que entregarle credenciales permanentes.

Amazon Translate está diseñado para preservar el contexto y el significado del texto original, asegurando que las traducciones no solo sean semánticamente exactas, sino que también suenen naturales. Usa modelos avanzados de deep learning entrenados con una vasta variedad de datos multilingües, lo que le permite mejorar continuamente con el tiempo. Esa mejora continua la hace AWS sobre sus propios modelos; si necesitas que ciertos términos se traduzcan siempre igual (nombres de producto, términos legales), la herramienta de configuración es la **terminología personalizada** (_custom terminology_), un glosario de pares origen-destino que cargas en la cuenta. Amazon Translate es altamente escalable y rentable, lo que lo hace accesible para empresas de todos los tamaños. También se integra de forma transparente con otros servicios de AWS, lo que permite construir soluciones completas que combinan la traducción con otras herramientas de IA y de la nube.

##### Casos de uso (_Use Cases_)

Amazon Translate atiende una amplia gama de casos de uso, lo que lo convierte en una herramienta invaluable para empresas y desarrolladores que necesitan comunicación multilingüe. Se usa ampliamente para traducir el contenido de sitios web, permitiendo a los usuarios de todo el mundo acceder a la información e interactuar con ella en su idioma nativo. Las plataformas de comercio electrónico lo aprovechan para ofrecer descripciones de productos en varios idiomas, mejorando la experiencia de usuario y ampliando su alcance de mercado. En la atención al cliente, ayuda a traducir las comunicaciones por chat y correo electrónico, asegurando un servicio fluido en distintas regiones. Amazon Translate también cumple un papel crucial en la traducción de documentos, informes y manuales de usuario, facilitando la colaboración global y el intercambio de información. Además, admite la traducción en tiempo real para aplicaciones como redes sociales y mensajería instantánea, fomentando una comunicación instantánea y eficaz en todo el mundo.

#### Amazon Comprehend

Amazon Comprehend es un servicio de NLP que usa ML para descubrir información y relaciones dentro de un texto. Permite a las empresas entender el sentimiento detrás de las reseñas de los clientes, extraer frases clave, identificar entidades con nombre e incluso detectar el idioma del texto de entrada. Amazon Comprehend puede analizar grandes volúmenes de texto con rapidez y exactitud, lo que lo convierte en una excelente opción para procesar comentarios de clientes, publicaciones en redes sociales y otros datos no estructurados. El servicio también puede organizar documentos identificando temas clave y categorizando el contenido, lo que ayuda a las organizaciones a gestionar y entender mejor sus datos.

Cada una de esas funciones es una operación de la API: `DetectSentiment`, `DetectKeyPhrases`, `DetectEntities` y `DetectDominantLanguage` responden de forma síncrona a un documento; sus variantes `BatchDetect*` aceptan un grupo pequeño de documentos por llamada, y los trabajos asíncronos (`Start*Job`) procesan una carpeta completa de S3 y dejan los resultados en otra.

Amazon Comprehend es un servicio altamente configurable que permite crear modelos personalizados adaptados a necesidades de negocio específicas. Estos modelos personalizados permiten un reconocimiento de entidades y una clasificación más precisos, basados en la terminología y el contexto propios de cada industria. En términos de infraestructura, un modelo personalizado de Comprehend (_custom classifier_ o _custom entity recognizer_) se entrena a partir de documentos de ejemplo que dejas en S3, sin que administres instancias. Para usarlo en tiempo real hay que crear un **endpoint** de Comprehend, que se aprovisiona con cierta capacidad (medida en «unidades de inferencia») y **se cobra por cada segundo que está activo, lo uses o no**; para análisis ocasionales sale más barato un trabajo asíncrono. Igual que otros servicios de IA de AWS, Amazon Comprehend se integra de forma transparente con el ecosistema de AWS, lo que permite construir canalizaciones completas de análisis de texto y automatizar flujos de trabajo. Ya sea para mejorar la atención al cliente analizando tickets de soporte o para mejorar sistemas de recomendación de contenido, Amazon Comprehend proporciona herramientas potentes para convertir texto en información accionable.

Para las organizaciones que requieren capacidades más avanzadas, Amazon SageMaker ofrece de serie algoritmos sofisticados de modelado de temas. Estos algoritmos se tratan en detalle más adelante en el capítulo.

> [!warning] Nota de precisión: Comprehend también hacía modelado de temas
> Comprehend tiene su propio modelado de temas (trabajos asíncronos `StartTopicsDetectionJob`), pero esa función, junto con la detección de eventos y la clasificación de seguridad de prompts, no admite clientes nuevos desde el 30-04-2026. Las cuentas que la usaron en los últimos 12 meses la conservan. Para cuentas nuevas, las alternativas en AWS son los algoritmos LDA y NTM de SageMaker (más adelante en este capítulo) o un modelo de Bedrock; el resto de Comprehend no está afectado.

##### Casos de uso (_Use Cases_)

Amazon Comprehend puede aplicarse a una amplia gama de casos de uso en distintas industrias. Las empresas suelen usarlo para el análisis de sentimiento, que les ayuda a entender los comentarios de los clientes y el sentimiento en redes sociales para mejorar productos y servicios. En la industria de la salud, Amazon Comprehend puede analizar expedientes médicos para extraer información valiosa e identificar entidades clave como medicamentos, padecimientos y tratamientos. También se usa en los sistemas de gestión de contenido para etiquetar y organizar automáticamente grandes volúmenes de documentos, facilitando su búsqueda y gestión. En la atención al cliente, Amazon Comprehend puede analizar tickets de soporte para identificar problemas comunes y mejorar los tiempos de respuesta. Además, ayuda en la detección de fraude analizando patrones de comunicación e identificando actividades sospechosas. En general, Amazon Comprehend permite a las organizaciones obtener información valiosa de los datos de texto, mejorando la toma de decisiones y la eficiencia operativa.

> [!warning] Nota de precisión: textos médicos
> La extracción de medicamentos, padecimientos y tratamientos que menciona el libro la hace **Amazon Comprehend Medical**, un servicio aparte con su propia API (`DetectEntitiesV2`, entre otras) y entrenado específicamente con texto clínico. El Comprehend general no reconoce esas entidades médicas de serie.

### Chatbot

AWS ofrece capacidades potentes de chatbot mediante servicios que permiten a las empresas crear interfaces conversacionales atractivas, interactivas e inteligentes. Estos chatbots pueden usarse en diversas aplicaciones, desde la atención al cliente hasta aplicaciones interactivas y mesas de ayuda internas. AWS proporciona herramientas como Amazon Lex, que usa NLP avanzado para entender la entrada del usuario y gestionar conversaciones complejas.

#### Amazon Lex

Amazon Lex es un servicio potente diseñado para integrar interfaces conversacionales de voz y texto en cualquier aplicación. Aprovechando las mismas tecnologías de deep learning que impulsan a Amazon Alexa, Amazon Lex ofrece capacidades avanzadas de comprensión del lenguaje natural. Esto permite a los desarrolladores crear chatbots sofisticados e interactivos que entienden la entrada del usuario y responden con un alto grado de exactitud. Al integrarse con otros servicios de AWS como AWS Lambda, Amazon Lex permite ejecutar sin fricciones la lógica de _backend_, lo que hace posible construir aplicaciones conversacionales completas de extremo a extremo. El _backend_ es la parte de la aplicación que no ve el usuario: consultar el estado de un pedido en la base de datos, agendar una cita en el sistema del hospital. Lex entiende qué quiere el usuario, pero no sabe consultar tus sistemas; para eso invoca una función Lambda escrita por ti (en la terminología de Lex, la función de **_fulfillment_** o cumplimiento), que hace la consulta y le devuelve el texto de la respuesta.

Amazon Lex puede manejar conversaciones de varios turnos (_multiturn_), lo que permite a los chatbots mantener el contexto y gestionar diálogos complejos. Esto es particularmente útil en las aplicaciones de atención al cliente, donde un bot puede necesitar recopilar información a lo largo de varias interacciones para resolver un problema. Amazon Lex también admite integraciones nativas con **Amazon Connect**, el servicio de centro de contacto de AWS impulsado por IA, lo que permite a las empresas crear agentes automatizados de centro de llamadas que atiendan a los clientes las 24 horas, reduciendo la carga de los agentes humanos y mejorando los tiempos de respuesta. Un **centro de contacto** (_contact center_) es la infraestructura con la que una empresa atiende llamadas y chats de clientes: números telefónicos, colas, reglas de enrutamiento hacia agentes, grabación y métricas. Amazon Connect ofrece todo eso como servicio en la nube, sin centralitas físicas, y un bot de Lex puede atender el primer tramo de la llamada antes de pasarla a una persona. Además, Amazon Lex puede usarse para crear chatbots para diversas plataformas, como aplicaciones web, apps móviles y canales de redes sociales, ofreciendo experiencias de usuario consistentes y escalables en los distintos puntos de contacto.

Desde el punto de vista de ingeniería, Amazon Lex ofrece una consola y un entorno de desarrollo fáciles de usar, donde los desarrolladores pueden definir el modelo de interacción del bot, incluidos las intenciones (_intents_), los espacios (_slots_) y las respuestas, sin necesitar experiencia extensa en ML o NLP. Estos tres términos son la configuración de un bot de Lex. Una **intención** es una acción que el usuario quiere realizar, como `AgendarCita`, y se define con varias frases de ejemplo (_utterances_) como «quiero una cita» o «necesito ver al médico». Un **slot** es un dato que el bot debe obtener para completar la intención, como la fecha o la especialidad, con un tipo que restringe sus valores válidos. La **respuesta** es lo que el bot contesta, fija o generada por la función Lambda. Además, Amazon Lex ofrece herramientas integradas para probar y monitorear el desempeño del chatbot, que ayudan a los desarrolladores a iterar y mejorar sus bots con el tiempo. Entre ellas están una ventana de prueba en la consola y los **registros de conversación**, que guardan el texto de cada intercambio en **Amazon CloudWatch Logs** (el servicio de AWS que centraliza los registros de las aplicaciones) y el audio en S3. Con sus capacidades robustas y su integración fluida con otros servicios de AWS, Amazon Lex es una solución integral para crear interfaces conversacionales inteligentes que mejoran el compromiso de los usuarios y agilizan los procesos de negocio.

##### Casos de uso (_Use Cases_)

Amazon Lex se usa principalmente para mejorar la experiencia de atención al cliente. Los clientes aprovechan Amazon Lex creando chatbots sofisticados que atienden consultas, proporcionan información y resuelven problemas, minimizando la necesidad de agentes humanos y reduciendo los tiempos de respuesta. En el comercio electrónico, los bots impulsados por Lex pueden ayudar a los clientes con la búsqueda de productos, el seguimiento de pedidos y las recomendaciones personalizadas, mejorando la experiencia de compra. Las organizaciones de salud usan Amazon Lex para automatizar la programación de citas, proporcionar información médica y realizar la preevaluación de pacientes. Por último, Amazon Lex se emplea en las operaciones internas de las empresas para crear asistentes virtuales que ayudan a los empleados con tareas como el soporte de TI, las consultas de recursos humanos y la automatización de flujos de trabajo. En general, Amazon Lex permite a las empresas crear interfaces conversacionales inteligentes que mejoran el compromiso de los usuarios y agilizan los procesos administrativos.

### Recomendación (_Recommendation_)

AWS proporciona capacidades de recomendación mediante su conjunto de servicios de IA. Estas capacidades buscan mejorar la experiencia de usuario ofreciendo contenido personalizado.

En el centro de estas capacidades está Amazon Personalize, que permite a los desarrolladores integrar fácilmente en sus aplicaciones recomendaciones personalizadas en tiempo real. Amazon Personalize utiliza algoritmos de ML sofisticados para analizar los datos de los usuarios, como el historial de navegación, el comportamiento de compra y las preferencias, y generar recomendaciones muy relevantes.

#### Amazon Personalize

Amazon Personalize funciona aprovechando algoritmos avanzados de ML para analizar los datos de los usuarios y generar recomendaciones altamente personalizadas. El proceso comienza con la recolección de datos de interacción de los usuarios con los ítems, como clics, visualizaciones y compras, además de información contextual adicional, como los metadatos de los ítems (p. ej., género, precio) y los datos demográficos de los usuarios (p. ej., edad, género). Estos datos alimentan después a los modelos de ML, que se entrenan para identificar patrones y relaciones entre usuarios e ítems. Los modelos pueden actualizarse continuamente con datos de interacción en tiempo real, asegurando que las recomendaciones sigan siendo exactas y relevantes a medida que evolucionan las preferencias de los usuarios.

Ese proceso corresponde a una secuencia de recursos que se crean en la cuenta, y conviene conocerlos por nombre porque aparecen en las preguntas de examen. Todo vive dentro de un **grupo de datasets** (_dataset group_). Los datos históricos se cargan como **datasets** (interacciones, usuarios e ítems), cada uno con un esquema declarado, mediante un trabajo de importación que lee archivos CSV desde S3 usando un rol de IAM. Los datos en tiempo real llegan por otra vía: tu aplicación envía cada clic o compra en el momento con la API `PutEvents`, a través de un **rastreador de eventos** (_event tracker_). El entrenamiento produce una **versión de solución** (_solution version_), que se cobra por horas de entrenamiento. Para servir recomendaciones en tiempo real hay que desplegarla en una **campaña** (_campaign_), un endpoint con una capacidad mínima aprovisionada en transacciones por segundo que **se cobra por hora mientras exista**; para generar recomendaciones de muchos usuarios de una vez (por ejemplo, para un correo masivo) se usa en cambio un trabajo de inferencia por lotes, que lee y escribe en S3 y no deja nada encendido.

Una característica clave y exclusiva de Amazon Personalize es el uso de técnicas de IA generativa para mejorar el proceso de recomendación. Los modelos de IA generativa pueden simular nuevos puntos de datos a partir de los existentes, rellenando de forma eficaz los huecos del dataset y mejorando la exactitud de las predicciones. Esto es particularmente útil cuando se trabaja con datos dispersos o con usuarios e ítems nuevos que tienen un historial de interacción limitado. Al generar puntos de datos sintéticos, el sistema puede entender mejor las preferencias de los usuarios y ofrecer recomendaciones más precisas, incluso en escenarios donde los modelos tradicionales tendrían dificultades. Además, Amazon Personalize permite la personalización basada en tus datos únicos. Puedes elegir los algoritmos de ML más adecuados para tu caso de uso y las características de tus datos, y proporcionar metadatos contextuales sobre usuarios e ítems para obtener recomendaciones mejor informadas. En Personalize, esos algoritmos se llaman **recetas** (_recipes_): cada receta es un algoritmo preconfigurado por AWS para un tipo de recomendación (por ejemplo, `User-Personalization` para «recomendado para ti», `Similar-Items` para «productos similares» o `Personalized-Ranking` para reordenar una lista). Para dominios comunes, como el comercio electrónico o el video bajo demanda, existen además **recomendadores** (_recommenders_) ya configurados por caso de uso, en los que ni siquiera eliges la receta.

> [!warning] Nota de precisión: IA generativa en Personalize
> La documentación vigente de Amazon Personalize no describe la generación de datos sintéticos para rellenar huecos que menciona el libro. Lo que documenta como IA generativa es **Content Generator**, que añade títulos temáticos generados por un LLM a lotes de recomendaciones de ítems similares (p. ej., «Para empezar el día» en lugar de «Comprados juntos con frecuencia»), la opción de devolver metadatos de los ítems para incorporarlos a prompts, y código preconfigurado para LangChain. El caso de usuarios e ítems nuevos se atiende con las recetas de personalización y los metadatos de los ítems, no con datos sintéticos. Trata la afirmación del libro con cautela en el examen.

Además, Amazon Personalize emplea algoritmos sofisticados para manejar diversas tareas de recomendación, como el filtrado colaborativo, el filtrado basado en contenido y los enfoques híbridos. El filtrado colaborativo se apoya en el comportamiento colectivo de los usuarios para identificar usuarios similares y recomendar los ítems que les gustaron. El filtrado basado en contenido se centra en los atributos de los ítems para sugerir productos similares a aquellos por los que un usuario ha mostrado interés. Los modelos híbridos combinan ambos enfoques para aprovechar las fortalezas de cada uno, ofreciendo recomendaciones robustas y completas. Al usar estas técnicas avanzadas de IA y generativas, Amazon Personalize ofrece recomendaciones altamente personalizadas y eficaces que mejoran el compromiso y la satisfacción de los usuarios. Su flexibilidad y facilidad de integración aseguran que las empresas puedan adaptar el servicio a sus necesidades específicas y obtener resultados óptimos.

##### Casos de uso (_Use Cases_)

Amazon Personalize se usa ampliamente en diversas industrias para mejorar la experiencia de usuario mediante recomendaciones personalizadas. En el comercio electrónico, sugiere productos basándose en el historial de navegación y el comportamiento de compra de los usuarios, aumentando las tasas de conversión y la satisfacción de los clientes. Los servicios de streaming y las plataformas de noticias lo aprovechan para recomendar películas, series y artículos adaptados a los intereses de cada persona, manteniendo a los usuarios comprometidos. Las empresas también lo usan en campañas de marketing dirigidas, personalizando el contenido de los correos electrónicos para promocionar productos y ofertas relevantes, lo que mejora el compromiso. Además, los equipos de atención al cliente usan Amazon Personalize para recomendar artículos de soporte relevantes, ayudando a los usuarios a resolver sus problemas de forma más eficiente.

Al ofrecer recomendaciones muy relevantes y oportunas, Amazon Personalize permite a las empresas mejorar el compromiso de los usuarios y obtener mejores resultados en diversos sectores.

### IA generativa (_Generative AI_)

La IA generativa representa un avance de vanguardia en la inteligencia artificial: permite a los sistemas generar (de ahí viene lo de IA «generativa») contenido nuevo, como texto, imágenes o incluso experiencias multimedia completas, a partir de patrones aprendidos de los datos. Con Amazon Bedrock, AWS pone las capacidades de IA generativa al alcance de los desarrolladores, con una plataforma fluida y escalable para crear, ajustar y desplegar modelos de IA generativa. Amazon Bedrock da acceso a una variedad de potentes modelos fundacionales (FM, _foundation models_) de empresas líderes en IA, junto con las herramientas necesarias para personalizarlos e integrarlos en las aplicaciones sin esfuerzo. Al usar Amazon Bedrock, las empresas pueden aprovechar el potencial de la IA generativa para innovar y mejorar sus soluciones digitales, impulsando la creatividad y la eficiencia en diversos ámbitos.

#### Amazon Bedrock

Amazon Bedrock es un servicio totalmente administrado que simplifica el proceso de crear y escalar aplicaciones de IA generativa con FMs. Da acceso a una amplia gama de FMs de alto rendimiento de empresas líderes en IA como AI21 Labs, Anthropic, Cohere, Meta, Mistral AI y Stability AI, así como a los modelos propios de Amazon, incluidos los recién lanzados FMs Nova.

La lista de proveedores del libro es la de su fecha de edición; hoy el catálogo de Bedrock incluye además, entre otros, a DeepSeek, Google (Gemma), OpenAI, Qwen, NVIDIA, Writer y xAI. Lo relevante para la infraestructura es que todos esos modelos se consumen igual: AWS los ejecuta en su propia infraestructura, dentro de la región que eliges, y tú solo haces llamadas a la API. Ni el proveedor del modelo ni tú administran servidores, y según la documentación de Bedrock tus peticiones y respuestas no se usan para entrenar modelos ni se comparten con el proveedor.

Los FMs de propósito general son modelos versátiles que pueden manejar una variedad de tareas, como generación de texto, traducción, resumen y más. Estos modelos son adecuados para una amplia gama de aplicaciones en distintas industrias. Con una sola API, los desarrolladores pueden experimentar con distintos modelos, personalizarlos con sus propios datos mediante técnicas como el ajuste fino (_fine-tuning_) y la generación aumentada por recuperación (RAG, _retrieval augmented generation_), y desplegarlos de forma segura dentro de sus aplicaciones. En Bedrock, el ajuste fino se ejecuta como un **trabajo de personalización de modelo** (_model customization job_) que lee los datos de entrenamiento desde S3 y produce un modelo personalizado privado de tu cuenta; la RAG se implementa con **Knowledge Bases**, que conectan el modelo con tus documentos almacenados en S3 u otras fuentes a través de una base de datos vectorial. «Desplegar de forma segura» se apoya en piezas estándar de AWS: políticas de IAM que limitan qué identidades pueden invocar qué modelos, cifrado con **AWS KMS** (el servicio de gestión de claves de cifrado de AWS) y, si tu aplicación corre en una red privada, un endpoint de VPC para que el tráfico hacia Bedrock no pase por internet (se explica en el primer escenario al final del capítulo).

La arquitectura serverless de Amazon Bedrock elimina la necesidad de administrar infraestructura, lo que permite a los desarrolladores centrarse en innovar y aportar valor a sus usuarios. La **Converse API**, parte de Amazon Bedrock, ofrece una interfaz consistente y simplificada para que los desarrolladores interactúen con estos FMs, lo que facilita integrar capacidades de IA conversacional en las aplicaciones para ofrecer experiencias de usuario mejoradas e interacciones naturales. La diferencia con la otra forma de llamar a un modelo, `InvokeModel`, es de formato. Con `InvokeModel`, el cuerpo de la petición y el de la respuesta siguen el esquema propio de cada proveedor, así que cambiar de modelo obliga a reescribir el código que arma y lee esos JSON. Con `Converse`, la petición tiene siempre la misma forma (una lista de mensajes con roles `user` y `assistant`, más parámetros comunes como la temperatura o el máximo de tokens), y cambiar de modelo suele reducirse a cambiar el `modelId`.

> [!note]
> No todos los FMs son compatibles con la Converse API. Para ver la lista detallada de modelos y funciones compatibles, visita https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference-supported-models-features.html.

Una de las funciones más destacadas de Amazon Bedrock es su _marketplace_, que ofrece una amplia selección de FMs especializados para distintos casos de uso. Los FMs especializados están adaptados a tareas o dominios específicos, lo que les da un rendimiento más exacto y eficiente en aplicaciones concretas. Estos modelos pueden ser muy beneficiosos en industrias como la salud, las finanzas y el entretenimiento, donde el conocimiento especializado y la precisión son críticos. El marketplace permite a los desarrolladores descubrir, probar e integrar estos modelos especializados en sus flujos de trabajo de ML sin fricciones. Ya sea para la generación de texto, la creación de imágenes o tareas multimodales complejas, Amazon Bedrock proporciona las herramientas y la flexibilidad necesarias para crear aplicaciones potentes de IA generativa. Al aprovechar Amazon Bedrock, las empresas pueden mantenerse a la vanguardia de la innovación en IA, asegurando que sus aplicaciones sean a la vez punteras y seguras. La Converse API refuerza aún más esta capacidad al ofrecer una forma sencilla de interactuar con estos modelos, lo que hace el proceso de integración fluido y eficiente.

El **Amazon Bedrock Marketplace** tiene una diferencia de infraestructura que el libro no menciona y que cambia el modelo de costos. Los modelos del catálogo principal son serverless: pagas por token y no hay nada encendido. Los modelos del marketplace, en cambio, se despliegan en un **endpoint de SageMaker** dentro de tu cuenta, para el cual eliges el tipo y el número de instancias (normalmente con GPU, como `ml.g5.2xlarge`). Ese endpoint se cobra por hora de instancia mientras exista, aunque no reciba peticiones, y hay que borrarlo al terminar. Se invoca con las mismas APIs de Bedrock, pero su operación se parece más a la de SageMaker que a la de un servicio serverless.

Al seleccionar un modelo fundacional, es esencial considerar las capacidades específicas que necesita tu aplicación. Si tu tarea consiste principalmente en generar, analizar o traducir texto, modelos como Amazon Titan Text G1 – Premier, Amazon Titan Text G1 – Express y Amazon Titan Text G1 – Lite son ideales. Estos modelos están integrados en Amazon Bedrock y admiten una amplia gama de tareas relacionadas con texto, como responder preguntas abiertas, generar código, resumir y conversar. Para aplicaciones creativas, modelos como Amazon Nova Canvas destacan en la generación de imágenes detalladas a partir de descripciones de texto, lo que los hace perfectos para tareas de arte y diseño. Para capacidades multimodales, modelos como Amazon Nova Lite y Amazon Nova Pro pueden manejar entradas de texto e imagen, ofreciendo soluciones versátiles. Para casos de uso sencillos y presupuestos bajos, Amazon Nova Micro es un modelo solo de texto adecuado para escenarios que requieren procesamiento de texto de baja latencia sin necesidad de entrada multimodal. Evaluar las capacidades de estos modelos en el contexto de las necesidades específicas de tu aplicación te ayudará a elegir el modelo fundacional más eficaz.

> [!warning] Nota de precisión: catálogo de modelos (verificado el 25-09-2026)
>
> - **Titan Text G1 – Premier, Express y Lite** ya no aparecen en el catálogo de modelos de Bedrock. Para texto, la familia de Amazon vigente es Nova (Nova Micro, Lite y Pro de primera generación y Nova 2 Lite).
> - **Nova Canvas** está en estado _Legacy_ desde el 30-03-2026 y su fin de vida (EOL) es el 30-09-2026. En estado Legacy los clientes nuevos no pueden usarlo, y después del EOL las peticiones fallan en todas las regiones.
> - En Bedrock, cada modelo tiene un **ciclo de vida** (Active, Legacy, EOL) que se consulta en su ficha (_model card_) y en el campo `modelLifecycle` de la API `ListFoundationModels`. Cuando un modelo pasa a Legacy, la migración a otro modelo no es automática: hay que cambiar el `modelId` en el código.
> - Para el examen, lo que se evalúa es el criterio (texto frente a imagen, multimodal, costo y latencia), no memorizar nombres de modelos concretos, que cambian cada pocos meses.

Otro criterio que debes considerar al seleccionar un modelo es su disponibilidad en la región donde operará tu aplicación de IA generativa. Esto asegura un rendimiento óptimo, el cumplimiento de las regulaciones locales y una menor latencia, lo que ofrece una experiencia de usuario fluida. La razón de fondo de esas tres ventajas es física y legal. Una región de AWS es un conjunto de centros de datos en una zona geográfica concreta: cuanto más lejos esté la región del usuario o de tu aplicación, más milisegundos tarda cada viaje de ida y vuelta por la red. Y muchas regulaciones exigen **residencia de datos** (_data residency_), es decir, que ciertos datos se almacenen o procesen dentro de un país o jurisdicción determinados; si el modelo que necesitas no existe en esa región, no puedes usarlo sin violar la norma.

> [!note]
> Los FMs están disponibles región por región. Para saber qué modelos se admiten en tu región, visita https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html.

Para el examen, ten presente que algunos modelos pueden no estar disponibles en tu región, pero aun así pueden usarse con **inferencia entre regiones** (_cross-region inference_) si esta función está disponible en tu región. La inferencia entre regiones te permite gestionar sin fricciones picos de tráfico no planificados aprovechando capacidad de cómputo de distintas regiones de AWS. Con ella, puedes distribuir el tráfico entre varias regiones de AWS, lo que permite un mayor rendimiento y una mayor resiliencia en los periodos de máxima demanda. Para usar la inferencia entre regiones, necesitas crear un **perfil de inferencia entre regiones** (_cross-region inference profile_). Para saber más, visita https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html.

Un **perfil de inferencia** es un identificador que usas en lugar del `modelId` de un modelo concreto. El perfil define un conjunto de regiones de destino, y Bedrock enruta cada petición a la que tenga capacidad disponible en ese momento. AWS ya ofrece perfiles predefinidos, reconocibles por un prefijo geográfico en el identificador (como `us.` o `eu.`), además de perfiles globales; crear uno propio solo hace falta en casos concretos. El detalle importante para el cumplimiento normativo es que los perfiles **geográficos** mantienen el procesamiento dentro de esa geografía (por ejemplo, solo regiones de la Unión Europea), mientras que los **globales** pueden enrutar a cualquier región del mundo. El tráfico entre regiones viaja por la red de AWS, no por internet público.

Por último, igual que en cualquier aplicación bien diseñada (_well-architected_; las aplicaciones de IA generativa no son distintas en esto), necesitas considerar el precio. El precio de Amazon Bedrock se basa en dos modelos principales:

- Bajo demanda y por lotes (_On-Demand and Batch_)
- Rendimiento aprovisionado (_Provisioned Throughput_)

Con el modo bajo demanda y por lotes, pagas por el número de tokens de entrada y de salida procesados, donde un token es una secuencia de caracteres que el modelo interpreta como una sola unidad de significado. Con el rendimiento aprovisionado, te comprometes a un nivel específico de rendimiento durante un periodo de tiempo, lo que puede ser más rentable para aplicaciones con un uso constante. Para más información, visita https://aws.amazon.com/bedrock/pricing.

Para dar escala a la unidad de cobro: como regla aproximada que varía según el modelo y el idioma, 1 000 tokens equivalen a unas 750 palabras en inglés, y el español suele necesitar algo más de tokens por palabra. Los tokens de salida suelen costar varias veces más que los de entrada, así que una aplicación que resume documentos largos (mucha entrada, poca salida) tiene un perfil de costo muy distinto al de una que redacta textos largos. En el término **_throughput_** (rendimiento) aparece otra métrica de infraestructura: la cantidad de trabajo que el servicio completa por unidad de tiempo, aquí tokens por minuto. El rendimiento aprovisionado reserva esa capacidad para tu cuenta a cambio de pagarla por hora, la uses o no, igual que una instancia encendida.

> [!warning] Nota de precisión: modalidades de precio vigentes
> Hoy la página de precios de Bedrock distingue más opciones que las dos del libro. La inferencia **por lotes** (trabajos que leen las peticiones de S3 y dejan las respuestas en S3) cuesta alrededor de un 50 % menos que la bajo demanda en los modelos que la admiten. La inferencia bajo demanda se ofrece en **niveles de servicio** (_service tiers_): Priority (respuesta más rápida, con sobreprecio), Standard (el predeterminado) y Flex (más barato, para trabajos que toleran esperas), además de un nivel Reserved de capacidad reservada por uno o tres meses. El rendimiento aprovisionado (_Provisioned Throughput_) sigue documentado. Verifica los precios y las condiciones vigentes antes de estimar costos.

La siguiente sección ofrece un ejemplo sencillo de cómo usar la IA generativa de forma programática.

##### Uso de Nova Canvas para generar una imagen (_Using Nova Canvas to Generate an Image_)

Los FMs Nova se anunciaron recientemente en AWS re:Invent 2024. **re:Invent** es la conferencia anual de AWS, que se celebra en Las Vegas a finales de año y donde se anuncia buena parte de los servicios nuevos. En este caso de uso vamos a usar Nova Canvas, un modelo de generación de imágenes de última generación, para generar una imagen a partir de un prompt de texto. Este ejemplo te mostrará cómo implementar este caso de uso de forma programática.

> [!warning] El ejemplo ya no se puede reproducir tal cual
> Nova Canvas no admite clientes nuevos desde el 30-03-2026 y deja de funcionar el 30-09-2026 (ver la nota anterior). El código sigue siendo útil para entender cómo se invoca un modelo de imagen en Bedrock; para ejecutarlo hoy habría que cambiar el `modelId` y el cuerpo de la petición por los de un modelo de imagen vigente del catálogo, porque cada modelo de imagen define su propio formato de petición.

Como consideración de diseño, la región predeterminada de mi cuenta de AWS es us-east-2 (Ohio). Como Nova Canvas no está disponible de forma nativa en mi región predeterminada (y no quiero implementar la inferencia entre regiones), voy a crear mi cliente de Python en us-east-1 (Norte de Virginia), donde el modelo está disponible de forma nativa actualmente. Desde la consola de AWS, abrimos Amazon Bedrock y seleccionamos us-east-1 (Norte de Virginia) como región de trabajo. Después solicitamos acceso al modelo, como se muestra en la Figura 4.2.

La **región predeterminada** es la que usan las herramientas cuando no indicas otra. En boto3 se resuelve en este orden: el parámetro `region_name` al crear el cliente, las variables de entorno `AWS_REGION` o `AWS_DEFAULT_REGION` y, por último, el archivo de configuración `~/.aws/config`. Esto importa en el código que sigue.

_Figura 4.2 Solicitud de acceso a Nova Canvas._

> [!warning] Nota de precisión: ya no se solicita acceso a los modelos
> Desde octubre de 2025, Bedrock habilita automáticamente los modelos serverless en todas las regiones donde están disponibles, y la página de «acceso a modelos» de la Figura 4.2 dejó de ser necesaria. El control pasa a ser el estándar de AWS: una política de IAM decide qué identidades pueden invocar qué modelos. La excepción son los modelos de Anthropic, que exigen enviar una sola vez un formulario de caso de uso por cuenta (o una vez en la cuenta de administración de la organización) antes del primer uso.

El siguiente programa en Python usa el módulo boto3 para crear un cliente de Amazon Bedrock en us-east-1 y luego interactuar con el FM Nova Canvas para solicitar la generación de una imagen. Ejecutamos este programa con la aplicación Code Editor de Amazon SageMaker Studio en us-east-1.

> [!note]
> Amazon SageMaker Studio es un entorno de desarrollo integrado (IDE) basado en web para tus canalizaciones de ML de extremo a extremo. Viene de serie con la mayoría de las bibliotecas de ML que necesitas y ofrece una experiencia de desarrollo fluida similar a Visual Studio (VS) Code.

Para empezar a usar Amazon SageMaker Studio, necesitas crear un **dominio** en la región donde se ejecutará tu programa (en nuestro caso, us-east-1). Un dominio de SageMaker es la configuración de Studio para un grupo de usuarios dentro de una región: define la red (la VPC) por la que salen las aplicaciones, el método de autenticación, los **perfiles de usuario** de cada persona y el **rol de ejecución** por defecto, que es el rol de IAM con cuyos permisos actúa todo lo que ejecutas desde Studio. Una **VPC** (_Virtual Private Cloud_) es una red privada virtual dentro de AWS, aislada de las de otros clientes, donde se colocan los recursos y se controla con reglas qué tráfico entra y sale. Una vez que tu dominio está disponible, puedes abrir Amazon SageMaker Studio, como se muestra en la Figura 4.3. La Figura 4.4 ilustra la página de inicio de Studio, donde puedes crear un espacio. Un **espacio** (_space_) es una instancia administrada que Amazon SageMaker Studio usa para lanzar la aplicación Code Editor. En la Figura 4.4, inicié un espacio `dario-ai-space` que usa un tipo de instancia `ml.t3.medium` con 5 GB de almacenamiento.

El nombre del tipo de instancia se lee por partes. El prefijo `ml.` indica que es una instancia administrada por SageMaker, con precio propio (distinto del de la misma máquina en EC2). `t3` es la familia y la generación: la familia T es de **rendimiento ampliable** (_burstable_), máquinas baratas que acumulan «créditos» de CPU cuando están ociosas y los gastan en ráfagas, pensadas para trabajo interactivo ligero como escribir código, no para entrenar. `medium` es el tamaño: 2 vCPU y 4 GiB de memoria, algo menos que una laptop básica. Si en el mismo espacio hubiera que procesar un dataset grande, se detendría el espacio y se cambiaría el tipo de instancia por uno mayor, sin perder los archivos.

Para desarrollar tus aplicaciones de ML, necesitas crear e iniciar tu espacio, como se muestra en la Figura 4.5. Tu espacio tiene conectado un Elastic File System (EFS), donde se guardan el código y otros artefactos de tus aplicaciones. Para evitar costos no deseados, asegúrate de detener tu espacio cuando termines de programar. Un espacio encendido es una instancia que se cobra por hora aunque no estés escribiendo; detenerlo apaga la instancia, pero conserva los archivos, cuyo almacenamiento sí se sigue cobrando, aunque mucho menos.

_Figura 4.3 Amazon SageMaker Studio._

_Figura 4.4 Espacio de Amazon SageMaker Studio._

_Figura 4.5 Code Editor de Amazon SageMaker Studio._

> [!warning] Nota de precisión: el almacenamiento del espacio es EBS, no EFS
> En la versión actual de Studio, cada espacio de JupyterLab o Code Editor usa **una instancia para el cómputo y un volumen de Amazon EBS para el almacenamiento**; en ese volumen viven el directorio de trabajo `/home/sagemaker-user`, el código, la configuración de Git y las variables de entorno. Los «5 GB de almacenamiento» de la Figura 4.4 son precisamente el tamaño predeterminado de ese volumen EBS (ampliable por el administrador). **EFS** era el almacenamiento del directorio personal en **Studio Classic**, la versión anterior, y hoy puede montarse de forma opcional en un espacio si el administrador lo configura.
>
> La diferencia no es de nombre. **Amazon EBS** (_Elastic Block Store_) es **almacenamiento de bloques**: un disco virtual que se conecta a una sola máquina, como el SSD de tu laptop, y que el sistema operativo formatea y usa como disco propio. **Amazon EFS** (_Elastic File System_) es **almacenamiento de archivos**: una carpeta compartida por red, a la que varias máquinas se conectan («montan») a la vez mediante **NFS**, el protocolo estándar de Linux para usar una carpeta remota como si fuera local; crece y se reduce sola y se cobra por lo que ocupa. En la práctica, con EBS tus archivos pertenecen a tu espacio y otro espacio no los ve; con EFS varios usuarios pueden compartirlos. Según la documentación, el volumen EBS del espacio ofrece 3 000 IOPS y 125 MB/s. Las **IOPS** son operaciones de lectura o escritura por segundo, y los MB/s miden el **throughput** de disco; como referencia, un SSD NVMe de laptop moderno supera con holgura ambas cifras, así que este volumen está pensado para editar código y datasets moderados, no para leer terabytes. Verifica las cifras vigentes en la documentación.

```python
import base64
import io
import json
import logging
import os
import boto3
from PIL import Image
from botocore.config import Config
from botocore.exceptions import ClientError

class ImageError(Exception):
    "Custom exception for errors returned by Amazon Nova Canvas"
    def __init__(self, message):
        self.message = message

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

def generate_image(model_id, body):
    """
    Generate an image using Amazon Nova Canvas model on demand.
    Args:
        model_id (str): The model ID to use.
        body (str) : The request body to use.
    Returns:
        image_bytes (bytes): The image generated by the model.
    """
    logger.info(f"Generating image with Amazon Nova Canvas model {model_id}")

    bedrock = boto3.client(
        service_name='bedrock-runtime',
        config=Config(read_timeout=300)
    )

    accept = "application/json"
    content_type = "application/json"

    response = bedrock.invoke_model(
        body=body, modelId=model_id, accept=accept, contentType=content_type
    )
    response_body = json.loads(response.get("body").read())

    base64_image = response_body.get("images")[0]
    base64_bytes = base64_image.encode('ascii')
    image_bytes = base64.b64decode(base64_bytes)

    finish_reason = response_body.get("error")

    if finish_reason is not None:
        raise ImageError(f"Image generation error. Error is {finish_reason}")

    logger.info(f"Successfully generated image with Amazon Nova Canvas model {model_id}")

    return image_bytes

def main():
    """
    Entrypoint for Amazon Nova Canvas example.
    """
    logging.basicConfig(level=logging.INFO,
                        format="%(levelname)s: %(message)s")

    model_id = 'amazon.nova-canvas-v1:0'
    prompt = "Generate an image of a white sand beach at sunset."

    body = json.dumps({
        "taskType": "TEXT_IMAGE",
        "textToImageParams": {
            "text": prompt
        },
        "imageGenerationConfig": {
            "numberOfImages": 1,
            "height": 1024,
            "width": 1024,
            "cfgScale": 8.0,
            "seed": 0
        }
    })

    try:
        image_bytes = generate_image(model_id=model_id, body=body)
        image = Image.open(io.BytesIO(image_bytes))

        # Create directory if it doesn't exist
        efs_directory = '/home/sagemaker-user/ch04/images'
        if not os.path.exists(efs_directory):
            os.makedirs(efs_directory)

        # Save the image to the EFS attached file system
        output_image_path = os.path.join(efs_directory, "white_sand_beach_sunset.png")
        image.save(output_image_path)
        print(f"Image saved to {output_image_path}")

    except ClientError as err:
        message = err.response["Error"]["Message"]
        logger.error(f"A client error occurred: {message}")
        print(f"A client error occurred: {message}")
    except ImageError as err:
        logger.error(err.message)
        print(err.message)
    else:
        print(f"Finished generating image with Amazon Nova Canvas model {model_id}.")

if __name__ == "__main__":
    main()
```

Guardé este programa en Code Editor como `test_nova_canvas.py`.

Este programa usa el módulo boto3 para construir un objeto cliente llamado `bedrock` que interactúa con Amazon Bedrock en us-east-1, donde Nova Canvas está disponible de forma nativa. Como se muestra en el método `generate_image`, este cliente usa el método `invoke_model` para enviar un prompt (encapsulado como la propiedad `text` del parámetro `body`) que pide crear la imagen de una playa de arena blanca al atardecer.

Hay cuatro detalles de infraestructura en ese código que el libro da por sentados:

- **Por qué el cliente es `bedrock-runtime` y no `bedrock`.** AWS separa en muchos servicios el **plano de control** (_control plane_), las operaciones que crean, listan o configuran recursos, del **plano de datos** (_data plane_), las que usan esos recursos. En Bedrock, el cliente `bedrock` sirve para tareas de administración (listar modelos, crear trabajos de personalización, gestionar perfiles de inferencia) y el cliente `bedrock-runtime` sirve para invocar modelos (`invoke_model`, `converse`). Tienen endpoints distintos y acciones de IAM distintas.
- **De dónde sale us-east-1.** El cliente se crea sin `region_name`, así que usa la región predeterminada del entorno. Funciona porque el programa corre en un espacio de Studio de un dominio en us-east-1, donde esa es la región del entorno. Ejecutado en tu laptop con la región us-east-2 configurada, el mismo código apuntaría a us-east-2 y fallaría. Para que el comportamiento no dependa del entorno, pasa `region_name='us-east-1'` al crear el cliente.
- **Por qué `read_timeout=300`.** botocore, la biblioteca sobre la que funciona boto3, espera por defecto hasta 60 segundos una respuesta antes de dar la conexión por perdida. Generar una imagen puede tardar más, así que el código amplía la espera a 300 segundos para no abortar una petición que el servicio sí está atendiendo.
- **Qué permisos necesita.** La llamada se hace con las credenciales del rol de ejecución del espacio. Ese rol necesita una política de IAM que permita la acción `bedrock:InvokeModel` sobre el ARN del modelo (el identificador único de un recurso de AWS). Si falta, la llamada falla con un `AccessDeniedException`, que el programa captura en el bloque `except ClientError`. Los encabezados `accept` y `contentType` son los tipos de contenido HTTP (tipos MIME) con los que se declara que tanto la petición como la respuesta van en JSON.

Al ejecutarse en Amazon SageMaker Studio, este código genera la imagen solicitada, como se muestra en las Figuras 4.6 y 4.7.

_Figura 4.6 Salida del programa._

_Figura 4.7 Una imagen generada por Nova Canvas._

Una vez generada la imagen, el programa decodifica los datos de la imagen en base64 y la guarda como `white_sand_beach_sunset.png` en la carpeta `/home/sagemaker-user/ch04/images` del EFS montado en tu instancia de Amazon SageMaker Studio.

> [!note]
> Las instancias de Amazon SageMaker Studio vienen con EFS montado de forma predeterminada, de modo que cualquier archivo guardado en directorios locales de la instancia se almacena en el EFS conectado. Esto asegura que la imagen generada quede disponible y persista entre distintas sesiones de Amazon SageMaker Studio.

Lo que el libro atribuye a EFS vale hoy para el volumen EBS del espacio (ver la nota de precisión anterior): la imagen persiste entre sesiones porque `/home/sagemaker-user` vive en un volumen independiente de la instancia, que sobrevive cuando detienes el espacio o cambias el tipo de instancia. El nombre de la variable `efs_directory` del código es solo eso, un nombre; la ruta funciona igual con cualquiera de los dos almacenamientos.

El programa también incluye un manejo de errores completo para asegurar un proceso fluido, registrando mensajes significativos por el camino. Esta configuración asegura que la imagen se genere y se almacene de forma eficiente, lista para usarse o mostrarse después.

> [!note]
> Durante la escritura de este libro, Amazon SageMaker pasó a llamarse Amazon SageMaker AI. Para el examen, ten en cuenta este cambio de nombre. Siempre que el libro mencione Amazon SageMaker (el servicio), se refiere a Amazon SageMaker AI (https://aws.amazon.com/blogs/aws/introducing-the-next-generation-of-amazon-sagemaker-the-center-for-all-your-data-analytics-and-ai).

El cambio de nombre se debe a que «Amazon SageMaker» designa ahora una plataforma más amplia que reúne datos, analítica e IA (con SageMaker Unified Studio como entorno común), y «SageMaker AI» es el servicio de ML que describe este libro: entrenamiento, endpoints, algoritmos integrados, Studio y el resto.

##### Casos de uso (_Use Cases_)

Amazon Bedrock es una opción excelente para los desarrolladores que buscan crear aplicaciones de IA generativa con modelos fundacionales robustos. Es ideal para tareas como la generación de texto, la creación de imágenes y el procesamiento multimodal, gracias a su integración con modelos de alto rendimiento como Amazon Titan y Amazon Nova. Si necesitas capacidades de IA escalables, confiables y fáciles de usar, con infraestructura administrada, Amazon Bedrock ofrece un entorno fluido tanto para experimentar como para desplegar. Además, el precio de pago por uso (_pay-as-you-go_) de Amazon Bedrock lo hace accesible para proyectos de distintos tamaños, desde pequeñas startups hasta grandes empresas. **Pago por uso** significa que no hay cuota fija ni compromiso: si un mes no envías peticiones, no pagas nada por la inferencia bajo demanda.

En cambio, Amazon Bedrock podría no ser la mejor opción si tu aplicación requiere modelos muy especializados o de dominio específico que no estén disponibles en la plataforma o en el marketplace. Si necesitas modelos entrenados con datos propietarios, tienes requisitos estrictos de residencia de datos o requisitos de latencia que exigen soluciones muy especializadas, otras plataformas o modelos construidos a medida podrían ser más apropiados. Además, si el costo es una preocupación principal y tu proyecto tiene necesidades mínimas de IA, soluciones más simples y rentables podrían bastar, sin las capacidades avanzadas de Amazon Bedrock.

> [!warning] Nota de precisión: modelos entrenados con datos propietarios
> Tener datos propietarios no obliga por sí solo a salir de Bedrock. Además del ajuste fino, Bedrock ofrece **Custom Model Import**, que permite cargar los pesos de un modelo propio (de arquitecturas compatibles, como algunas de las familias Llama, Mistral o Qwen; verifica la lista vigente) e invocarlo con las mismas APIs serverless. La alternativa a Bedrock dentro de AWS, cuando el modelo o su configuración de despliegue no encajan, es alojarlo tú mismo en un endpoint de SageMaker AI, donde eliges instancias, contenedor y red, a cambio de administrarlo y pagar las instancias por hora.

## Desarrollar modelos con los algoritmos integrados de Amazon SageMaker (_Developing Models with Amazon SageMaker Built-in Algorithms_)

Aunque los servicios de IA de AWS como Amazon Rekognition, Comprehend y Personalize ofrecen soluciones potentes y preconstruidas para tareas comunes como el análisis de imágenes, el procesamiento de texto y los sistemas de recomendación, pueden no ofrecer la flexibilidad necesaria para casos de uso especializados. Estos servicios administrados están diseñados para ser fáciles de usar y requerir una experiencia mínima en ML, lo que los hace adecuados para despliegues rápidos y aplicaciones estándar. Sin embargo, cuando tu proyecto exige ingeniería de características a medida, arquitecturas de modelo avanzadas u optimización para métricas de rendimiento específicas, los algoritmos integrados de Amazon SageMaker ofrecen las herramientas y la flexibilidad necesarias para lograr esos objetivos.

Elegir construir modelos con los algoritmos integrados de Amazon SageMaker ofrece varias ventajas, en particular cuando necesitas más control y personalización sobre tus proyectos de ML. Los algoritmos integrados de Amazon SageMaker están altamente optimizados en velocidad, escala y exactitud, lo que te permite entrenar, refinar y desplegar de forma eficiente modelos adaptados a las necesidades específicas de tu negocio. Este enfoque es ideal cuando necesitas ajustar hiperparámetros, incorporar conocimiento del dominio o manejar requisitos únicos de preprocesamiento de datos. Al aprovechar los algoritmos integrados de Amazon SageMaker, puedes lograr un mayor grado de precisión y rendimiento, especialmente en aplicaciones complejas o hechas a medida donde las soluciones listas para usar se quedan cortas.

Conviene precisar qué es físicamente un «algoritmo integrado», porque de ahí se derivan casi todas las preguntas de configuración. Cada algoritmo integrado es una **imagen de contenedor** que AWS mantiene. Un contenedor (en la práctica, Docker) empaqueta un programa con todas sus dependencias (sistema operativo base, bibliotecas, código de entrenamiento) para que se ejecute igual en cualquier máquina; la imagen es la plantilla desde la que se arranca. AWS publica esas imágenes en **Amazon ECR** (_Elastic Container Registry_), su registro de imágenes de contenedor, con una copia por región. Tú no escribes código de entrenamiento: aportas los datos en el formato que el algoritmo espera, eliges los hiperparámetros y el tipo de instancia, y SageMaker arranca el contenedor. Es la primera de tres formas de entrenar en SageMaker; las otras dos son el **modo script** (_script mode_), en el que escribes tu propio código de entrenamiento y lo ejecutas dentro de un contenedor de framework que mantiene AWS (scikit-learn, PyTorch, TensorFlow, XGBoost), y **traer tu propio contenedor** (_bring your own container_), en el que construyes tú la imagen completa.

Además, Amazon SageMaker ofrece una integración fluida con el ecosistema más amplio de AWS, lo que asegura un flujo de trabajo cohesivo desde la preparación de datos hasta el despliegue del modelo. Puedes importar datos fácilmente desde Amazon S3, Amazon EFS o Amazon FSx for Lustre; usar funciones de AWS Lambda para el preprocesamiento; y desplegar modelos mediante endpoints de Amazon SageMaker o AWS Lambda para la inferencia en tiempo real. Esta estrecha integración simplifica el ciclo de vida de ML, mejora la eficiencia operativa y reduce el tiempo y el esfuerzo necesarios para gestionar los distintos componentes de la canalización de ML. También te permite aprovechar las capacidades de entrenamiento distribuido de Amazon SageMaker, que pueden acelerar significativamente el entrenamiento de modelos con datasets grandes.

Cada pieza de ese párrafo tiene implicaciones de infraestructura:

- **Las tres fuentes de datos.** **S3** es la fuente por defecto y la única que no requiere configuración de red. **EFS** es el sistema de archivos compartido por NFS que se explicó en la sección de Bedrock; conviene cuando los datos ya viven allí porque otros sistemas los usan. **Amazon FSx for Lustre** es un sistema de archivos administrado basado en **Lustre**, un sistema de archivos paralelo de código abierto usado en supercómputo, que reparte cada archivo entre muchos servidores para que cientos de procesos lean a la vez; en ML se usa como caché de alta velocidad delante de un bucket de S3 cuando el entrenamiento con GPU caras espera por los datos. Para leer de EFS o FSx, el trabajo de entrenamiento debe ejecutarse **dentro de tu VPC**: hay que indicarle subredes y grupos de seguridad (las reglas de firewall de AWS) desde los que alcance el sistema de archivos. Con S3 eso no hace falta.
- **Cómo lee los datos el entrenamiento desde S3.** En **modo File**, SageMaker copia todo el dataset de S3 al disco de la instancia antes de empezar, así que el disco debe ser suficientemente grande y el arranque tarda más cuanto mayor es el dataset. En **modo Pipe**, los datos se transmiten desde S3 mientras el algoritmo entrena, sin copia previa. El **modo FastFile**, más reciente, presenta los objetos de S3 como archivos locales, pero los descarga a medida que se leen. No todos los algoritmos integrados admiten Pipe (ver la tabla de abajo).
- **Lambda para preprocesar** sirve para transformaciones ligeras por evento (limpiar un registro, cambiar un formato), porque una ejecución de Lambda dura como máximo 15 minutos y dispone de hasta unos 10 GB de memoria (verifica los límites vigentes). Para procesar un dataset completo se usan trabajos de procesamiento de SageMaker, AWS Glue o Amazon EMR.
- **Un endpoint de SageMaker** es un servicio HTTPS persistente que SageMaker levanta sobre una o más instancias con tu modelo cargado; tu aplicación le envía datos y recibe predicciones en milisegundos. SageMaker administra las instancias, los parches y el balanceo de carga entre ellas, pero **el endpoint se cobra por hora de instancia mientras exista**, reciba o no peticiones, así que hay que borrarlo cuando deja de usarse. **Lambda como alternativa** sirve para modelos pequeños con tráfico esporádico: no cobra en reposo, pero la primera invocación tras un periodo inactivo tarda más (el llamado _cold start_) y no tiene GPU.
- **El entrenamiento distribuido** consiste en repartir un mismo entrenamiento entre varias instancias. Solo algunos algoritmos integrados lo admiten.

> [!tip] Relevancia para el examen
> La configuración de datos de entrenamiento en EFS o FSx for Lustre aparece en el MLA-C01, pero ya no figura entre las habilidades del MLA-C02. S3 como origen de los datos, y los modos File y Pipe, siguen siendo materia de ambos.

En esta sección exploraremos los algoritmos integrados que ofrece Amazon SageMaker, sus casos de uso apropiados y cómo usarlos eficazmente para construir modelos de ML robustos. La Figura 4.8 puede ayudarte a relacionar los algoritmos integrados con tipos específicos de problemas de ML, formatos de entrada y casos de uso.

_Figura 4.8 Algoritmos integrados de Amazon SageMaker._

Como la figura no está en el `.txt`, la tabla siguiente reúne los datos de infraestructura de cada algoritmo integrado que trata el capítulo, tomados de la tabla «Parameters for Built-in Algorithms» de la documentación de SageMaker (verificada el 25-09-2026). Es la información que suele decidir las preguntas de configuración del examen: qué formato de archivo espera cada algoritmo, si admite el modo Pipe, si necesita GPU y si puede entrenar en varias instancias.

| Algoritmo                    | Formato de los datos de entrenamiento                                | Modo de entrada | Instancias de entrenamiento           | ¿Varias instancias? |
| ---------------------------- | -------------------------------------------------------------------- | --------------- | ------------------------------------- | ------------------- |
| Linear Learner               | recordIO-protobuf o CSV                                              | File o Pipe     | CPU o GPU                             | Sí                  |
| k-NN                         | recordIO-protobuf o CSV                                              | File o Pipe     | CPU o GPU (una GPU por instancia)     | Sí                  |
| XGBoost                      | CSV, LibSVM o Parquet                                                | File o Pipe     | CPU (GPU desde la versión 1.2)        | Sí                  |
| Factorization Machines       | recordIO-protobuf                                                    | File o Pipe     | CPU (GPU para datos densos)           | Sí                  |
| Object2Vec                   | JSON Lines                                                           | File            | CPU o GPU (una sola instancia)        | No                  |
| DeepAR                       | JSON Lines o Parquet                                                 | File            | CPU o GPU                             | Sí                  |
| K-Means                      | recordIO-protobuf o CSV                                              | File o Pipe     | CPU o GPU (una GPU por instancia)     | No\*                |
| PCA                          | recordIO-protobuf o CSV                                              | File o Pipe     | CPU o GPU                             | Sí                  |
| LDA                          | recordIO-protobuf o CSV                                              | File o Pipe     | CPU (una sola instancia)              | No                  |
| NTM                          | recordIO-protobuf o CSV                                              | File o Pipe     | CPU o GPU                             | Sí                  |
| Random Cut Forest            | recordIO-protobuf o CSV                                              | File o Pipe     | Solo CPU                              | Sí                  |
| IP Insights                  | CSV                                                                  | File            | CPU o GPU                             | Sí                  |
| BlazingText                  | Texto: una oración por línea, con tokens separados por espacios      | File o Pipe     | CPU o GPU (una sola instancia)        | No                  |
| Seq2Seq                      | recordIO-protobuf (más un canal `vocab`)                             | File            | Solo GPU (una sola instancia)         | No                  |
| Image Classification – MXNet | recordIO o imágenes .jpg/.png                                        | File o Pipe     | Solo GPU                              | Sí                  |
| Object Detection – MXNet     | recordIO o imágenes .jpg/.png                                        | File o Pipe     | Solo GPU                              | Sí                  |
| Semantic Segmentation        | Imágenes                                                             | File o Pipe     | Solo GPU (una sola instancia)         | No                  |

\*La misma tabla de AWS indica para K-Means «una GPU en una o más instancias», lo que contradice el «No»; consulta la página del algoritmo si la pregunta depende de ello.

Dos formatos de la tabla necesitan explicación. **CSV** en SageMaker tiene reglas propias: el archivo va **sin fila de encabezado** y, en los algoritmos supervisados, **con la etiqueta en la primera columna**; en los no supervisados, que no tienen etiqueta, hay que declararlo con el tipo de contenido `text/csv;label_size=0`, porque si no el algoritmo toma la primera columna como etiqueta. **recordIO-protobuf** es el formato binario nativo de muchos algoritmos de AWS. Combina dos piezas: **Protocol Buffers** (protobuf), un formato de serialización binaria creado por Google, parecido a un JSON compacto con esquema fijo, que se lee mucho más rápido que texto; y **RecordIO**, un formato que concatena muchos registros en un solo archivo con un encabezado de longitud por registro, de modo que se pueden leer uno tras otro en flujo continuo. Esa combinación es la que hace eficiente el modo Pipe. Cada observación se guarda como un vector de números de 32 bits, y el SDK de SageMaker para Python trae funciones para convertir arreglos de NumPy a este formato.

### Algoritmos de ML supervisado (_Supervised ML Algorithms_)

Como aprendiste en el capítulo 1, el ML supervisado es un tipo de algoritmo que aprende de datos de entrenamiento etiquetados para hacer predicciones o tomar decisiones. En el aprendizaje supervisado, cada ejemplo de entrenamiento incluye datos de entrada en forma de un conjunto de características, junto con la salida o etiqueta correspondiente. El algoritmo usa estos ejemplos etiquetados para aprender la correspondencia entre entradas y salidas, y después puede aplicar esa correspondencia aprendida a datos nuevos, no vistos, para predecir resultados.

Este enfoque es particularmente útil cuando existe una relación clara y predefinida entre las características de entrada y la etiqueta correspondiente, también conocida como variable objetivo. Entre las aplicaciones comunes del aprendizaje supervisado están tareas como la regresión, cuyo objetivo es predecir un valor continuo (p. ej., predecir el precio de una casa a partir de características como el tamaño y la ubicación), y la clasificación, cuyo objetivo es categorizar los datos en clases discretas (p. ej., identificar si un correo electrónico es spam o no).

El aprendizaje supervisado debe usarse cuando hay datos etiquetados disponibles y el objetivo es hacer predicciones exactas a partir de ellos. Dentro del aprendizaje supervisado hay varias vías, según la naturaleza de tu problema de ML.

Para los problemas de regresión, pueden emplearse algoritmos como la regresión lineal, los árboles de decisión y XGBoost para predecir valores continuos.

En la clasificación binaria, algoritmos como la regresión logística, las máquinas de vectores de soporte y los k vecinos más cercanos (k-NN) ayudan a distinguir entre dos clases.

Para la clasificación multiclase, en la que los datos deben categorizarse en más de dos clases, se usan comúnmente enfoques como los árboles de decisión, los bosques aleatorios (_random forests_) y las redes neuronales.

Además, el aprendizaje supervisado abarca algoritmos de pronóstico de series de tiempo como DeepAR, que predice valores futuros a partir de observaciones pasadas. Asimismo, algoritmos como Object2Vec se usan para crear _embeddings_ que capturan información semántica de objetos o entidades, lo que mejora el rendimiento de tareas posteriores como los sistemas de recomendación o la búsqueda semántica.

#### Algoritmos generales de regresión y clasificación (_General Regression and Classification Algorithms_)

Esta sección presenta los algoritmos supervisados de regresión y clasificación más comunes que ofrece Amazon SageMaker.

##### Linear Learner

Empezaremos con el algoritmo Linear Learner de Amazon SageMaker, que ofrece una solución tanto para problemas de clasificación como de regresión.

En los casos de uso de regresión lineal, alimentas al algoritmo con muestras etiquetadas $(\mathbf{x}, y)$. $\mathbf{x}$ es un vector de alta dimensión de valores de características, $\mathbf{x} \in \mathbb{R}^d$, e $y$ es un número real que denota la etiqueta asociada al punto de datos de la muestra. _(Notación reconstruida: en el `.txt` los símbolos aparecen como huecos.)_

- En los problemas de clasificación binaria, la etiqueta debe ser 0 o 1.
- En los problemas de clasificación multiclase, las etiquetas deben ir de 0 a `num_classes` – 1.

El algoritmo aprende una función lineal o, en los problemas de clasificación, una función lineal de umbral, y asigna a un vector $\mathbf{x}$ una aproximación de la etiqueta $y$.

El algoritmo Linear Learner requiere una matriz de datos, con filas que representan las observaciones (o tus puntos de datos) y columnas que representan las dimensiones de las características. También requiere una columna adicional que contenga las etiquetas correspondientes a los puntos de datos. En CSV, esa columna adicional debe ser la **primera**, y el archivo no debe llevar encabezado; en recordIO-protobuf, la etiqueta va en un campo propio de cada registro. Como mínimo, Linear Learner de Amazon SageMaker requiere que especifiques como argumentos las ubicaciones de los datos de entrada y de salida y el tipo de objetivo (clasificación o regresión). La dimensión de las características también es obligatoria. Para más información, consulta https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateTrainingJob.html.

`CreateTrainingJob`, la página que enlaza el libro, es la operación de la API de SageMaker que crea un trabajo de entrenamiento. Todo lo que hace el SDK de Python al entrenar termina en esa llamada, que declara la imagen del algoritmo, el rol de IAM, los **canales** de datos de entrada (cada canal es un nombre, como `train` o `validation`, asociado a una ubicación de S3 y a un tipo de contenido), la ubicación de S3 donde dejar el modelo, el tipo y el número de instancias y los hiperparámetros.

> [!warning] Nota de precisión: parámetros obligatorios de Linear Learner (verificado el 25-09-2026)
> Según la documentación vigente, el único hiperparámetro siempre obligatorio es `predictor_type` (`binary_classifier`, `multiclass_classifier` o `regressor`), que es el «tipo de objetivo» del libro; `num_classes` es obligatorio solo si el tipo es `multiclass_classifier`. **`feature_dim` ya no es obligatorio**: su valor predeterminado es `auto` y el algoritmo lo deduce de los datos.

> [!note]
> El algoritmo Linear Learner de Amazon SageMaker requiere que los datos de entrada estén en un archivo CSV o en formato RecordIO-protobuf.

Puedes especificar parámetros adicionales en el mapa de cadenas `HyperParameters` del cuerpo de la petición. Estos parámetros controlan el procedimiento de optimización o aspectos específicos de la función objetivo con la que entrenas, como el número de épocas, la regularización y el tipo de pérdida. Que sea un «mapa de cadenas» significa que todos los valores viajan en la API como texto (`"epochs": "15"`, no el número 15); el SDK de Python hace la conversión por ti.

Profundicemos en la regresión lineal y la regresión logística, que son los casos de uso de modelado lineal más comunes para problemas de regresión y de clasificación, respectivamente. Las máquinas de vectores de soporte (SVM) también se incluyen como otra opción del algoritmo Linear Learner.

> [!warning] Nota de precisión: qué «variante» de Linear Learner eliges y cómo
> Linear Learner es **un solo algoritmo**; la regresión lineal, la regresión logística y la SVM no son algoritmos integrados separados, sino configuraciones del hiperparámetro `loss` según el `predictor_type`. Con `regressor`, `squared_loss` (el valor por defecto) corresponde a la regresión lineal, y las pérdidas `eps_insensitive_squared_loss` y `eps_insensitive_absolute_loss` a una regresión de tipo SVR lineal. Con `binary_classifier`, `logistic` (por defecto) corresponde a la regresión logística y `hinge_loss` a una SVM lineal. Linear Learner **no implementa kernels** (polinomial, RBF), así que las SVM no lineales que describe la sección siguiente no están disponibles en este algoritmo integrado; para usarlas en SageMaker hay que recurrir al modo script con scikit-learn.

###### Regresión lineal (_Linear Regression_)

La regresión lineal es uno de los algoritmos más simples y más usados del ML supervisado. Su objetivo principal es modelar la relación entre una variable dependiente (también llamada variable objetivo o de salida) y una o más variables independientes (llamadas características o predictores). Lo hace ajustando una ecuación lineal a los datos observados. La ecuación lineal puede escribirse de la siguiente forma _(fórmula reconstruida)_:

$$
y = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \dots + \beta_n x_n + \varepsilon
$$

donde $y$ es la salida realmente observada, $\beta_0$ es el intercepto, $\beta_1, \dots, \beta_n$ son los coeficientes (o pesos) de las características $x_1, \dots, x_n$, y $\varepsilon$ representa el error, también llamado residuo (la diferencia entre los valores reales y los predichos).

Tu objetivo es predecir los coeficientes $\beta_0, \dots, \beta_n$: el modelo aprenderá a minimizar el error $\varepsilon$ en todo el dataset.

Al evaluar el modelo, observamos los residuos para entender qué tan bien se está desempeñando. En la regresión lineal, el modelo busca la función lineal que minimiza el error cuadrático medio (MSE, _mean squared error_). La fórmula del MSE es la siguiente _(fórmula reconstruida)_:

$$
\mathrm{MSE} = \frac{1}{n} \sum_{i=1}^{n} \left(y_i - \hat{y}_i\right)^2
$$

donde $n$ denota el número de observaciones de tu dataset, $y_i$ denota el valor real de la observación $i$, y $\hat{y}_i$ denota el valor predicho para la observación $i$.

Minimizar el MSE ayuda a que las predicciones queden lo más cerca posible de los valores reales, reduciendo de forma efectiva el error total.

Puedes usar la clase `LinearRegression` del módulo `sklearn.linear_model` para construir un modelo de regresión lineal a partir de un dataset, como se muestra en el siguiente fragmento:

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.linear_model import LinearRegression

# Generate some sample data
np.random.seed(0)
X = 2 * np.random.rand(100, 1)
y = 4 + 3 * X + np.random.randn(100, 1)

# Fit a linear regression model
model = LinearRegression()
model.fit(X, y)
y_pred = model.predict(X)

# Plot the data points
plt.scatter(X, y, color='blue', label='Data Points')

# Plot the regression line
plt.plot(X, y_pred, color='red', label='Regression Line')

# Plot the residuals
for i in range(len(X)):
    plt.plot([X[i], X[i]], [y[i], y_pred[i]], color='green', linestyle='--')

plt.xlabel('X')
plt.ylabel('y')
plt.title('Linear Regression with Residuals')
plt.legend()
plt.show()

# Print Mean Squared Error
mse = np.mean((y - y_pred) ** 2)
print(f"Mean Squared Error: {mse}")
```

Este fragmento y los demás programas de scikit-learn y XGBoost del capítulo no usan ningún servicio de AWS. Se ejecutan en la máquina donde los lances: tu laptop o la instancia de un espacio de SageMaker Studio. En este último caso, el cómputo lo aporta la instancia del espacio (la `ml.t3.medium` del ejemplo anterior), no se lanza ningún trabajo de SageMaker y no hay más costo que el del espacio encendido.

Veamos cómo funciona este programa:

- **Generación de datos.** Generamos una matriz $X$ con 100 filas (muestras) y 1 columna (característica). Cada fila denota el valor (observación o muestra) de una sola característica, que es un número aleatorio distribuido uniformemente entre 0 y 2. Después definimos una relación lineal $y = 4 + 3X$ con un residuo añadido _(expresión reconstruida a partir del código)_. En el programa, $y$ representa el vector de variables objetivo correspondiente a cada muestra.
- **Ajuste del modelo.** Creamos y ajustamos un modelo de regresión lineal con `LinearRegression` de scikit-learn.
- **Visualización.** Como se ilustra en la Figura 4.9, graficamos los puntos de datos como dispersión y trazamos la recta de regresión. Por último, dibujamos líneas discontinuas para mostrar los residuos (las diferencias entre los valores reales y los predichos) de cada punto de datos (100 en total).

_Figura 4.9 Regresión lineal._

- **MSE.** Calculamos e imprimimos el MSE para cuantificar el rendimiento del modelo.

Salida del `print` al ejecutar el fragmento con las versiones del encabezado:

```text
Mean Squared Error: 0.9924386487246479
```

###### Regresión logística (_Logistic Regression_)

A pesar de su nombre, la regresión logística es un algoritmo de clasificación, que permite modelar la probabilidad de una predicción de categoría correcta usando una combinación lineal de las características de entrada. Este algoritmo es particularmente útil cuando la variable dependiente es binaria, es decir, cuando solo puede tomar dos resultados posibles: predicción de categoría correcta o predicción de categoría incorrecta.

La regresión logística funciona aplicando una función logística a la combinación lineal de las características de entrada, lo que transforma la salida para que caiga dentro del rango de 0 a 1. Esta salida puede interpretarse entonces como la probabilidad de que ocurra el evento, con un umbral (comúnmente 0.5) que se usa para clasificar el evento en una de las dos categorías posibles.

La función logística, también conocida como función sigmoide, es clave en la regresión logística y se define así _(fórmula reconstruida)_:

$$
\sigma(z) = \frac{1}{1 + e^{-z}}
$$

En la fórmula, $z$ es la combinación lineal de las características de entrada y sus coeficientes correspondientes _(fórmula reconstruida)_:

$$
z = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \dots + \beta_n x_n
$$

Como se ilustra en la Figura 4.10, la función logística transforma cualquier número real en una medida de probabilidad (es decir, en el intervalo unitario $(0, 1)$), lo que la hace adecuada para estimar probabilidades. La curva en forma de S de la función sigmoide asegura que, a medida que la entrada $z$ se hace muy grande o muy pequeña, la salida se aproxima a 1 o a 0, respectivamente, pero nunca alcanza esos valores extremos. Esta característica es crucial para modelar probabilidades, porque asegura que las probabilidades predichas siempre estén dentro de un rango válido.

_Figura 4.10 Función logística._

La mejor forma de entender cómo funciona un algoritmo es con un ejemplo sencillo. El siguiente programa en Python usa el dataset Iris para predecir si una flor es de la especie Iris-virginica a partir de la longitud y el ancho del sépalo. El programa usa un algoritmo de regresión logística y grafica la frontera de decisión, así como las estimaciones de probabilidad que proporciona la función sigmoide.

> [!note]
> El dataset Iris es un dataset popular que contiene mediciones de distintas características de flores de Iris (longitud del sépalo, ancho del sépalo, longitud del pétalo, ancho del pétalo) de tres especies (Iris-setosa, Iris-versicolor, Iris-virginica). Este dataset está disponible en la función `load_iris` del módulo `sklearn.datasets`. Para más información, visita https://scikit-learn.org/1.5/auto_examples/datasets/plot_iris_dataset.html.

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn import datasets
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split

# Load the Iris dataset
iris = datasets.load_iris()
X = iris.data[:, :2]  # Using only sepal length and sepal width
y = (iris.target == 2).astype(int)  # Only interested in whether it is Iris-Virginica or not

# Split the data into training and test sets
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Train the Logistic Regression model
model = LogisticRegression()
model.fit(X_train, y_train)

# Generate grid of points to plot the sigmoid function
x_min, x_max = X[:, 0].min() - 1, X[:, 0].max() + 1
y_min, y_max = X[:, 1].min() - 1, X[:, 1].max() + 1
xx, yy = np.meshgrid(np.arange(x_min, x_max, 0.01), np.arange(y_min, y_max, 0.01))

# Calculate the probabilities
Z = model.predict_proba(np.c_[xx.ravel(), yy.ravel()])[:, 1]
Z = Z.reshape(xx.shape)

# Plot the decision boundary and sigmoid probabilities
contour = plt.contourf(xx, yy, Z, alpha=0.8, cmap=plt.cm.Greys, levels=np.linspace(0, 1, 11))

# Update scatter plot with different markers for each class
plt.scatter(X[y == 0, 0], X[y == 0, 1], c='white', edgecolors='k', marker='o', s=40, label='Not Iris-Virginica')
plt.scatter(X[y == 1, 0], X[y == 1, 1], c='white', edgecolors='k', marker='s', s=40, label='Iris-Virginica')

plt.colorbar(contour, label='Probability of Iris-Virginica')
plt.xlabel('Sepal length')
plt.ylabel('Sepal width')
plt.title('Logistic Regression Sigmoid Function Visualization')

# Add legend
plt.legend()

plt.show()
```

Veamos primero cómo funciona este programa:

- **Cargar y preprocesar los datos.** Se carga el dataset Iris y se seleccionan solo las características longitud y ancho del sépalo. La variable objetivo se convierte a formato binario, que indica si la flor es Iris-virginica.
- **Dividir los datos.** Los datos se dividen en conjuntos de entrenamiento y de prueba con `train_test_split`. Esto ayuda a evaluar el rendimiento del modelo con datos no vistos.
- **Entrenar el modelo de regresión logística.** Se instancia un modelo de regresión logística, que se entrena con los datos de entrenamiento mediante el método `fit`.
- **Generar una malla de puntos.** Se crea con `np.meshgrid` una malla de puntos que cubre el espacio de características (longitud y ancho del sépalo). Esta malla se usará para visualizar la frontera de decisión y las probabilidades.
- **Calcular las probabilidades.** El modelo entrenado se usa para predecir, con el método `predict_proba`, la probabilidad de que cada punto de la malla pertenezca a la clase Iris-virginica. Después, estas probabilidades se redimensionan para que coincidan con la malla.
- **Graficar los resultados.** La frontera de decisión y las probabilidades se grafican con `contourf`. Encima se grafican los puntos de datos con `scatter`. Se añade una barra de color que indica la escala de probabilidad y una leyenda que indica qué representan los puntos circulares y cuadrados (Not Iris-Virginica e Iris-Virginica).

Este programa demuestra la capacidad de la regresión logística para crear una frontera de decisión lineal y visualiza cómo varían las probabilidades del modelo en el espacio de características. La Figura 4.11 muestra la gráfica generada por este programa.

_Figura 4.11 Rangos de probabilidad calculados con regresión logística._

En la Figura 4.11, los puntos cuadrados indican muestras que realmente son Iris-virginica, mientras que los puntos circulares indican otras especies de Iris (Iris-setosa o Iris-versicolor). Observa cómo los rangos de probabilidad están separados entre sí por líneas. Esto se debe a que el algoritmo de regresión logística intenta correlacionar las características (en este caso, la longitud y el ancho del sépalo) mediante una ecuación lineal. Durante el entrenamiento, un algoritmo de regresión logística intenta encontrar una frontera de decisión lineal que separe las dos clases. Profundicemos un poco más.

Matemáticamente, el algoritmo de regresión logística ajusta una combinación lineal de las características de entrada al logaritmo de las probabilidades relativas (_log-odds_) del resultado binario.

En nuestro ejemplo, el resultado binario es el evento de que una muestra sea realmente una Iris-virginica, y $p$ denota la probabilidad de que ese evento ocurra _(fórmula reconstruida)_:

$$
\log\left(\frac{p}{1 - p}\right) = \beta_0 + \beta_1 x_1 + \beta_2 x_2
$$

La última parte de la ecuación se conoce como función _logit_ y devuelve el _log-odds_ de $p$, que después pasa por la función logística (sigmoide) para producir una probabilidad entre 0 y 1 _(fórmula reconstruida)_:

$$
p = \sigma(z) = \frac{1}{1 + e^{-(\beta_0 + \beta_1 x_1 + \beta_2 x_2)}}
$$

Como el logit $z$ es una combinación lineal de las características, la frontera de decisión que crea la regresión logística también será lineal. Esta frontera representa los puntos donde el modelo predice una probabilidad del 50 % (es decir, $p = 0.5$, o equivalentemente $z = 0$). Los puntos de datos de un lado de la línea tendrán probabilidades mayores al 50 %, y los del otro lado, menores al 50 %.

En la Figura 4.11, las líneas de contorno que ves representan los umbrales de probabilidad que genera el modelo de regresión logística. Cada línea corresponde a un valor de probabilidad específico. Como la ecuación subyacente con la que se calculan estas probabilidades es lineal, las líneas de contorno son paralelas y están espaciadas de manera uniforme. Estas líneas indican regiones con probabilidades similares y ayudan a visualizar cómo separa el modelo las distintas clases.

###### Máquinas de vectores de soporte (_Support Vector Machines_)

Las SVM son algoritmos versátiles de aprendizaje supervisado que pueden usarse tanto en tareas de clasificación como de regresión, y ofrecen soluciones potentes para una variedad de problemas de ML.

En clasificación, las SVM funcionan encontrando el hiperplano óptimo que separa las clases en el espacio de características con el margen máximo. Este margen es la distancia entre el hiperplano y los puntos de datos más cercanos de cualquiera de las clases, llamados vectores de soporte. Al maximizar este margen, las SVM buscan lograr una mejor generalización con datos no vistos. Las SVM pueden manejar problemas de clasificación tanto lineales como no lineales usando el truco del kernel (_kernel trick_) para transformar el espacio de características a dimensiones más altas, lo que permite trazar una frontera lineal en ese espacio transformado. Entre los kernels más usados están el lineal, el polinomial y el de función de base radial (RBF), cada uno adecuado para distintos tipos de distribuciones de datos.

En las tareas de regresión, las SVM se adaptan como regresión de vectores de soporte (SVR). El objetivo principal de la SVR es encontrar una función que se desvíe de los valores objetivo reales en no más de un margen especificado y que, al mismo tiempo, sea lo más plana posible. En otras palabras, la SVR intenta ajustar la mejor línea posible dentro de un margen de tolerancia ($\varepsilon$) para los valores objetivo. Este método usa un concepto similar al de los vectores de soporte en la clasificación con SVM: identifica los puntos de datos clave que definen la línea o curva de regresión óptima e ignora los puntos que caen dentro del margen. El resultado es un modelo robusto y flexible que puede manejar con eficacia los valores atípicos y el ruido. Al igual que la SVM para clasificación, la SVR también se beneficia de las funciones kernel para capturar relaciones complejas entre las características y la variable objetivo, lo que la convierte en una herramienta potente para el análisis de regresión.

###### Casos de uso (_Use Cases_)

La regresión lineal se usa mejor cuando hay una relación lineal clara entre la variable dependiente y una o más variables independientes. Es ideal para problemas de complejidad simple a moderada cuyo objetivo es predecir un resultado continuo, como predecir el precio de una casa a partir de los metros cuadrados y el número de recámaras. La regresión lineal es fácil de implementar e interpretar, lo que la hace adecuada para escenarios donde la transparencia del modelo es importante. Sin embargo, puede no ser la mejor opción para relaciones más complejas que impliquen no linealidad o interacciones entre variables. En esos casos, usar la regresión lineal puede llevar a un rendimiento pobre del modelo y a predicciones inexactas. Además, la regresión lineal es sensible a los valores atípicos y a la multicolinealidad, que pueden distorsionar los resultados. Por lo tanto, es menos apropiada para datasets con valores atípicos significativos, datos de alta dimensión o casos donde no se cumple el supuesto de una relación lineal. Para estos escenarios más complejos, otros algoritmos como la regresión polinomial, los árboles de decisión o el _gradient boosting_ podrían ser más eficaces.

La regresión logística es un método de referencia para los problemas de clasificación binaria, donde la variable de resultado es categórica con dos resultados posibles, como la detección de spam, el diagnóstico de una enfermedad (positivo/negativo) o la predicción de la baja de clientes (sí/no). Es adecuada cuando la relación entre las variables independientes y el _log-odds_ de la variable dependiente es lineal. La regresión logística es fácil de implementar e interpretar, y proporciona no solo una clasificación, sino también la probabilidad de pertenencia a la clase. Sin embargo, es menos eficaz con datasets complejos con relaciones no lineales o interacciones entre variables. En esos casos, su rendimiento puede quedar por detrás de técnicas más avanzadas como los árboles de decisión, las SVM o las redes neuronales, que pueden modelar patrones más intrincados. Además, la regresión logística supone la ausencia de multicolinealidad entre las variables independientes y requiere un dataset suficientemente grande para producir resultados confiables, lo que la hace menos apropiada para datos pequeños, muy correlacionados o de dimensión muy alta.

> [!note]
> La multicolinealidad se refiere a una situación de la modelización estadística, específicamente del análisis de regresión múltiple, en la que dos o más variables predictoras están altamente correlacionadas. Esta alta correlación significa que una variable predictora puede predecirse casi linealmente a partir de las otras. Esto crea redundancia y puede causar problemas para estimar de manera confiable los coeficientes del modelo de regresión. Una forma de abordar la multicolinealidad es con técnicas de regularización, que añaden al modelo de regresión lineal una penalización por haber aprendido coeficientes grandes que no generalizan bien a datos no vistos. La regularización se cubrirá en el capítulo 5.

Las SVM son modelos potentes de aprendizaje supervisado que se usan principalmente en tareas de clasificación, aunque también pueden aplicarse a problemas de regresión. Son particularmente eficaces con datasets de alta dimensión y en casos donde las clases están bien separadas por un margen claro. Las SVM resultan ventajosas cuando el número de características supera al número de muestras, porque en esos escenarios son menos propensas al sobreajuste.

> [!note]
> En ML, el sobreajuste (_overfitting_) ocurre cuando un modelo aprende demasiado bien los datos de entrenamiento, capturando el ruido y los valores atípicos además de los patrones subyacentes. El resultado es un modelo que rinde excepcionalmente bien con los datos de entrenamiento, pero mal con datos no vistos, porque no logra generalizar a ejemplos nuevos. El sobreajuste puede mitigarse con técnicas como la validación cruzada, la poda en los árboles de decisión, los métodos de regularización o la simplificación del modelo para que capture solo los patrones relevantes sin el ruido. Cubriremos el sobreajuste y las técnicas para abordarlo en el capítulo 5.

Sin embargo, las SVM no son ideales para datasets grandes por su intensidad computacional, que puede llevar a tiempos de entrenamiento más largos y a un mayor uso de memoria. Además, las SVM pueden tener dificultades con datasets ruidosos y clases que se solapan, donde podrían no rendir tan bien como algoritmos más flexibles como los _random forests_ o el _gradient boosting_. También requieren un ajuste cuidadoso de hiperparámetros como el tipo de kernel y los parámetros de regularización para lograr un rendimiento óptimo. En resumen, las SVM se usan mejor con datasets pequeños a medianos (hasta aproximadamente 10 000 muestras) y de alta dimensión, con clases claramente separadas, y son menos adecuadas para datasets grandes y ruidosos o para problemas que requieren entrenamiento y predicciones rápidos.

##### K vecinos más cercanos (_K-Nearest Neighbors_)

El algoritmo k-NN es un método de ML supervisado simple pero eficaz que se usa en problemas de clasificación y de regresión. La idea central de k-NN es que los puntos de datos similares tienden a estar cerca unos de otros en el espacio de características. Al hacer una predicción, k-NN identifica los $k$ vecinos más cercanos de un punto de datos dado y usa sus etiquetas para determinar la etiqueta del punto nuevo. En clasificación, el algoritmo asigna la etiqueta más común entre los vecinos más cercanos, mientras que en regresión promedia las etiquetas de los vecinos.

El valor de $k$ se fija antes de que comience el proceso de aprendizaje y representa el número de puntos de datos más cercanos (vecinos) que se consideran al hacer una predicción para un punto de datos nuevo. Es un hiperparámetro crucial del algoritmo k-NN. Un valor pequeño de $k$ (p. ej., $k = 1$) hace que el modelo sea muy sensible al ruido de los datos, porque solo considera al vecino más cercano. Por el contrario, un valor grande de $k$ (el valor de ejemplo del libro no se conserva en el `.txt`) suaviza las predicciones al promediar sobre más vecinos, lo que puede ser beneficioso con datos ruidosos, pero puede pasar por alto patrones locales. Elegir el valor correcto de $k$ implica equilibrar estos factores y suele hacerse con técnicas como la validación cruzada, que se verán en el capítulo 5. En esencia, $k$ determina el «vecindario» alrededor de un punto de datos que influye en su predicción.

A pesar de su simplicidad, k-NN puede ser computacionalmente costoso, sobre todo con datasets grandes, porque requiere calcular la distancia entre el punto de consulta y todos los demás puntos del dataset. Pueden usarse varias métricas de distancia para medir distancias, como la euclidiana, la de Manhattan y la de Minkowski. La elección de la métrica de distancia puede afectar significativamente el rendimiento del algoritmo y debe estar alineada con la naturaleza de los datos y el problema en cuestión.

El siguiente ejemplo demuestra visualmente cómo funciona k-NN en un problema de clasificación, usando una versión simplificada y bidimensional del dataset Iris:

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.neighbors import KNeighborsClassifier
from matplotlib.colors import ListedColormap

# Load the Iris dataset
iris = load_iris()
X = iris.data[:, :2]  # Use only the first two features for visualization
y = iris.target

# Split the data into training and testing sets
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Create and train the k-NN classifier
k = 3
knn = KNeighborsClassifier(n_neighbors=k)
knn.fit(X_train, y_train)

# Plot the decision boundary
x_min, x_max = X[:, 0].min() - 1, X[:, 0].max() + 1
y_min, y_max = X[:, 1].min() - 1, X[:, 1].max() + 1
xx, yy = np.meshgrid(np.arange(x_min, x_max, 0.1), np.arange(y_min, y_max, 0.1))
Z = knn.predict(np.c_[xx.ravel(), yy.ravel()])
Z = Z.reshape(xx.shape)

plt.figure(figsize=(8, 6))
plt.contourf(xx, yy, Z, alpha=0.3, cmap=ListedColormap(('gray', 'lightgray', 'darkgray')))

# Colors and markers for the classes
training_markers = ['o', 's', '^']
test_markers = ['o', 's', '^']
colors = ['black', 'black', 'black']
labels = ['Iris-setosa', 'Iris-versicolor', 'Iris-virginica']

# Plot training data points
for i, (color, marker) in enumerate(zip(colors, training_markers)):
    mask = y_train == i
    plt.scatter(X_train[mask, 0], X_train[mask, 1], c=color, label=f'Training {labels[i]}', marker=marker, edgecolor='k')

# Plot testing data points
for i, marker in enumerate(test_markers):
    mask = y_test == i
    plt.scatter(X_test[mask, 0], X_test[mask, 1], facecolors='none', edgecolors='k', label=f'Test {labels[i]}', marker=marker)

# Add legend for the classes
plt.xlabel(iris.feature_names[0])
plt.ylabel(iris.feature_names[1])
plt.title(f'k-NN Classification (k={k})')
plt.legend()
plt.show()
```

Esta es una explicación de cómo funciona el programa:

- **Carga de datos.** En este paso cargamos el dataset Iris y seleccionamos las dos primeras características para facilitar la visualización: la longitud y el ancho del sépalo.
- **División de datos.** Dividimos el dataset en conjuntos de entrenamiento y de prueba.
- **Entrenamiento del modelo.** Creamos un clasificador k-NN con $k = 3$ y lo ajustamos a los datos de entrenamiento.
- **Graficar la frontera de decisión.** La frontera de decisión de la gráfica es el resultado de la predicción de k-NN con $k = 3$ y se muestra visualmente en la Figura 4.12 mediante las tres regiones del espacio bidimensional de características de nuestro dataset Iris. La región del lado izquierdo de la Figura 4.12 indica predicciones de Iris-setosa, la región de abajo indica predicciones de Iris-versicolor y la región del lado derecho indica predicciones de Iris-virginica.

_Figura 4.12 Ajuste de k-NN al dataset Iris en 2D._

- **Graficar los puntos de entrenamiento y de prueba.** De forma similar, la forma de cada punto de datos corresponde directamente a su etiqueta de clase real (es decir, el valor verdadero de la clase: Iris-setosa, Iris-versicolor o Iris-virginica), lo que te permite distinguir visualmente entre predicciones y etiquetas de los distintos puntos de la gráfica. Los puntos de prueba se grafican con relleno blanco y contorno negro usando `facecolors='none'` y `edgecolors='k'`.

Observa en la Figura 4.12 cómo algunos puntos de entrenamiento de Iris-virginica están ubicados en la zona inferior del espacio de características (es decir, la zona de Iris-versicolor), lo que muestra cómo nuestro algoritmo k-NN (con $k = 3$) clasificó por error estos puntos de Iris-virginica como Iris-versicolor. Como resultado, este modelo no tuvo un buen desempeño. Se darán más detalles en el capítulo 5, cuando aprendas a evaluar el rendimiento de un modelo ajustando el valor de sus hiperparámetros, como $k$.

Para usar el algoritmo k-NN en Amazon SageMaker, primero necesitas preparar tu dataset y subirlo a una ubicación de almacenamiento adecuada (p. ej., Amazon S3, Amazon EFS o Amazon FSx for Lustre). Después puedes configurar un _estimator_ con el algoritmo k-NN especificando parámetros como el número de vecinos $k$ y la métrica de distancia. Tras configurar el _estimator_ con los hiperparámetros necesarios y las rutas de los datos de entrada, inicias el trabajo de entrenamiento, que Amazon SageMaker gestionará, incluidos el aprovisionamiento y el escalado de la infraestructura necesaria. Una vez completado el entrenamiento, el modelo puede desplegarse como un endpoint en Amazon SageMaker para hacer predicciones en tiempo real. Este enfoque integrado aprovecha la infraestructura robusta de Amazon SageMaker para entrenar y desplegar modelos k-NN de forma eficiente.

Este párrafo describe un ciclo que se repite, con otros nombres, en todos los algoritmos integrados del capítulo, así que conviene verlo una vez con detalle:

1. **El _estimator_** es un objeto del SDK de SageMaker para Python que describe un trabajo de entrenamiento: qué imagen de contenedor usar, con qué rol de IAM, en qué tipo y número de instancias, qué hiperparámetros pasar y en qué carpeta de S3 dejar el resultado. Crearlo no lanza nada ni cuesta nada.
2. **El trabajo de entrenamiento** (_training job_) empieza cuando llamas a `fit()`. SageMaker **aprovisiona** las instancias (es decir, las reserva y las arranca en su propia infraestructura), descarga la imagen desde ECR, copia o transmite los datos de cada canal desde S3, ejecuta el algoritmo y, al terminar, empaqueta el modelo en un archivo `model.tar.gz` que sube a la ruta de salida de S3. Después **apaga las instancias automáticamente**. Se cobra por segundo de instancia mientras dura el trabajo, y los registros quedan en CloudWatch Logs.
3. **El endpoint** se crea con `deploy()`. A partir de `model.tar.gz`, SageMaker arranca nuevas instancias con el contenedor de inferencia del algoritmo y publica un endpoint HTTPS. A diferencia del trabajo de entrenamiento, **el endpoint no se apaga solo**: se cobra hasta que lo borras.

> [!warning] Nota de precisión: hiperparámetros de k-NN en SageMaker (verificado el 25-09-2026)
> En el k-NN integrado son obligatorios cuatro hiperparámetros: `feature_dim`, `k`, `predictor_type` (`classifier` o `regressor`) y `sample_size`, que es el número de puntos que el algoritmo toma del dataset de entrenamiento para construir su índice. La métrica de distancia se elige con `index_metric`, que solo admite `L2` (euclidiana, el valor por defecto), `INNER_PRODUCT` y `COSINE`: las distancias de Manhattan y Minkowski que menciona el texto no están disponibles en el algoritmo integrado. Internamente, el k-NN de SageMaker construye un índice con la biblioteca FAISS (hiperparámetro `index_type`), lo que acelera la búsqueda de vecinos en datasets grandes.

###### Casos de uso (_Use Cases_)

k-NN es más adecuado para problemas de clasificación y de regresión cuando el dataset es relativamente pequeño (menos de 1 000 muestras) o de tamaño moderado (entre 1 000 y aproximadamente 10 000 muestras) y las fronteras de decisión no son demasiado complejas.

Es particularmente eficaz cuando los datos tienen una proximidad clara e intuitiva en el espacio de características que da sentido al concepto de «cercanía». k-NN también debe considerarse cuando el costo de la interpretabilidad del modelo es alto. Esto se debe a que k-NN se considera un algoritmo inherentemente muy interpretable, lo que significa que puedes entender fácilmente cómo hace sus predicciones a partir de la proximidad de los puntos en el espacio de características; por eso es una buena elección cuando la explicabilidad es una preocupación principal.

Sin embargo, k-NN es menos adecuado para datasets grandes por su alto costo computacional y su uso de memoria. Rinde mal con datos de alta dimensión (la «maldición» de la dimensionalidad), porque la métrica de distancia pierde significado, y puede tener dificultades con datos ruidosos si no se ajusta con cuidado. Además, k-NN requiere un preprocesamiento significativo, como el escalado de características, para funcionar con eficacia. Si el dataset contiene muchas características irrelevantes o muy correlacionadas, el rendimiento de k-NN puede degradarse, y en esos casos es necesario considerar algoritmos alternativos como las SVM o los _random forests_.

##### Árboles de decisión (Random Forest y XGBoost) (_Decision Trees (Random Forest and XGBoost)_)

Los algoritmos de árboles de decisión están entre los métodos de ML más populares y usados, tanto para tareas de clasificación como de regresión. Su naturaleza intuitiva y su capacidad de manejar distintos tipos de datos los convierten en una opción de referencia para muchos científicos de datos e ingenieros de ML.

Un árbol de decisión es un modelo de decisiones con forma de árbol, formado por nodos, ramas y hojas. El nodo raíz es el punto de partida y representa el dataset completo; los nodos internos son puntos donde los datos se dividen según ciertas condiciones, y las hojas son puntos terminales que representan los resultados o predicciones finales. Las ramas son caminos que representan los resultados de las decisiones.

Los árboles de decisión operan particionando recursivamente el dataset en subconjuntos según los valores de las características, lo que da como resultado una estructura de árbol. El objetivo es crear ramas de modo que los datos dentro de cada subconjunto (nodo) sean lo más homogéneos posible. La homogeneidad significa que cada subconjunto de datos (o nodo) resultante de una división debe contener puntos de datos lo más parecidos posible entre sí. En última instancia, buscas una alta pureza dentro de cada nodo, que indique que los puntos de datos de cada nodo pertenecen a la misma clase (en tareas de clasificación) o tienen valores similares (en tareas de regresión).

Este proceso implica varios pasos clave:

- **Selección de características.** El algoritmo selecciona la mejor característica para dividir los datos en cada nodo. La selección se basa en métricas como la impureza de Gini, la ganancia de información (entropía) y la reducción de la varianza, según el tipo de tarea (clasificación o regresión). La fórmula para calcular la varianza es la siguiente _(fórmula reconstruida)_:

  $$
  \mathrm{Var} = \frac{1}{n} \sum_{i=1}^{n} \left(y_i - \bar{y}\right)^2
  $$

- **División de los datos.** El dataset se divide en subconjuntos según la característica seleccionada. Cada subconjunto se convierte entonces en un nuevo nodo del árbol, y el proceso se repite recursivamente.
- **Criterios de parada.** La división recursiva se detiene cuando se cumple una condición predefinida. Puede ser una profundidad máxima del árbol, un número mínimo de muestras por nodo o que ya no sea posible mejorar la pureza.
- **Poda.** Para prevenir el sobreajuste, los árboles de decisión pueden podarse. La poda consiste en eliminar las ramas que tienen poca importancia y no contribuyen al rendimiento del modelo. Puede hacerse con técnicas como la poda por costo-complejidad.

Aunque los árboles de decisión son interpretables y versátiles por su capacidad de capturar relaciones complejas y no lineales en los datos, también tienen la desventaja de ser muy sensibles al dataset de entrenamiento. Como resultado, el algoritmo podría generar predicciones completamente distintas ante el cambio de un solo punto de datos. Para abordar esta limitación se usan métodos de ensamble como el _random forest_ y el _gradient boosting_. Los _random forests_ combinan múltiples árboles de decisión, cada uno entrenado en paralelo con distintos subconjuntos de los datos, para mejorar la exactitud y la robustez, mientras que el _gradient boosting_ construye los árboles de forma secuencial, y cada árbol corrige los errores de los anteriores.

###### Random Forest

Con los _random forests_, se eligen aleatoriamente varios puntos de datos del dataset de entrenamiento para crear un dataset nuevo. Para cada punto de datos, se seleccionan unas pocas características para construir el árbol de decisión del nuevo dataset. Este proceso se conoce como _bootstrapping_. El algoritmo de _random forest_ implica usar varios árboles de decisión (de ahí el nombre, porque un bosque está formado por muchos árboles), y cada árbol se entrena con un subconjunto aleatorio de los datos, contribuyendo a una predicción agregada, más exacta y robusta, mediante la votación de la clase más popular en las tareas de clasificación o el promedio de las predicciones en las tareas de regresión.

Los algoritmos de _random forest_ tienen la ventaja de entrenar múltiples árboles de decisión en paralelo, lo que da tiempos de entrenamiento significativamente más rápidos. Sin embargo, como cada árbol de decisión se genera de forma independiente, el modelo puede no optimizar del todo los errores cometidos por los árboles individuales, y perderse potencialmente los beneficios del aprendizaje secuencial y de los ajustes basados en gradientes que ofrecen algoritmos más avanzados como XGBoost.

> [!warning] Nota de precisión: SageMaker no tiene un Random Forest integrado
> El título de la sección y los «Puntos esenciales para el examen» presentan Random Forest como un algoritmo integrado de SageMaker, pero **no existe un algoritmo integrado de Random Forest** (verificado en la lista oficial el 25-09-2026). No lo confundas con **Random Cut Forest** (RCF), que sí es integrado, pero es un algoritmo **no supervisado de detección de anomalías** que se verá más adelante. Para entrenar un _random forest_ en SageMaker, lo habitual es el **modo script** con el contenedor de scikit-learn que mantiene AWS: escribes un script que usa `RandomForestClassifier`, y SageMaker lo ejecuta como trabajo de entrenamiento. Los algoritmos integrados de árboles son XGBoost, LightGBM y CatBoost (estos dos últimos no aparecen en el libro), además de AutoGluon-Tabular, que combina varios modelos.

###### XGBoost

El algoritmo XGBoost (_extreme gradient boosting_), conocido por su eficiencia y rendimiento, es un algoritmo de ML potente basado en árboles. El objetivo principal de entrenar un modelo XGBoost es construir un ensamble de árboles de decisión que mejoren secuencialmente la exactitud de las predicciones corrigiendo los errores de los árboles anteriores. Un resultado clave de este proceso de entrenamiento es la determinación de las características más importantes del dataset. La importancia de las características es crucial porque ayuda a entender qué atributos de los datos contribuyen más al poder predictivo del modelo. Esta comprensión puede orientar la selección de características y el ajuste del modelo, además de aportar información sobre los patrones subyacentes de los datos.

Al comparar XGBoost con el _random forest_, ambos algoritmos son métodos de ensamble que usan árboles de decisión, pero difieren significativamente en su enfoque y rendimiento. Como se ilustra en la Figura 4.13, el _random forest_ construye múltiples árboles de decisión independientes en paralelo y agrega sus predicciones, lo que ayuda a reducir la varianza y mejorar la robustez. Es particularmente eficaz para manejar datasets grandes con alta varianza, pero no siempre captura patrones complejos en los datos con tanta eficacia como los métodos de _gradient boosting_.

_Figura 4.13 Comparación entre random forest y XGBoost._

Por otro lado, XGBoost construye los árboles de decisión de forma secuencial, y cada árbol busca corregir los errores de los anteriores. Este proceso secuencial de _boosting_ ayuda a XGBoost a reducir tanto el sesgo como la varianza, lo que da modelos más exactos, especialmente con datasets complejos. Además, las técnicas avanzadas de optimización y regularización de XGBoost lo hacen más eficiente y menos propenso al sobreajuste que el _random forest_.

El código de Python que se presenta demuestra cómo entrenar un modelo XGBoost con el dataset Iris. Muestra los pasos para preparar los datos, entrenar el modelo y evaluar la importancia de las características. Al graficar la importancia de las características, el código revela qué características influyen más en las predicciones.

```python
import numpy as np
import matplotlib.pyplot as plt
import xgboost as xgb
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score

# Load the Iris dataset
iris = load_iris()
X = iris.data
y = iris.target

# Split the data into training and testing sets
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Create a DMatrix, the internal data structure for XGBoost
dtrain = xgb.DMatrix(X_train, label=y_train)
dtest = xgb.DMatrix(X_test, label=y_test)

# Define the parameters for the XGBoost model
params = {
    'max_depth': 3,
    'eta': 0.1,
    'objective': 'multi:softprob',  # Multiclass classification
    'num_class': 3
}

# Train the model
num_round = 50
bst = xgb.train(params, dtrain, num_round)

# Prepare new data points for prediction
new_data = np.array([
    [5.1, 3.5, 1.4, 0.2],  # Example data point 1
    [6.2, 3.4, 5.4, 2.3]   # Example data point 2
])

# Create a DMatrix for the new data
dnew = xgb.DMatrix(new_data)

# Make predictions
preds = bst.predict(dnew)
predictions = [np.argmax(pred) for pred in preds]

# Define the class names
class_names = {0: 'Iris-setosa', 1: 'Iris-versicolor', 2: 'Iris-virginica'}

# Print the predictions with class names and detailed output, including the actual data points
for i, (data_point, pred) in enumerate(zip(new_data, predictions)):
    class_label = class_names[pred]
    print(f'Prediction for datapoint {i+1} ({data_point}): {class_label}')

# Feature importance plot
fig, ax = plt.subplots(figsize=(10, 6))
xgb.plot_importance(bst, ax=ax)

# Add custom legend for features
features = ['sepal length (cm)', 'sepal width (cm)', 'petal length (cm)', 'petal width (cm)']
feature_labels = {f'f{i}': feature for i, feature in enumerate(features)}

# Create custom legend entries
handles = [plt.Line2D([0], [0], marker='o', color='w', label=f'{key}: {value}',
                      markersize=10, markerfacecolor='gray') for key, value in feature_labels.items()]
ax.legend(handles=handles, title='Feature Legend', loc='upper right')

plt.show()
```

En la Figura 4.14, la longitud del pétalo (f2) aparece como una característica significativa, lo que indica su fuerte correlación con las clases objetivo del dataset.

_Figura 4.14 Importancia de las características con XGBoost._

Al ejecutar el programa con las versiones del encabezado, los valores que grafica `plot_importance` (su métrica por defecto, `weight`) son f0 = 42, f1 = 46, **f2 = 241** y f3 = 110, lo que coincide con lo que describe el libro.

Después de identificar las características más importantes, el modelo XGBoost puede usarse para hacer predicciones con datos nuevos, no vistos. La Figura 4.15 muestra las predicciones de dos puntos de datos, cada uno formado por cuatro valores numéricos expresados en «cm» (centímetros) que corresponden a las características f0, f1, f2 y f3.

_Figura 4.15 Predicciones de XGBoost para dos puntos de datos._

Salida real del programa:

```text
Prediction for datapoint 1 ([5.1 3.5 1.4 0.2]): Iris-setosa
Prediction for datapoint 2 ([6.2 3.4 5.4 2.3]): Iris-virginica
```

El código también ilustra cómo preparar puntos de datos nuevos, ejecutar predicciones con el modelo entrenado e interpretar los resultados. Este proceso asegura que el modelo no solo sea robusto en cuanto a su rendimiento de entrenamiento, sino también eficaz en aplicaciones del mundo real donde se encuentran datos nuevos continuamente. Al aprovechar la información obtenida de la importancia de las características y la capacidad predictiva de XGBoost, los ingenieros de ML pueden desarrollar modelos de ML muy exactos y confiables para diversos problemas con datos estructurados.

Para usar el algoritmo XGBoost en Amazon SageMaker, primero necesitas preparar tu dataset y subirlo a una ubicación de almacenamiento adecuada (p. ej., Amazon S3, Amazon EFS o Amazon FSx for Lustre). Después configuras un _estimator_ de XGBoost especificando hiperparámetros esenciales como la función objetivo (p. ej., `binary:logistic` para clasificación binaria o `reg:squarederror` para regresión), el número de rondas de _boosting_ y la tasa de aprendizaje. Otros hiperparámetros importantes son `max_depth`, para controlar la profundidad de cada árbol; `subsample`, para especificar la fracción de muestras que se usan para entrenar cada árbol, y `colsample_bytree`, para determinar la fracción de características que se usan al construir cada árbol. La lista detallada de hiperparámetros está disponible en https://docs.aws.amazon.com/sagemaker/latest/dg/xgboost_hyperparameters.html.

Después de configurar el _estimator_ con estos hiperparámetros, inicias el trabajo de entrenamiento, y Amazon SageMaker gestiona la infraestructura necesaria para el proceso de entrenamiento. Una vez completado el entrenamiento, el modelo XGBoost entrenado puede desplegarse como un endpoint en Amazon SageMaker, lo que te permite hacer predicciones en tiempo real con datos nuevos, no vistos.

XGBoost es el algoritmo integrado con más opciones de configuración, y varias son preguntas típicas de examen:

- **Dos formas de usarlo.** Según la documentación, XGBoost puede usarse como **algoritmo integrado** (solo aportas datos e hiperparámetros, como en el resto de este capítulo) o como **framework** en modo script (aportas tu propio script de entrenamiento, que se ejecuta en el contenedor de XGBoost administrado por AWS). La segunda forma permite, por ejemplo, calcular métricas propias o preprocesar dentro del mismo trabajo.
- **Versión explícita.** Al obtener la imagen del contenedor hay que indicar una versión concreta soportada (por ejemplo, `1.7-1`); la documentación advierte que no se usen las etiquetas `latest` ni `1`.
- **Formato de los datos.** Acepta CSV (sin encabezado y con la etiqueta en la primera columna), LibSVM y Parquet, e indicar el tipo de contenido del canal es obligatorio en la práctica: sin él, el contenedor no asume CSV.
- **Tipo de instancia.** La documentación describe XGBoost en CPU como un algoritmo **limitado por memoria**, no por cómputo, y por eso recomienda instancias de propósito general (familia M, como `ml.m5`) antes que las optimizadas para cómputo (familia C), con memoria total suficiente para contener los datos de entrenamiento. Desde la versión 1.2-2 admite entrenamiento con GPU (familias G4dn y G5, entre otras; cada versión reciente dejó de admitir alguna familia antigua de GPU).
- **Entrenamiento distribuido.** Con varias instancias de CPU, los datos deben repartirse entre ellas: se dividen en varios archivos y el canal se configura con distribución `ShardedByS3Key`, para que cada instancia reciba aproximadamente 1/n de los archivos.
- **Hiperparámetros obligatorios.** El número de rondas se llama `num_round` y es obligatorio; `num_class` también lo es cuando el objetivo es multiclase (`multi:softmax` o `multi:softprob`).

###### Casos de uso (_Use Cases_)

Los árboles de decisión son una opción adecuada cuando la interpretabilidad es un aspecto crítico de tu problema de ML. Son fáciles de entender y visualizar, lo que los hace ideales en escenarios donde es esencial explicar las decisiones del modelo a las partes interesadas. Los árboles de decisión funcionan bien con datos numéricos y categóricos y no requieren un preprocesamiento extenso. Sin embargo, son propensos al sobreajuste, especialmente con datasets complejos, lo que lleva a una generalización pobre con datos nuevos. Por eso son menos adecuados para tareas con datasets grandes y ruidosos o que requieren una alta exactitud predictiva.

Los _random forests_ deben considerarse cuando el objetivo es mejorar el rendimiento predictivo mitigando al mismo tiempo el problema de sobreajuste asociado a los árboles de decisión individuales. Al promediar los resultados de múltiples árboles, los _random forests_ ofrecen predicciones más robustas y exactas. Son particularmente eficaces para datasets grandes con alta varianza y cuando se necesitan conocimientos sobre la importancia de las características. Sin embargo, los _random forests_ pueden volverse computacionalmente intensivos con un gran número de árboles y modelos profundos, lo que los hace menos ideales para aplicaciones que requieren predicciones en tiempo real o entornos con recursos computacionales limitados.

XGBoost es el algoritmo de elección cuando la alta exactitud predictiva y la eficiencia son primordiales. Destaca en el manejo de datasets grandes e interacciones complejas entre características, gracias a su proceso secuencial de construcción de árboles y a sus técnicas avanzadas de regularización (la regularización es una técnica para abordar el sobreajuste y se cubrirá en detalle en el capítulo 5). XGBoost es particularmente útil en tareas competitivas de ML y en escenarios que requieren el mejor rendimiento posible del modelo. Sin embargo, requiere un ajuste cuidadoso de los hiperparámetros y puede ser más complejo de implementar que los árboles de decisión y los _random forests_. Además, la complejidad y las demandas computacionales de XGBoost podrían ser excesivas para tareas más simples, donde la interpretabilidad y un despliegue rápido son más críticos que alcanzar la máxima exactitud.

#### Recomendación (_Recommendation_)

Los sistemas de recomendación son fundamentales para ofrecer experiencias personalizadas en diversas industrias. Un enfoque eficaz dentro de Amazon SageMaker es el algoritmo de máquinas de factorización (_factorization machines_), que destaca en el manejo de los datos dispersos de alta dimensión comunes en las tareas de recomendación. Al aprender factores latentes que representan las interacciones entre usuarios e ítems, las máquinas de factorización pueden predecir con exactitud las preferencias de los usuarios. Un aspecto clave de estos modelos son los _embeddings_, representaciones de características con valores continuos que capturan las relaciones semánticas dentro de los datos, lo que permite un procesamiento eficiente y un mejor rendimiento. El algoritmo Object2Vec de Amazon SageMaker amplía aún más esta capacidad al aprender _embeddings_ que pueden representar diversos tipos de objetos, facilitando sistemas de recomendación sofisticados que pueden entender y predecir con eficacia comportamientos y preferencias complejos de los usuarios.

Frente a Amazon Personalize, visto al principio del capítulo, la diferencia es de responsabilidad sobre la infraestructura. Personalize es un servicio de IA: AWS elige y opera los modelos, y tú cargas datos y consumes una API. Factorization Machines y Object2Vec son algoritmos: tú preparas los datos en el formato exacto que exigen, lanzas los trabajos de entrenamiento, eliges las instancias, despliegas y mantienes los endpoints, y construyes la canalización que los reentrena cuando llegan datos nuevos.

##### Máquinas de factorización (_Factorization Machines_)

El algoritmo Factorization Machines de Amazon SageMaker es una herramienta potente diseñada para manejar datasets dispersos de alta dimensión, comunes en los sistemas de recomendación y en las tareas de predicción de clics. Este algoritmo es particularmente eficaz para capturar interacciones entre características que los modelos lineales tradicionales podrían pasar por alto. Las máquinas de factorización amplían las capacidades de la factorización de matrices, permitiendo un rango más amplio de interacciones entre variables. Al descomponer las relaciones complejas de los datos en componentes más simples y manejables, el algoritmo puede aprender con eficiencia de grandes cantidades de datos y ofrecer predicciones exactas. La implementación de Amazon SageMaker admite tareas tanto de clasificación como de regresión, lo que la hace versátil para diversas aplicaciones.

Una de las ventajas clave del algoritmo Factorization Machines de Amazon SageMaker es su capacidad de manejar la dispersión con eficiencia. Los datasets dispersos, donde muchas características son cero o faltan, pueden plantear desafíos significativos para muchos algoritmos de ML. Sin embargo, las máquinas de factorización aprovechan factores latentes para modelar estas interacciones de forma compacta, reduciendo la dimensionalidad del problema y mejorando el rendimiento. Esto hace que el algoritmo sea particularmente adecuado para tareas como las recomendaciones de ítems a usuarios, donde la matriz de datos puede ser muy grande pero también muy dispersa.

Para usar el algoritmo Factorization Machines en Amazon SageMaker, primero necesitas preparar tu dataset y subirlo a una ubicación de almacenamiento adecuada (p. ej., Amazon S3, Amazon EFS o Amazon FSx for Lustre). Después configuras un _estimator_ para el algoritmo de máquinas de factorización especificando los hiperparámetros esenciales. Estos incluyen el `predictor_type` (`binary_classifier` o `regressor`), el número de factores para capturar las interacciones entre características, `feature_dim` (el número total de características), `mini_batch_size` y `epochs` para la duración del entrenamiento. Además, puedes fijar parámetros de regularización como `bias_lr` (tasa de aprendizaje del término de sesgo), `linear_lr` (tasa de aprendizaje del término lineal) y `factor_lr` (tasa de aprendizaje del término de factores) para controlar el sobreajuste. Para saber más sobre los hiperparámetros de las máquinas de factorización, visita https://docs.aws.amazon.com/sagemaker/latest/dg/fact-machines-hyperparameters.html.

> [!warning] Nota de precisión: configuración de Factorization Machines (verificado el 25-09-2026)
>
> - **Obligatorios:** `feature_dim`, `num_factors` (el «número de factores» del libro) y `predictor_type`. No existe un tipo multiclase: solo `binary_classifier` y `regressor`.
> - **Nombres de los hiperparámetros:** el de factores se llama `factors_lr`, no `factor_lr`. Además, `bias_lr`, `linear_lr` y `factors_lr` son tasas de aprendizaje; los hiperparámetros de regularización propiamente dichos son `bias_wd`, `linear_wd` y `factors_wd` (_weight decay_).
> - **Formato de entrada:** para entrenar, solo acepta **recordIO-protobuf** con tensores `Float32`; no acepta CSV. Es una pregunta de examen clásica, porque obliga a convertir los datos antes de entrenar.
> - **Instancias:** la documentación recomienda CPU, y GPU solo cuando los datos son densos.

Una vez configurado el _estimator_ con estos hiperparámetros, inicia el trabajo de entrenamiento, y Amazon SageMaker gestionará la infraestructura necesaria. Después del entrenamiento, despliega el modelo entrenado como un endpoint en Amazon SageMaker para hacer predicciones en tiempo real con datos nuevos.

###### Casos de uso (_Use Cases_)

Los principales casos de uso de las máquinas de factorización de Amazon SageMaker incluyen los sistemas de recomendación y el modelado predictivo en el comercio electrónico. Por ejemplo, pueden usarse para predecir las preferencias de los usuarios en el comercio minorista en línea analizando el comportamiento de compra pasado y las interacciones con los productos. Además, las máquinas de factorización son muy eficaces en la predicción de la tasa de clics (CTR, _click-through rate_) para la publicidad en línea, donde entender las interacciones de los usuarios con los anuncios puede influir significativamente en la segmentación y la personalización de los anuncios. En general, la versatilidad y la eficiencia del algoritmo de máquinas de factorización lo convierten en una herramienta valiosa para cualquier escenario que implique datasets dispersos a gran escala.

##### Object2Vec

Object2Vec de Amazon SageMaker es un algoritmo versátil de _embeddings_ neuronales, diseñado para generar _embeddings_ densos de baja dimensión a partir de datos de alta dimensión. Estos _embeddings_ capturan las relaciones semánticas entre objetos, lo que los hace útiles para tareas como la búsqueda de vecinos más cercanos, el clustering y la representación de características en tareas posteriores de aprendizaje supervisado. Object2Vec generaliza la conocida técnica Word2Vec, optimizada para varios tipos de datos estructurados y no estructurados.

Para configurar los hiperparámetros de Object2Vec en Amazon SageMaker, necesitas especificar varios parámetros clave. Estos incluyen `enc0_max_seq_len` (longitud máxima de secuencia del codificador enc0), `enc0_vocab_size` (tamaño del vocabulario de tokens de enc0), `dropout` (probabilidad de _dropout_ de las capas de la red), `early_stopping_patience` (número de épocas consecutivas sin mejora antes de la detención temprana) y `enc_dim` (dimensión de la capa de _embedding_ de salida). Además, puedes personalizar la lista de comparadores (`comparator_list`) para definir cómo se comparan los _embeddings_, y fijar la tasa de aprendizaje (`learning_rate`), el tamaño de minilote (`mini_batch_size`) y el tipo de optimizador (`optimizer`). Estos hiperparámetros te permiten ajustar el algoritmo a tu caso de uso específico, asegurando un rendimiento y una exactitud óptimos. Para saber más sobre los hiperparámetros de Object2Vec, visita https://docs.aws.amazon.com/sagemaker/latest/dg/object2vec-hyperparameters.html.

De esa lista, según la documentación vigente, solo `enc0_max_seq_len` y `enc0_vocab_size` son obligatorios; el resto tiene valores por defecto. Del lado de la infraestructura, Object2Vec se entrena con datos en **JSON Lines** (un objeto JSON por línea, donde cada registro es un par de entradas `in0` e `in1` expresadas como listas de identificadores enteros de tokens, más una etiqueta que indica su relación), solo admite el modo File y entrena en **una sola instancia**, de CPU o de GPU.

###### Casos de uso (_Use Cases_)

Object2Vec de Amazon SageMaker es ideal para escenarios en los que necesitas generar _embeddings_ de alta calidad a partir de datos complejos de alta dimensión para capturar relaciones semánticas. Es particularmente útil en sistemas de recomendación, búsquedas de similitud y clustering, y como características de entrada para tareas posteriores de aprendizaje supervisado. Sin embargo, Object2Vec puede no ser la mejor opción si tu objetivo principal es realizar transformaciones lineales simples o si tu dataset no es disperso y no se beneficia de capturar interacciones intrincadas. Además, para aplicaciones que requieren entrenamiento en tiempo real o datasets de escala extremadamente grande con restricciones estrictas de latencia, otros algoritmos optimizados para esas tareas podrían ser más adecuados. En general, Object2Vec destaca en situaciones que requieren _embeddings_ robustos e interpretables para datos complejos, estructurados o no estructurados.

#### Pronóstico (_Forecasting_)

El pronóstico con Amazon SageMaker ofrece soluciones robustas para predecir valores futuros a partir de datos históricos, lo que permite a las empresas tomar decisiones basadas en datos. Amazon SageMaker proporciona DeepAR como algoritmo integrado para el pronóstico de series de tiempo. Veamos cómo funciona.

> [!warning] Estado del servicio administrado de pronóstico
> AWS tenía también un servicio de IA para pronósticos, **Amazon Forecast**, que no admite clientes nuevos desde el 29-07-2024. Para cuentas nuevas, las alternativas dentro de AWS son DeepAR en SageMaker AI y las funciones de pronóstico de series de tiempo de SageMaker Canvas.

##### DeepAR

El algoritmo DeepAR de Amazon SageMaker es una herramienta potente diseñada para el pronóstico de series de tiempo. Aprovecha redes neuronales recurrentes (RNN) para capturar patrones y dependencias temporales complejos dentro de los datos. A diferencia de los métodos estadísticos tradicionales, DeepAR puede manejar datasets a gran escala con múltiples series de tiempo, lo que lo hace particularmente eficaz en tareas de pronóstico que involucran productos o ubicaciones diversos. Al entrenar con series de tiempo relacionadas, el algoritmo puede aprender patrones compartidos y mejorar la exactitud de los pronósticos individuales, lo que es especialmente beneficioso en aplicaciones como el pronóstico de demanda, la gestión de inventarios y las predicciones financieras.

DeepAR está diseñado para ofrecer pronósticos probabilísticos. Esto significa que no solo predice una estimación puntual, sino que también cuantifica la incertidumbre generando un rango de valores futuros posibles. Matemáticamente hablando, DeepAR produce una distribución de probabilidad alrededor de un valor futuro predicho, en lugar de solo una estimación puntual, lo que permite a los usuarios entender la incertidumbre asociada a sus predicciones. La capacidad de DeepAR de producir pronósticos probabilísticos es crítica para los procesos de toma de decisiones que necesitan considerar varios resultados y sus probabilidades asociadas. Esto permite a las empresas gestionar mejor los riesgos y optimizar los recursos al entender todo el espectro de escenarios futuros posibles.

Al configurar DeepAR desde Amazon SageMaker, hay que considerar varios hiperparámetros clave. El hiperparámetro `epochs` determina el número de veces que el modelo iterará sobre los datos de entrenamiento, lo que influye en la duración del entrenamiento y en el rendimiento. El hiperparámetro `context_length` especifica el número de pasos de tiempo del pasado que el modelo usa para hacer pronósticos. El hiperparámetro `prediction_length` define el número de pasos de tiempo futuros que el modelo predecirá. Además, `num_layers` y `num_cells` controlan la profundidad y el tamaño de la RNN, respectivamente, lo que afecta la capacidad del modelo de aprender patrones complejos. Otros hiperparámetros importantes son `mini_batch_size`, para la eficiencia del entrenamiento, y `learning_rate`, para ajustar los pesos del modelo durante el entrenamiento. Ajustar correctamente estos hiperparámetros es esencial para optimizar el rendimiento y la exactitud del modelo en tareas de pronóstico específicas. Para ver la lista completa de hiperparámetros de DeepAR, visita https://docs.aws.amazon.com/sagemaker/latest/dg/deepar_hyperparameters.html.

> [!warning] Nota de precisión: configuración de DeepAR (verificado el 25-09-2026)
>
> - **Obligatorios:** `context_length`, `epochs`, `prediction_length` y **`time_freq`**, la granularidad de la serie (`5min`, `H`, `D`, `W`, `M`), que el libro no menciona. `num_layers`, `num_cells`, `mini_batch_size` y `learning_rate` son opcionales, con valores por defecto.
> - **Formato de los datos:** JSON Lines (opcionalmente comprimido con gzip) o Parquet, con un registro por serie de tiempo. Cada registro lleva `start` (la marca de tiempo inicial, sin zona horaria), `target` (el arreglo de valores) y, opcionalmente, `cat` (categorías de la serie) y `dynamic_feat` (series de covariables). El canal `train` es obligatorio y el canal `test` es opcional.
> - **Instancias:** entrena en CPU o GPU, en una o varias instancias; la documentación recomienda empezar con una sola instancia de CPU. **Para inferencia, DeepAR solo admite instancias de CPU.**
> - La documentación exige que el total de observaciones entre todas las series de entrenamiento sea de al menos 300.

###### Casos de uso (_Use Cases_)

Usa DeepAR para el pronóstico de series de tiempo cuando trabajes con datasets a gran escala formados por múltiples series de tiempo relacionadas, especialmente cuando necesites pronósticos probabilísticos que cuantifiquen la incertidumbre de las predicciones. Es particularmente eficaz en aplicaciones como el pronóstico de demanda, la gestión de inventarios y las predicciones financieras, donde capturar patrones y dependencias complejos es crítico. Sin embargo, evita usar DeepAR si tu dataset es pequeño o consiste en series de tiempo simples de una sola variable, donde los métodos estadísticos tradicionales (como ARIMA o el suavizamiento exponencial) pueden bastar. Además, si se requiere entrenamiento en tiempo real o predicciones de muy baja latencia, otros algoritmos optimizados para esas tareas podrían ser más apropiados.

### Algoritmos de ML no supervisado (_Unsupervised ML Algorithms_)

Si alguna vez has resuelto crucigramas, conoces la satisfacción de llenar los espacios en blanco y ver cómo las palabras encajan. Ahora imagina una versión más desafiante llamada «crucigramas sin diagrama», en la que no solo tienes que resolver las pistas, sino también descubrir la estructura de la cuadrícula. Este intrigante pasatiempo es una metáfora perfecta para entender el ML no supervisado. Igual que en los crucigramas sin diagrama, donde quien los resuelve debe descubrir la cuadrícula oculta y las palabras, los algoritmos de ML no supervisado trabajan sin etiquetas predefinidas y buscan patrones y estructuras directamente en los datos.

En el dominio del ML no supervisado hay varios algoritmos clave, cada uno con un propósito distinto. Una técnica popular es el clustering, con K-means como ejemplo clásico. K-means agrupa los puntos de datos en clusters según su similitud, de forma parecida a ordenar las piezas de un rompecabezas en secciones coherentes. Otro método esencial es la reducción de dimensionalidad, ejemplificada por el análisis de componentes principales (PCA). PCA simplifica datasets complejos reduciendo el número de dimensiones, algo así como doblar un mapa grande para centrarse en una zona específica. El modelado de temas es otra aplicación excelente del ML no supervisado, con algoritmos como la asignación latente de Dirichlet (LDA) y el modelo neuronal de temas (NTM). Estos algoritmos descubren temas ocultos dentro de grandes datasets de texto, de forma similar a revelar la historia subyacente en un revoltijo de palabras. Además, la detección de anomalías es una aplicación crítica del ML no supervisado que se usa para identificar valores atípicos o patrones inusuales en los datos. Algoritmos como Random Cut Forest (RCF) e IP Insights destacan en este ámbito, detectando anomalías que podrían indicar fraude u otras actividades irregulares. Juntos, estos algoritmos muestran la versatilidad del ML no supervisado y ofrecen herramientas potentes para explorar y entender las estructuras ocultas de tus datos sin necesitar etiquetas ni guía predefinidas.

#### Clustering

En el ML no supervisado, se explora un dataset para descubrir patrones ocultos, agrupar puntos de datos similares en clusters e identificar estructuras intrínsecas sin etiquetas predefinidas, lo que aporta información valiosa sobre las características y relaciones subyacentes de los datos. El clustering, en particular, facilita la transición de los datos crudos a la información valiosa al organizar los datos en grupos significativos. Esta organización transforma puntos de datos aislados en información reveladora, que después puede interpretarse y analizarse para generar conocimiento. Al entender estos clusters, se pueden obtener conocimiento y conclusiones accionables que impulsan la toma de decisiones informada y la planificación estratégica.

##### Clustering K-means (_K-Means Clustering_)

El algoritmo K-means es un enfoque popular de ML no supervisado que se usa para agrupar datos en un número predefinido de clusters, $k$. El algoritmo funciona de forma iterativa asignando cada punto de datos a uno de los $k$ clusters según la similitud entre los puntos de datos.

En este contexto, la similitud suele definirse con una medida de distancia. La métrica de distancia más común es la distancia euclidiana, que calcula la distancia en línea recta entre dos puntos en un espacio multidimensional (nuestro espacio de características). El algoritmo minimiza la suma de las distancias al cuadrado de cada punto al centroide del cluster al que está asignado. Un centroide es el punto central de un cluster, calculado como la media de todos los puntos de ese cluster. Representa el centro de masa del cluster y se usa para asignar los puntos de datos a los clusters.

Un desafío común es determinar el número apropiado de clusters para un dataset dado. El método del codo (_elbow method_) es una heurística que se usa para estimar el número de clusters graficando la suma de cuadrados dentro de los clusters (WCSS, _within-cluster sum of squares_) frente al número de clusters. A medida que aumenta el número de clusters, la WCSS disminuye, pero hay un punto en el que la tasa de disminución se desacelera bruscamente, formando un «codo» en la gráfica. Este punto de codo sugiere un número óptimo de clusters, que equilibra el subajuste y el sobreajuste. Matemáticamente, la WCSS de un conjunto de clusters se calcula como la suma de las distancias al cuadrado entre cada punto de datos y el centroide de su cluster correspondiente.

Dado un conjunto de clusters $C_1, C_2, \dots, C_k$, con centroides $\mu_1, \mu_2, \dots, \mu_k$, la WCSS se calcula así _(fórmula reconstruida)_:

$$
\mathrm{WCSS} = \sum_{i=1}^{k} \sum_{\mathbf{x} \in C_i} \left\lVert \mathbf{x} - \mu_i \right\rVert^2
$$

donde $k$ es el número de clusters, $C_i$ es el conjunto de puntos de datos del $i$-ésimo cluster, $\mathbf{x}$ es un punto de datos del $i$-ésimo cluster, $\mu_i$ es el centroide del $i$-ésimo cluster y $\lVert \mathbf{x} - \mu_i \rVert^2$ es la distancia euclidiana al cuadrado entre el punto de datos $\mathbf{x}$ y el centroide $\mu_i$.

###### Visualización del método del codo (_Elbow Method Visualization_)

Para ayudarte a entender el método del codo, el siguiente programa en Python grafica la función del codo para el dataset Iris usando la clase `KMeans` del módulo `sklearn.cluster`:

```python
import os
import matplotlib.pyplot as plt
from sklearn.cluster import KMeans
from sklearn.datasets import load_iris
from sklearn.preprocessing import StandardScaler

# Create images directory if it doesn't exist
images_dir = './images'
os.makedirs(images_dir, exist_ok=True)

# Load the Iris dataset
iris = load_iris()
data = iris.data

# Standardize the data
scaler = StandardScaler()
scaled_data = scaler.fit_transform(data)

# List to hold the within-cluster sum of squares (WCSS)
wcss = []

# Run K-Means for 1 to 10 clusters
for k in range(1, 11):
    kmeans = KMeans(n_clusters=k, random_state=0)
    kmeans.fit(scaled_data)
    wcss.append(kmeans.inertia_)

# Plot the WCSS values to visualize the elbow method
plt.figure(figsize=(8, 5))
plt.plot(range(1, 11), wcss, marker='o')
plt.title('Elbow Method for Determining Optimal Number of Clusters')
plt.xlabel('Number of Clusters')
plt.ylabel('Within-Cluster Sum of Squares (WCSS)')
plt.grid(True)

# Save the plot to the images directory
plot_file = os.path.join(images_dir, 'elbow_method_plot.png')
plt.savefig(plot_file)
print(f"Elbow method plot has been saved to '{plot_file}'")
```

La mayoría de las instrucciones de este programa son sencillas y se explican solas, pero ¿qué ocurre dentro del bucle?

- **Inicialización del modelo.** `KMeans(n_clusters=k, random_state=0)` inicializa el modelo K-means con `k` clusters y un estado aleatorio fijo para la reproducibilidad.
- **Entrenamiento del modelo.** `kmeans.fit(scaled_data)` entrena el modelo K-means con `scaled_data`. Durante este proceso se realizan los siguientes pasos:
  - _Inicialización de los centroides:_ el algoritmo asigna inicialmente posiciones aleatorias a los centroides de los clusters.
  - _Asignación de los puntos de datos:_ cada punto de datos se asigna al centroide más cercano según la distancia euclidiana.
  - _Actualización de los centroides:_ los centroides se recalculan como la media de todos los puntos de datos asignados a ese cluster.
  - _Reiteración:_ el proceso de asignación de puntos y actualización de centroides se repite hasta la convergencia (es decir, hasta que los centroides ya no cambian significativamente).
- **Cálculo de la WCSS.** Después de entrenar el modelo, se añade `kmeans.inertia_` a la lista `wcss`. `kmeans.inertia_` es un atributo de la clase `KMeans` (del módulo `sklearn.cluster`) que representa la WCSS. Este atributo mide la compacidad de los clusters: cuanto menor es la WCSS, más cerca están los puntos de un cluster de su centroide.

Al ejecutarse en Amazon SageMaker Studio, este código genera la gráfica del codo y la guarda como `elbow_method_plot.png` en la carpeta `./images` del EFS montado en tu instancia de Amazon SageMaker Studio.

> [!note]
> Las instancias de Amazon SageMaker Studio vienen con EFS montado de forma predeterminada, de modo que cualquier archivo guardado en directorios locales de la instancia se almacena en el EFS conectado. Esto asegura que la gráfica generada quede disponible y persista entre distintas sesiones de Amazon SageMaker Studio.

Como se explicó en la sección de Bedrock, en la versión actual de Studio ese almacenamiento persistente es el volumen EBS del espacio, no EFS. Además, `./images` es una ruta **relativa**: se crea dentro del directorio desde el que se ejecuta el programa, que en Studio suele ser una carpeta bajo `/home/sagemaker-user`, y por eso persiste.

La gráfica del codo se muestra en la Figura 4.16 e ilustra de forma eficaz el número óptimo de clusters, destacando el punto donde la WCSS empieza a disminuir más lentamente y forma una figura de «codo». Esta señal visual ayuda a determinar el número apropiado de clusters para el dataset.

_Figura 4.16 Función del codo._

El programa solo imprime el mensaje de confirmación (`Elbow method plot has been saved to './images/elbow_method_plot.png'`). Los valores de WCSS que grafica, obtenidos con el mismo código y las versiones del encabezado, son, para $k$ = 1 a 10: 600.0, 222.4, 139.8, 114.1, 90.8, 81.5, 72.8, 65.2, 57.3 y 47.3.

Según los datos de la Figura 4.16, $k = 3$ es el número óptimo de clusters _(valor reconstruido; en el `.txt` aparece como hueco, pero el resto de la sección usa tres clusters)_. Esto significa que añadir más clusters después de $k = 3$ no aporta una mejora sustancial en la compacidad de los clusters. En el caso del dataset Iris, este clustering óptimo captura de forma eficaz la agrupación natural de las distintas especies, equilibrando simplicidad y exactitud sin sobreajustar.

Pongamos ahora a trabajar el algoritmo K-means y veamos con qué eficacia se agrupan nuestros puntos de datos en tres clusters. Ten en cuenta que el dataset Iris es un espacio de características de cuatro dimensiones con 150 puntos de datos. Como resultado, una gráfica que ilustre el clustering no puede representarse visualmente a menos que se reduzca la dimensionalidad de cuatro a tres (o a dos). En la próxima sección aprenderemos otra técnica de ML no supervisado pensada para reducir la dimensionalidad de un dataset. Para ofrecer una visión completa manteniendo la visualización intuitiva y accesible, vamos a ejecutar el algoritmo K-means con una versión simplificada y tridimensional del dataset Iris.

> [!note]
> En la próxima sección, cuando veamos el algoritmo PCA, cubriremos una forma más exacta de agrupar el dataset Iris. PCA está diseñado para reducir la dimensionalidad de tu dataset de características manteniendo la mayor varianza posible de los datos.

El siguiente programa en Python realiza el clustering K-means con tres características del dataset Iris (longitud del sépalo, longitud del pétalo y ancho del pétalo) y guarda la gráfica con los clusters y los centroides como `kmeans_iris_3d_plot.png` en la carpeta `./ch04/images` del EFS montado en tu instancia de Amazon SageMaker Studio:

```python
import os
import numpy as np
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D
from sklearn.datasets import load_iris
from sklearn.preprocessing import StandardScaler
from sklearn.cluster import KMeans

# Define the output directory and file path
output_dir = './ch04/images'
output_file = os.path.join(output_dir, 'kmeans_iris_3d_plot.png')

# Create the output directory if it doesn't exist
os.makedirs(output_dir, exist_ok=True)

# Load Iris dataset
iris = load_iris()
data = iris.data[:, [0, 2, 3]]
labels = iris.target
feature_names = ['sepal length (cm)', 'petal length (cm)', 'petal width (cm)']

# Standardize the features
scaler = StandardScaler()
scaled_data = scaler.fit_transform(data)

# Perform K-Means clustering
kmeans = KMeans(n_clusters=3, random_state=42)
kmeans.fit(scaled_data)
clusters = kmeans.labels_

# Plot the clustering results in 3D
fig = plt.figure(figsize=(10, 8))
ax = fig.add_subplot(111, projection='3d')

# Define colors and markers
colors = ['blue', 'orange', 'purple']
markers = ['o', 's', 'D']

# Plot each cluster with different color and marker
for cluster in np.unique(clusters):
    ax.scatter(scaled_data[clusters == cluster, 0],
               scaled_data[clusters == cluster, 1],
               scaled_data[clusters == cluster, 2],
               c=colors[cluster],
               marker=markers[cluster],
               label=f'Cluster {cluster+1}',
               edgecolor='black')

# Plot centroids with a star marker
centroids = kmeans.cluster_centers_
for i, centroid in enumerate(centroids):
    ax.scatter(centroid[0], centroid[1], centroid[2],
               s=300,
               c='yellow',
               marker='*',
               edgecolor='black',
               linewidths=2,
               label=f'Centroid {i+1}')

ax.set_xlabel('Standardized Sepal Length (cm)')
ax.set_ylabel('Standardized Petal Length (cm)')
ax.set_zlabel('Standardized Petal Width (cm)')
ax.legend()
ax.set_title('K-Means Clustering of Iris Dataset (3D)')

# Save the plot to the specified file path
plt.savefig(output_file)

# Display a message confirming the plot has been saved
print(f"Plot has been saved as '{output_file}'")
```

En el código presentado, la instrucción `clusters = kmeans.labels_` asigna las etiquetas calculadas a la variable `clusters`, que después se usa para colorear y agrupar los puntos de datos en la gráfica según su pertenencia a un cluster.

El atributo `kmeans.labels_` devuelve un arreglo de etiquetas enteras que indican el cluster al que pertenece cada punto de datos. Esto ocurre después de ajustar el algoritmo K-means al dataset de entrenamiento con la instrucción `kmeans.fit(scaled_data)`. En esencia, la instrucción `clusters = kmeans.labels_` asigna una etiqueta de cluster a cada punto de datos del dataset, mostrando en qué cluster quedó agrupado según los resultados del algoritmo de clustering.

> [!note]
> No confundas las etiquetas con las predicciones. Aunque la instrucción `clusters = kmeans.labels_` asigna las etiquetas de cluster a cada punto de datos después de entrenar el algoritmo K-means con la instrucción `kmeans.fit(scaled_data)`, no es en sí un predictor. Es una salida del algoritmo K-means que te dice a qué cluster pertenece cada punto de datos. No predice la pertenencia a clusters de puntos de datos nuevos, no vistos. Si quieres predecir el cluster de puntos de datos nuevos con un modelo K-means entrenado, necesitas usar el método `kmeans.predict()` de la clase `KMeans`. Se darán más detalles en el próximo capítulo.

En resumen, el método `kmeans.fit()` entrena el modelo K-means con los datos de entrenamiento, mientras que el atributo `kmeans.labels_` devuelve las etiquetas de cluster de los datos de entrenamiento, y el método `kmeans.predict()` predice las etiquetas de cluster de puntos de datos nuevos, no vistos.

Ahora que entiendes cómo funciona el código, puedes ver los tres clusters en la Figura 4.17. La gráfica tridimensional del clustering K-means del dataset Iris ilustra de forma eficaz la agrupación de los puntos de datos según tres características estandarizadas: longitud del sépalo, longitud del pétalo y ancho del pétalo. La gráfica usa formas distintas para diferenciar los tres clusters, destacando su separación espacial en el espacio de características. Los centroides de cada cluster se muestran de forma prominente con marcadores de estrella amarillos más grandes, lo que los hace fáciles de identificar. Esta visualización ofrece una representación clara e intuitiva de cómo el algoritmo K-means ha segmentado el dataset Iris en tres grupos distintos, reflejando el agrupamiento natural de las distintas especies de Iris.

_Figura 4.17 Clustering K-means en 3D del dataset Iris._

Para usar el algoritmo K-means integrado en Amazon SageMaker, puedes seguir estos pasos. Primero, configura una sesión de Amazon SageMaker y especifica el contenedor del algoritmo K-means. Después, crea un _estimator_ de K-means, especifica el número de clusters y ajusta el modelo a tus datos. Por último, despliega el _estimator_ entrenado en un endpoint para obtener inferencias. El siguiente fragmento muestra un ejemplo breve:

```python
import sagemaker
from sagemaker import get_execution_role
from sklearn.preprocessing import StandardScaler

# Set up SageMaker session and role
sagemaker_session = sagemaker.Session()
role = get_execution_role()

# Specify the K-Means algorithm container
container = sagemaker.image_uris.retrieve(
    'kmeans',
    sagemaker_session.boto_region_name
)

# Create KMeans estimator
kmeans = sagemaker.estimator.Estimator(
    container,
    role,
    instance_count=1,
    instance_type='ml.m4.xlarge',
    output_path='s3://your-bucket/path-to-output',
    sagemaker_session=sagemaker_session
)

# Specify the number of clusters
kmeans.set_hyperparameters(k=3)

# Fit the model on your data stored in S3
kmeans.fit({'train': 's3://your-bucket/path-to-train-data'})

# Once trained, the model can be deployed to an endpoint for inference
predictor = kmeans.deploy(
    initial_instance_count=1,
    instance_type='ml.m4.xlarge'
)

# Use the predictor to make predictions on new data
new_data = [[5.1, 3.5, 1.4, 0.2], [6.2, 3.4, 5.4, 2.3]]
scaler = StandardScaler()
scaled_new_data = scaler.transform(new_data) # Ensure new data is standardized

# Make predictions
predicted_clusters = predictor.predict(scaled_new_data)
print(predicted_clusters)
```

Este es el primer fragmento del capítulo que llama a SageMaker, así que conviene leerlo línea por línea en clave de infraestructura:

- **`sagemaker.Session()`** crea un objeto que guarda la región y las credenciales con las que se hablará con SageMaker y S3; también sabe cuál es tu bucket predeterminado.
- **`get_execution_role()`** devuelve el ARN del rol de IAM del entorno donde corre el código (el rol de ejecución del espacio de Studio o de la instancia de notebook). Fuera de SageMaker, por ejemplo en tu laptop, falla, y hay que escribir el ARN del rol a mano. Ese rol es el que asumirán el trabajo de entrenamiento y el endpoint, así que necesita permisos para leer los datos de S3 y escribir el modelo.
- **`image_uris.retrieve('kmeans', región)`** devuelve la dirección en ECR de la imagen del algoritmo K-means **en esa región**. La región importa porque SageMaker solo puede usar imágenes de la misma región donde corre el trabajo.
- **`ml.m4.xlarge`** es una instancia de propósito general de la familia M (4 vCPU y 16 GiB de memoria) de cuarta generación, hoy considerada de generación anterior; en código nuevo conviene usar una generación actual, como `ml.m5.xlarge`, y verificar su disponibilidad en la región.
- **`fit({'train': ...})`** lanza el trabajo de entrenamiento con un canal llamado `train` que apunta a una carpeta de S3; `output_path` es donde quedará `model.tar.gz`.
- **`deploy()`** crea el endpoint, que se cobra hasta que lo borres.

> [!warning] Nota de precisión: el fragmento no funciona tal cual
> Además de requerir el SDK v2 (ver el recuadro del inicio), el fragmento tiene cinco problemas. Los dos primeros se verificaron en la documentación del algoritmo y los tres últimos en el código fuente del SDK 2.257.6:
>
> 1. **Falta `feature_dim`.** En K-means son obligatorios `k` y `feature_dim`; sin el segundo, el trabajo de entrenamiento falla.
> 2. **No se declara el formato de los datos.** Al pasar una simple cadena `s3://...`, el canal no lleva tipo de contenido y K-means asume recordIO-protobuf. Si los datos son CSV, hay que declararlo con `TrainingInput(..., content_type='text/csv;label_size=0')`; el `label_size=0` indica que no hay columna de etiqueta.
> 3. **El endpoint recibiría bytes crudos.** El `Predictor` genérico que devuelve `deploy()` usa por defecto `IdentitySerializer`, que envía los datos sin convertir, y `BytesDeserializer`, que devuelve la respuesta como bytes. Para enviar una lista de listas o un arreglo de NumPy hay que indicar `serializer=CSVSerializer()` y, para leer la respuesta JSON, `deserializer=JSONDeserializer()`.
> 4. **El `StandardScaler` nuevo no está ajustado.** `StandardScaler().transform()` sin un `fit` previo lanza `NotFittedError`. Hay que reutilizar el mismo objeto ajustado con los datos de entrenamiento, que a su vez deben haberse estandarizado con él antes de subirlos a S3.
> 5. **Nunca se borra el endpoint.** El fragmento deja una instancia encendida y cobrando indefinidamente.
>
> Versión corregida (SDK v2; no se ejecutó contra AWS):
>
> ```python
> import sagemaker
> from sagemaker import get_execution_role
> from sagemaker.inputs import TrainingInput
> from sagemaker.serializers import CSVSerializer
> from sagemaker.deserializers import JSONDeserializer
>
> sagemaker_session = sagemaker.Session()
> role = get_execution_role()
> container = sagemaker.image_uris.retrieve('kmeans', sagemaker_session.boto_region_name)
>
> kmeans = sagemaker.estimator.Estimator(
>     container, role,
>     instance_count=1,
>     instance_type='ml.m5.xlarge',
>     output_path='s3://your-bucket/path-to-output',
>     sagemaker_session=sagemaker_session,
> )
> kmeans.set_hyperparameters(k=3, feature_dim=4)
>
> train_input = TrainingInput(
>     's3://your-bucket/path-to-train-data',   # CSV sin encabezado, ya estandarizado
>     content_type='text/csv;label_size=0',
> )
> kmeans.fit({'train': train_input})
>
> predictor = kmeans.deploy(
>     initial_instance_count=1,
>     instance_type='ml.m5.xlarge',
>     serializer=CSVSerializer(),
>     deserializer=JSONDeserializer(),
> )
>
> # scaler: el mismo StandardScaler ajustado con los datos de entrenamiento
> new_data = [[5.1, 3.5, 1.4, 0.2], [6.2, 3.4, 5.4, 2.3]]
> print(predictor.predict(scaler.transform(new_data)))
> # {'predictions': [{'closest_cluster': ..., 'distance_to_cluster': ...}, ...]}
>
> predictor.delete_endpoint()   # el endpoint se cobra mientras exista
> ```

###### Casos de uso (_Use Cases_)

K-means es más adecuado para escenarios en los que tienes una idea clara del número de clusters que esperas encontrar en tu dataset. Este algoritmo es muy eficaz para particionar datasets en grupos distintos y sin solapamiento, según la similitud de las características. Destaca en escenarios con clusters bien separados y esféricos en el espacio de características. Aplicaciones como la segmentación de clientes, el análisis de la canasta de mercado, la compresión de imágenes y la detección de anomalías son ideales para K-means, porque el algoritmo maneja eficientemente datasets grandes y escala bien con un número creciente de muestras. Cuando los datos de entrada son principalmente numéricos y están relativamente bien estructurados, K-means puede ofrecer agrupaciones reveladoras y accionables.

Evita usar K-means si tus datos no cumplen los supuestos de clusters esféricos y de tamaño similar, o si los clusters que buscas detectar tienen formas y densidades variables. K-means es sensible a los valores atípicos, así que si tus datos contienen anomalías significativas, estas pueden sesgar los resultados y producir un clustering pobre. Además, K-means requiere un número predefinido de clusters ($k$), que puede ser difícil de determinar sin conocimiento previo de la estructura de los datos. Si tu dataset incluye características no numéricas o si necesitas un método de clustering más flexible que pueda manejar formas y densidades arbitrarias de clusters, como el clustering espacial basado en densidad de aplicaciones con ruido (DBSCAN) o el clustering jerárquico, K-means puede no ser la mejor opción.

#### Reducción de dimensionalidad (_Dimensionality Reduction_)

Los algoritmos de reducción de dimensionalidad son fundamentales en el ML no supervisado, porque ayudan a simplificar datasets complejos reduciendo el número de características sin perder la información esencial. Estas técnicas transforman datos de alta dimensión en una forma de menor dimensión, lo que facilita visualizar y analizar tus datos. Entre los algoritmos comunes están PCA, que identifica las direcciones (componentes principales) que maximizan la varianza de los datos, y la incrustación estocástica de vecinos distribuida en t (t-SNE), que se centra en preservar la estructura local y las relaciones entre los puntos de datos. Al reducir la dimensionalidad, estos algoritmos ayudan a descubrir patrones subyacentes, facilitan la compresión de datos y mejoran la eficiencia de las tareas de ML posteriores.

Amazon SageMaker ofrece PCA como algoritmo integrado de reducción de dimensionalidad, lo que permite a ingenieros de ML y científicos de datos reducir eficazmente el número de características de un dataset preservando la mayor variabilidad posible, y así mejorar la eficiencia y el rendimiento de los modelos de ML posteriores.

##### Análisis de componentes principales (_Principal Component Analysis_)

PCA es una técnica potente de reducción de dimensionalidad, muy usada en ML y análisis de datos. Funciona identificando los componentes principales, que son las direcciones de los datos que explican la mayor varianza. Al transformar los datos originales de alta dimensión a un nuevo sistema de coordenadas definido por estos componentes principales, PCA reduce eficazmente el número de características preservando la mayor parte posible de la variabilidad de los datos. Esto facilita visualizar y analizar los datos, y también mejora el rendimiento de los algoritmos de ML posteriores al mitigar problemas relacionados con la multicolinealidad y el sobreajuste.

Con referencia al ejemplo anterior, apliquemos el algoritmo PCA al dataset Iris para reducir la dimensionalidad de cuatro a tres.

El siguiente programa en Python usa el algoritmo PCA para reducir la dimensionalidad de nuestro dataset y después aplica el clustering K-means para agrupar los puntos de datos en tres clusters:

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from sklearn.datasets import load_iris
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA
from sklearn.cluster import KMeans
from mpl_toolkits.mplot3d import Axes3D
import os

# Create the output directory if it doesn't exist
output_dir = './ch04/images'
os.makedirs(output_dir, exist_ok=True)

# Load the Iris dataset
iris = load_iris()
data = iris.data

# Scale the data
scaler = StandardScaler()
scaled_data = scaler.fit_transform(data)

# Apply PCA to reduce to 3 dimensions
pca_3d = PCA(n_components=3)
pca_data_3d = pca_3d.fit_transform(scaled_data)

# Apply K-Means clustering
kmeans = KMeans(n_clusters=3, random_state=42)
kmeans.fit(pca_data_3d)
cluster_assignments_3d = kmeans.predict(pca_data_3d)
centroids_3d = kmeans.cluster_centers_

# Plotting the results in 3D
fig = plt.figure(figsize=(10, 8))
ax = fig.add_subplot(111, projection='3d')

# Define colors and markers
colors = ['blue', 'orange', 'purple']
markers = ['o', 's', 'D']

# Plot each cluster with different color and marker
for cluster in np.unique(cluster_assignments_3d):
    ax.scatter(pca_data_3d[cluster_assignments_3d == cluster, 0],
               pca_data_3d[cluster_assignments_3d == cluster, 1],
               pca_data_3d[cluster_assignments_3d == cluster, 2],
               c=colors[cluster],
               marker=markers[cluster],
               label=f'Cluster {cluster + 1}',
               edgecolor='black')

# Plot centroids with yellow stars and enumerate them
for i, centroid in enumerate(centroids_3d):
    ax.scatter(centroid[0], centroid[1], centroid[2],
               s=300, c='yellow', marker='*',
               edgecolor='black', label=f'Centroid {i + 1}')
    ax.text(centroid[0], centroid[1], centroid[2], f'C{i + 1}',
            color='black', fontsize=12, ha='center', va='center')

ax.set_xlabel('Principal Component 1')
ax.set_ylabel('Principal Component 2')
ax.set_zlabel('Principal Component 3')
ax.legend()
ax.set_title('3D K-Means Clustering of Iris Dataset Using PCA')

# Save the plot to the specified file path
plt.savefig(os.path.join(output_dir, 'pca_kmeans_iris_plot_3d.png'))
plt.close()
print(f"3D plot has been saved in '{output_dir}' as 'pca_kmeans_iris_plot_3d.png'")
```

Ejecuté este programa en Amazon SageMaker Studio; la Figura 4.18 muestra la imagen resultante, que ilustra el clustering K-means en 3D del dataset Iris con las dimensiones reducidas por PCA y los centroides resaltados.

_Figura 4.18 Clustering K-means en 3D del dataset Iris usando PCA._

La gráfica muestra tres clusters distintos, cada uno representado con una forma diferente:

- Cluster 1: círculos
- Cluster 2: cuadrados
- Cluster 3: rombos

Los ejes de la gráfica 3D representan los tres componentes principales (PC1, PC2 y PC3) que produce el algoritmo PCA. Estos componentes capturan la varianza máxima del dataset, reduciendo eficazmente su dimensionalidad y preservando su estructura esencial.

Los centroides de cada cluster se resaltan con estrellas y se enumeran como C1, C2 y C3. Estos centroides representan los puntos centrales de cada cluster y sirven de referencia para las características típicas de los puntos de datos de cada grupo.

La separación espacial de los clusters en la gráfica 3D demuestra la eficacia de PCA para reducir la dimensionalidad y de K-means para identificar grupos distintos dentro de los datos. La clara distinción entre los clusters sugiere que las características de las flores de Iris quedan bien representadas por los componentes principales elegidos.

Amazon SageMaker ofrece una implementación de PCA altamente escalable y eficiente, que puede manejar datasets grandes con facilidad. Para usar PCA en Amazon SageMaker, empiezas creando un objeto _estimator_ de PCA y especificando el número de componentes que quieres conservar. Después ajustas el _estimator_ a tu dataset, lo que implica calcular los componentes principales y transformar los datos originales al nuevo espacio de menor dimensión. El algoritmo PCA de Amazon SageMaker admite formatos de datos tanto dispersos como densos, lo que lo hace versátil para diversos tipos de tareas de preprocesamiento de datos.

Una vez entrenado el modelo PCA, puedes desplegarlo como un endpoint en Amazon SageMaker para transformar datos nuevos al vuelo, o puedes usarlo en transformaciones por lotes (_batch transformations_) para datasets más grandes. Esto permite una integración fluida en tus flujos de trabajo de ML, lo que te permite preprocesar datos en tiempo real o a escala. Una **transformación por lotes** (_batch transform_) es la alternativa al endpoint cuando no necesitas respuestas inmediatas: SageMaker arranca instancias con el modelo, lee todos los archivos de una carpeta de S3, escribe los resultados en otra carpeta de S3 (un archivo con el sufijo `.out` por cada archivo de entrada) y **apaga las instancias al terminar**. Pagas solo la duración del trabajo, no un servicio encendido.

Además, la integración de Amazon SageMaker con otros servicios de AWS, como Amazon S3 para el almacenamiento de datos y Amazon SageMaker Studio para el desarrollo, ofrece un entorno integral para desarrollar, probar y desplegar modelos PCA. Al aprovechar el algoritmo PCA integrado de Amazon SageMaker, los científicos de datos y los ingenieros de ML pueden reducir eficientemente la dimensionalidad, mejorar el rendimiento de los modelos y agilizar sus canalizaciones de procesamiento de datos.

El siguiente fragmento en Python muestra un ejemplo de cómo usar PCA con Amazon SageMaker:

```python
import pandas as pd
import boto3
import sagemaker
from sagemaker import get_execution_role
import os

# Initialize the SageMaker session
sagemaker_session = sagemaker.Session()

# Get the execution role
role = get_execution_role()

# Load the Iris dataset
from sklearn.datasets import load_iris
iris = load_iris()
df = pd.DataFrame(iris.data, columns=iris.feature_names)

# Save the DataFrame to a CSV file
data_csv = 'iris.csv'
df.to_csv(data_csv, index=False, header=False)

# Upload the CSV file to S3
s3_bucket = sagemaker_session.default_bucket()
s3_data_path = f's3://{s3_bucket}/pca/iris.csv'
boto3.Session().resource('s3').Bucket(s3_bucket).Object('pca/iris.csv').upload_file(data_csv)

# Get the PCA container image URI
pca_image_uri = sagemaker.image_uris.retrieve(
    'pca',
    boto3.Session().region_name
)

# Create the PCA estimator
pca = sagemaker.estimator.Estimator(
    image_uri=pca_image_uri,
    role=role,
    instance_count=1,
    instance_type='ml.m4.xlarge',
    output_path=f's3://{s3_bucket}/pca/output'
)

# Set PCA hyperparameters
pca.set_hyperparameters(
    feature_dim=4,
    num_components=3,
    subtract_mean=True,
    algorithm_mode='regular'
)

# Fit the PCA model
pca.fit({'train': s3_data_path})

# Deploy the PCA model
pca_transformer = pca.transformer(
    instance_count=1,
    instance_type='ml.m4.xlarge',
    strategy='SingleRecord'
)

# Perform the transformation
transformed_output = pca_transformer.transform(
    data=s3_data_path,
    content_type='text/csv',
    split_type='Line'
)

transformed_output.wait()

# Download and view the transformed data
transformed_output_path = transformed_output.output_path
transformed_data = pd.read_csv(transformed_output_path + '/train.csv.out', header=None)
print(transformed_data.head())
```

Tres elementos de infraestructura nuevos respecto al fragmento de K-means:

- **`default_bucket()`** devuelve (y crea si no existe) un bucket con el nombre `sagemaker-<región>-<id de cuenta>`, pensado para que los ejemplos tengan dónde guardar datos sin configurar nada. En un proyecto real se usan buckets propios con cifrado y políticas de acceso definidas.
- **La subida con boto3** usa el recurso de S3 de boto3 directamente. El `index=False, header=False` de `to_csv` no es un detalle cosmético: SageMaker exige CSV sin encabezado.
- **`pca.transformer(...)`** no despliega un endpoint, aunque el comentario diga `# Deploy the PCA model`: crea un objeto para una transformación por lotes. `strategy='SingleRecord'` envía los registros al modelo de uno en uno (la alternativa, `MultiRecord`, los agrupa en lotes hasta un tamaño máximo), y `split_type='Line'` le dice a SageMaker que cada línea del archivo es un registro.

> [!warning] Nota de precisión: el fragmento de PCA no funciona tal cual
> Además de requerir el SDK v2, tiene seis problemas. Los tres primeros se verificaron en la documentación del algoritmo y el resto en el código fuente del SDK 2.257.6:
>
> 1. **Falta `mini_batch_size`**, que en PCA es obligatorio junto con `feature_dim` y `num_components`.
> 2. **El canal de entrenamiento no declara CSV.** Igual que en K-means, hay que pasar `TrainingInput(s3_data_path, content_type='text/csv;label_size=0')`.
> 3. **La salida no es CSV.** PCA devuelve sus resultados en JSON, JSON Lines o recordIO-protobuf (por ejemplo, `{"projection": [...]}` por registro), así que `pd.read_csv` no los interpreta. Conviene pedir JSON Lines con `accept='application/jsonlines'` y `assemble_with='Line'` al crear el transformer.
> 4. **`transform()` no devuelve nada.** El método devuelve `None` (y por defecto espera a que termine el trabajo), así que `transformed_output.wait()` lanza `AttributeError`. La ruta de salida está en `pca_transformer.output_path`.
> 5. **El archivo de salida se llama `iris.csv.out`**, porque toma el nombre del archivo de entrada, no `train.csv.out`.
> 6. **Leer `s3://...` con pandas** requiere el paquete `s3fs` instalado; si no está, hay que descargar el archivo antes con boto3.
>
> Versión corregida de la parte final (SDK v2; no se ejecutó contra AWS):
>
> ```python
> from sagemaker.inputs import TrainingInput
>
> pca.set_hyperparameters(feature_dim=4, num_components=3, mini_batch_size=50,
>                         subtract_mean=True, algorithm_mode='regular')
> pca.fit({'train': TrainingInput(s3_data_path, content_type='text/csv;label_size=0')})
>
> pca_transformer = pca.transformer(
>     instance_count=1,
>     instance_type='ml.m5.xlarge',
>     strategy='SingleRecord',
>     accept='application/jsonlines',
>     assemble_with='Line',
> )
> pca_transformer.transform(data=s3_data_path, content_type='text/csv', split_type='Line')
>
> out = f"{pca_transformer.output_path}/iris.csv.out"
> transformed_data = pd.read_json(out, lines=True)   # requiere s3fs
> print(transformed_data.head())
> ```

Aprenderás más sobre el ajuste de hiperparámetros, el despliegue de modelos y el monitoreo de modelos en los capítulos 5, 6 y 7, respectivamente.

###### Casos de uso (_Use Cases_)

PCA es muy útil en escenarios donde hay que reducir la dimensionalidad de los datos para mejorar la eficiencia del análisis y del modelado. Es particularmente eficaz con datasets de alta dimensión en los que preocupa la multicolinealidad (alta correlación) entre características. Al transformar los datos en un nuevo conjunto de componentes ortogonales, PCA ayuda a conservar la varianza más significativa reduciendo el número de variables. Entre los casos de uso comunes están la compresión de imágenes, donde PCA reduce el número de píxeles necesarios para representar una imagen sin una pérdida significativa de calidad, y el análisis exploratorio de datos, donde ayuda a visualizar datasets complejos proyectándolos en espacios de menor dimensión. Además, PCA se usa para preprocesar datos para modelos de ML, mejorando el rendimiento al reducir el ruido y el riesgo de sobreajuste.

Sin embargo, PCA no debe usarse de manera indiscriminada. Una limitación es que supone relaciones lineales entre las variables, lo que podría no capturar la complejidad de datos con interacciones no lineales. Tampoco es adecuado para datasets donde la varianza no es una medida apropiada de importancia, porque PCA prioriza las características con mayor varianza y puede descartar varianzas menores pero cruciales. Además, la interpretabilidad de los componentes principales puede ser un desafío, porque son combinaciones de las características originales que pueden no tener un significado claro e intuitivo. Por lo tanto, aunque PCA es una herramienta potente de reducción de dimensionalidad, es esencial considerar la naturaleza de tus datos y los requisitos específicos de tu análisis antes de aplicarlo.

#### Modelado de temas (_Topic Modeling_)

El modelado de temas se considera una técnica de ML no supervisado porque no requiere datos etiquetados para identificar patrones o temas dentro de una colección de documentos. A diferencia del aprendizaje supervisado, donde los modelos se entrenan con pares de entrada y salida para predecir resultados específicos, los algoritmos de modelado de temas como LDA y NTM exploran la estructura inherente de los datos. Buscan descubrir los temas subyacentes que mejor representan la semántica de los documentos, basándose únicamente en patrones de coocurrencia de palabras, sin ningún conocimiento previo ni anotaciones. Esto hace que el modelado de temas sea muy valioso para explorar y entender grandes datasets de texto no estructurado, lo que permite a los usuarios descubrir temas e ideas ocultos sin necesidad de datos de entrenamiento previamente etiquetados.

Por analogía, piensa en el modelado de temas como algo parecido al clustering. Igual que los algoritmos de clustering agrupan puntos de datos según su similitud, los algoritmos de modelado de temas agrupan documentos según sus temas subyacentes. Al reconocer patrones y relaciones en el texto, estos algoritmos pueden categorizar documentos en temas distintos, lo que facilita gestionar y entender grandes volúmenes de datos textuales.

Amazon SageMaker ofrece LDA y NTM de serie, lo que permite a los usuarios implementar y escalar sin esfuerzo sus soluciones de modelado de temas sin requerir experiencia previa en deep learning.

Un requisito de formato común a ambos algoritmos, que el libro no menciona: LDA y NTM de SageMaker no aceptan texto. Los documentos deben convertirse antes a una tabla de conteos, donde cada fila es un documento y cada columna es el número de veces que aparece una palabra del vocabulario, guardada en recordIO-protobuf o en CSV. Esa conversión (tokenizar, construir el vocabulario, contar) la haces tú antes de subir los datos a S3, por ejemplo en un trabajo de procesamiento de SageMaker o de AWS Glue.

##### Asignación latente de Dirichlet (_Latent Dirichlet Allocation_)

LDA es un modelo probabilístico generativo que se usa para identificar temas dentro de un gran corpus de texto. Supone que cada documento es una mezcla de temas y que cada tema es una mezcla de palabras. El nombre «Dirichlet» proviene de la distribución de Dirichlet, un tipo de distribución de probabilidad que es esencial en LDA. La distribución de Dirichlet se usa para modelar la incertidumbre sobre las probabilidades de los distintos temas dentro de un documento. En esencia, ofrece una forma de asignar probabilidades a los distintos temas de un documento, asegurando que la suma de estas probabilidades sea uno. Esta distribución ayuda a determinar la proporción de los distintos temas en cada documento, lo que permite a LDA descubrir la estructura temática latente del texto. La forma de la distribución de Dirichlet se controla con un conjunto de parámetros (llamados parámetros de concentración) que pueden ajustarse para influir en cómo se distribuye la masa de probabilidad entre los distintos temas. Al actualizar iterativamente las distribuciones de temas sobre documentos y de palabras sobre temas, LDA puede identificar temas significativos que representan mejor la semántica del texto.

En AWS, LDA puede usarse de forma eficiente mediante Amazon SageMaker, que ofrece un algoritmo LDA integrado para el modelado de temas. Para usar LDA en Amazon SageMaker, empiezas preparando tus datos de texto y subiéndolos a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Después creas un _estimator_ de LDA especificando el número de temas y otros hiperparámetros. Amazon SageMaker se encarga del proceso de entrenamiento, aprovechando su infraestructura escalable para manejar datasets grandes y tareas computacionalmente intensivas. Una vez entrenado el modelo, puedes usarlo para transformar documentos nuevos y obtener información sobre sus distribuciones de temas. Esta capacidad es particularmente útil en aplicaciones como el clustering de documentos, la recomendación de contenido y la recuperación de información, donde entender la estructura temática del texto es crucial. Al integrar LDA en tu flujo de trabajo de AWS, puedes aprovechar el poder del modelado de temas para mejorar tus soluciones de análisis de datos y de ML.

> [!warning] Nota de precisión: la «infraestructura escalable» de LDA tiene un límite
> Según la tabla de parámetros de los algoritmos integrados, LDA **solo entrena en CPU y en una sola instancia**: no admite GPU ni entrenamiento distribuido. Si el corpus crece, la única palanca es una instancia más grande. NTM, en cambio, admite CPU o GPU y varias instancias, lo que es un criterio de elección entre ambos en preguntas de examen.

###### Casos de uso (_Use Cases_)

LDA es muy eficaz en escenarios donde entender la estructura temática de un gran corpus de texto es crítico. Destaca en aplicaciones como el clustering de documentos, la recomendación de contenido y la recuperación de información. Por ejemplo, en un servicio de agregación de noticias, LDA puede ayudar a agrupar artículos por temas como política, deportes o tecnología, mejorando la experiencia de usuario al permitir la navegación por temas. De forma similar, en la investigación académica, LDA puede usarse para analizar un gran número de artículos de investigación y descubrir los temas predominantes y sus tendencias a lo largo del tiempo. Al identificar la distribución de temas dentro de cada documento, LDA permite a empresas e investigadores extraer información de datos de texto no estructurado de forma eficiente.

Sin embargo, LDA tiene algunas limitaciones debidas a su supuesto de temas con distribución de Dirichlet, que podría no capturar dependencias más complejas entre palabras y temas. En esos casos, NTM puede entrar en juego. NTM aprovecha redes neuronales para aprender representaciones de temas, lo que permite mayor flexibilidad y la capacidad de capturar patrones intrincados en los datos. Esto hace que NTM sea particularmente eficaz con datasets con fuertes dependencias temporales o secuenciales, o donde las relaciones entre palabras y temas no están bien modeladas por una distribución de Dirichlet. Al incorporar NTM a tu flujo de trabajo de AWS, puedes abordar algunas de las deficiencias de LDA y mejorar tu capacidad de analizar y entender datos de texto complejos, asegurando representaciones de temas más exactas y significativas.

##### Modelo neuronal de temas (_Neural Topic Model_)

NTM es un enfoque avanzado de modelado de temas que utiliza redes neuronales para aprender representaciones de temas más flexibles y matizadas que los métodos tradicionales como LDA. Al aprovechar el poder del deep learning, NTM puede capturar patrones y dependencias complejos en los datos textuales, lo que lo hace muy eficaz para extraer temas significativos de grandes volúmenes de texto. A diferencia de LDA, que se basa en distribuciones de Dirichlet para modelar la probabilidad de los temas dentro de los documentos, NTM usa redes neuronales para aprender automáticamente estas distribuciones, lo que ofrece mayor adaptabilidad a distintos tipos de datos de texto. Esto da como resultado temas más exactos e interpretables, que pueden ser increíblemente valiosos en aplicaciones como el clustering de documentos, la recomendación de contenido y el análisis de sentimiento.

NTM puede implementarse sin fricciones con Amazon SageMaker, que ofrece un algoritmo integrado de NTM. Para usar NTM en Amazon SageMaker, empiezas preparando tus datos de texto y subiéndolos a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Después creas un _estimator_ de NTM en Amazon SageMaker especificando los hiperparámetros y las configuraciones de entrenamiento necesarios. La infraestructura escalable de Amazon SageMaker maneja las demandas computacionales del entrenamiento del algoritmo NTM, lo que lo hace adecuado para datasets grandes. Una vez entrenado el modelo, puede usarse para transformar documentos nuevos y generar distribuciones de temas, lo que aporta información valiosa sobre la estructura temática de tus datos de texto. Esta capacidad permite a las empresas analizar y entender sus datos textuales de forma eficiente, tomar decisiones basadas en datos y mejorar sus flujos de trabajo de ML. Al integrar NTM en tu entorno de AWS, puedes aprovechar las fortalezas de las redes neuronales para descubrir información más profunda en datos textuales complejos.

###### Casos de uso (_Use Cases_)

NTM es particularmente útil en escenarios donde los métodos tradicionales de modelado de temas, como LDA, se quedan cortos. Por ejemplo, NTM destaca con datasets grandes y complejos en los que las relaciones entre palabras y temas son intrincadas y no quedan bien representadas por distribuciones de Dirichlet. Su capacidad de aprovechar el deep learning le permite capturar representaciones de temas más matizadas y flexibles, lo que lo hace ideal para aplicaciones como el análisis de sentimiento, el análisis de comentarios de clientes y la exploración temática de corpus de texto extensos. Además, NTM es muy adecuado para tareas que requieren entender dependencias temporales o secuenciales en el texto, lo que aporta información más exacta y significativa sobre la estructura temática subyacente.

No obstante, NTM puede no ser la mejor opción en todos los casos de uso. Con datasets más pequeños o en situaciones donde los recursos computacionales son limitados, la complejidad y las demandas de recursos del algoritmo NTM podrían superar sus beneficios. En esos casos, modelos más simples como LDA pueden ser más eficientes, rentables y suficientes. Además, si los datos de texto están muy estructurados o si los temas ya están bien definidos, la flexibilidad adicional de NTM podría no ofrecer ventajas significativas frente a los métodos tradicionales de modelado de temas. Es esencial considerar la naturaleza de los datos y los requisitos específicos del análisis para determinar el algoritmo más apropiado para la tarea.

#### Detección de anomalías (_Anomaly Detection_)

La detección de anomalías es un enfoque de ML no supervisado porque identifica valores atípicos o patrones inusuales en los datos sin requerir ejemplos etiquetados de anomalías. Esto es particularmente útil cuando los datos etiquetados son escasos o cuando las anomalías son raras e impredecibles.

Amazon SageMaker ofrece Random Cut Forest (RCF) e IP Insights como algoritmos integrados de detección de anomalías.

- RCF es un algoritmo potente de detección de anomalías no supervisado que funciona creando un bosque de árboles de decisión aleatorios para identificar los puntos de datos que se desvían de la norma.
- IP Insights aprende los patrones de uso de las direcciones IPv4 para detectar actividades sospechosas, como intentos de inicio de sesión inusuales o la creación de recursos desde direcciones IP anómalas.

Ambos algoritmos están diseñados para funcionar sin datos etiquetados, lo que los hace ideales para la detección de anomalías en tiempo real en diversas aplicaciones.

##### Random Cut Forest

El algoritmo RCF funciona construyendo un bosque de árboles de decisión aleatorios. Cada árbol del bosque se construye con una muestra aleatoria de los datos de entrenamiento, y las anomalías se detectan según la profundidad a la que caen los puntos de datos dentro de estos árboles. Los puntos de datos que alteran significativamente la estructura de los árboles, dando lugar a ubicaciones inusualmente profundas, se marcan como anomalías. Este método es muy eficaz para identificar valores atípicos en datasets grandes, como la detección de transacciones fraudulentas o de intrusiones en la red.

Como otros algoritmos de ML, para implementar RCF en Amazon SageMaker empiezas preparando tu dataset y subiéndolo a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Después creas un _estimator_ de RCF en Amazon SageMaker especificando hiperparámetros como el número de árboles y el número de muestras por árbol. Amazon SageMaker se encarga del proceso de entrenamiento, aprovechando su infraestructura escalable para construir el RCF de forma eficiente. Una vez entrenado el modelo, puedes desplegarlo en un endpoint de Amazon SageMaker para la detección de anomalías en tiempo real, lo que te permite monitorear las anomalías y responder a ellas a medida que ocurren.

Del lado de la infraestructura, RCF es de los pocos algoritmos integrados que **solo entrena en CPU** (según la tabla de parámetros), aunque admite el modo Pipe y varias instancias. Los hiperparámetros que menciona el libro se llaman `num_trees` y `num_samples_per_tree`. Un detalle operativo importante: el endpoint no responde «anomalía sí o no», sino que devuelve una **puntuación de anomalía** por registro. El umbral a partir del cual actuar (alertar, bloquear, revisar) lo defines tú en la aplicación que llama al endpoint.

###### Casos de uso (_Use Cases_)

RCF es particularmente útil para detectar anomalías en datasets grandes y de alta dimensión, donde los métodos tradicionales podrían tener dificultades. Destaca en aplicaciones como la detección de fraude, la seguridad de redes y el mantenimiento predictivo. Por ejemplo, en los servicios financieros, RCF puede identificar patrones de transacción inusuales que podrían indicar actividad fraudulenta. En la seguridad de redes, puede detectar anomalías en el tráfico que sugieran posibles intrusiones. De forma similar, en entornos industriales, RCF puede monitorear los datos de los sensores de la maquinaria para detectar señales tempranas de falla (p. ej., MTBF, tiempo medio entre fallas), lo que ayuda a prevenir averías y tiempos de inactividad costosos. Su capacidad de manejar grandes volúmenes de datos y descubrir patrones sutiles e inesperados lo convierte en una herramienta potente para la detección de anomalías en tiempo real.

En AWS, el «tráfico» de red del caso de seguridad suele venir de los **VPC Flow Logs**, registros que capturan metadatos de cada conexión de red dentro de una VPC (direcciones IP de origen y destino, puertos, bytes transferidos, si se aceptó o rechazó), que se pueden enviar a S3 o a CloudWatch Logs y usar como datos de entrenamiento para RCF. Si lo que se busca es detección de amenazas lista para usar sobre la propia cuenta de AWS, sin entrenar nada, el servicio administrado es **Amazon GuardDuty**, que analiza de forma continua esos registros y los de CloudTrail.

> [!warning] Nota de precisión: MTBF no es una señal de falla
> El MTBF (_mean time between failures_, tiempo medio entre fallas) es una métrica de confiabilidad que se calcula a posteriori sobre un equipo o una flota: cuántas horas de operación transcurren en promedio entre dos fallas. No es una señal que RCF detecte en los sensores. Lo que RCF puede detectar son lecturas anómalas (vibración, temperatura, presión) que suelen preceder a una falla; el MTBF sirve después para medir si el mantenimiento predictivo lo mejoró.

RCF es menos eficaz cuando las anomalías ya están bien definidas y etiquetadas, porque en esos casos los algoritmos de aprendizaje supervisado pueden dar resultados más precisos. Además, RCF podría no rendir bien con datasets muy pequeños, donde el enfoque de muestreo aleatorio podría pasar por alto patrones importantes. Cuando el dataset es pequeño o hay ejemplos etiquetados de anomalías disponibles, otros métodos, como los algoritmos de clasificación supervisada o técnicas estadísticas más simples, pueden ser más apropiados. Es esencial considerar la naturaleza de los datos y los requisitos específicos de la aplicación para determinar si RCF es la opción adecuada.

##### IP Insights

El algoritmo IP Insights es un método de aprendizaje no supervisado que aprende los patrones de uso de las direcciones IPv4 asociándolas con entidades como identificadores de usuario o números de cuenta. Captura las asociaciones entre direcciones IP y entidades, determinando qué tan probable es que una entidad use una dirección IP determinada. IP Insights usa redes neuronales para aprender representaciones vectoriales latentes tanto de las entidades como de las direcciones IP, y puede generar _embeddings_ que pueden usarse en tareas de ML posteriores. Cuando se le consulta con un par (entidad, dirección IPv4), IP Insights devuelve una puntuación que indica qué tan anómalo es el patrón, lo que lo hace útil para detectar actividades sospechosas como intentos de inicio de sesión inusuales o la creación de recursos desde direcciones IP anómalas.

Para leer bien esta sección hace falta el concepto de red que está debajo. Una **dirección IP** identifica a un dispositivo en una red, y es lo que un servidor web ve como «origen» de cada petición. **IPv4** es la versión original del protocolo: direcciones de 32 bits escritas como cuatro números separados por puntos (p. ej., `203.0.113.25`), unos 4 300 millones en total, una cantidad que se agotó hace años. **IPv6** es su sucesora, con direcciones de 128 bits (p. ej., `2001:db8::1`) y un espacio prácticamente inagotable, y una parte creciente del tráfico de internet, sobre todo el móvil, ya llega por IPv6. **IP Insights solo admite IPv4**: si tu aplicación recibe conexiones IPv6, esas peticiones no se pueden puntuar con este algoritmo.

Para implementar IP Insights en Amazon SageMaker, empiezas preparando tus datos en forma de pares (entidad, dirección IPv4) y subiéndolos a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Después creas un _estimator_ de IP Insights en Amazon SageMaker, especificando hiperparámetros como el número de vectores de entidad y el tamaño de los vectores de _embedding_. Amazon SageMaker se encarga del proceso de entrenamiento y, una vez entrenado el modelo, puedes desplegarlo en un endpoint de Amazon SageMaker para predicciones en tiempo real o para procesamiento por lotes. Esto te permite monitorear las anomalías y responder a ellas a medida que ocurren, mejorando tus medidas de seguridad y tu eficiencia operativa.

Según la documentación, los datos de entrenamiento son un **CSV sin encabezado de exactamente dos columnas**: la primera es el identificador de la entidad (una cadena opaca, como un ID de usuario) y la segunda la dirección IPv4 en notación decimal con puntos. IP Insights solo admite el modo File. Los hiperparámetros que menciona el libro se llaman `num_entity_vectors` y `vector_dim`, y juntos determinan el tamaño del modelo (hasta 8 GB) y, con él, la memoria de la instancia necesaria. Para entrenar se recomiendan instancias con GPU y, para la inferencia, instancias de CPU. El endpoint devuelve, para cada par consultado, un número (`dot_product`) que tu aplicación compara con un umbral; la documentación pone como ejemplo que un servidor de inicio de sesión pida un segundo factor de autenticación (MFA) cuando la puntuación cruza ese umbral.

###### Casos de uso (_Use Cases_)

IP Insights es más adecuado para detectar comportamientos anómalos en escenarios donde rastrear los patrones de uso de las direcciones IP es crítico. Destaca en aplicaciones como la detección de intentos de inicio de sesión fraudulentos, de patrones de acceso inusuales y la identificación de cuentas comprometidas. Por ejemplo, las plataformas de comercio electrónico y los servicios en línea pueden usar IP Insights para monitorear y marcar actividades de inicio de sesión sospechosas que se desvían del comportamiento normal de los usuarios, mejorando la seguridad y previniendo el acceso no autorizado. Además, puede usarse para proteger recursos identificando y mitigando patrones inusuales de creación o uso de recursos desde direcciones IP anómalas, lo que ayuda a mantener la integridad y la seguridad de los sistemas.

Sin embargo, IP Insights puede no ser adecuado para todos los escenarios. Es menos eficaz en entornos donde las direcciones IP cambian con frecuencia o donde las entidades no tienen patrones de uso de IP consistentes, porque el modelo depende de aprender asociaciones estables entre entidades y direcciones IP. En esos casos, los sistemas tradicionales basados en reglas u otros métodos de detección de anomalías podrían ser más apropiados. Además, IP Insights podría no rendir bien en contextos donde hay datos etiquetados disponibles y los enfoques de aprendizaje supervisado podrían ofrecer una detección de anomalías más precisa.

Que las IP «cambien con frecuencia» es la norma en muchas redes, no la excepción, y conviene saber por qué. Los proveedores de internet domésticos asignan direcciones **dinámicas**, que pueden cambiar cada cierto tiempo o al reiniciar el módem. Los operadores móviles suelen usar **CGNAT** (_carrier-grade NAT_): como no hay direcciones IPv4 suficientes, cientos o miles de clientes salen a internet compartiendo la misma IP pública, de modo que una misma IP corresponde a muchos usuarios y un mismo usuario aparece con IP distintas a lo largo del día. Las **VPN** corporativas y los servicios de privacidad hacen que muchos usuarios compartan unas pocas IP de salida. En todos esos casos, la asociación entre entidad e IP que aprende el algoritmo es débil.

### Algoritmos de análisis textual (_Textual Analysis Algorithms_)

El análisis textual se refiere al proceso de usar algoritmos y técnicas de ML para entender, interpretar y obtener información significativa de datos de texto. Puede implicar diversas tareas, como la clasificación de textos, el análisis de sentimiento, el reconocimiento de entidades, el modelado de temas y el resumen. El objetivo del análisis textual es transformar el texto no estructurado en información estructurada que pueda usarse para tomar decisiones, mejorar la experiencia de usuario y, en última instancia, obtener inferencias.

Ya aprendiste al principio del capítulo cómo Amazon Comprehend aborda el análisis textual como servicio de IA administrado. También aprendiste antes cómo los algoritmos LDA y NTM, que son técnicas de ML no supervisado, cumplen un papel significativo en el análisis textual, específicamente en el modelado de temas. Estos algoritmos identifican automáticamente patrones y agrupan palabras relacionadas en temas dentro de grandes datasets de texto, sin necesitar datos etiquetados.

En esta sección nos centraremos en técnicas más avanzadas de análisis textual que ofrecen los algoritmos integrados BlazingText y Sequence-to-Sequence de Amazon SageMaker.

#### BlazingText

BlazingText es un algoritmo eficiente y escalable que ofrece Amazon SageMaker para tareas de NLP, optimizado específicamente para la clasificación de textos y la generación de _word embeddings_ con el modelo Word2Vec. BlazingText está diseñado para procesar grandes volúmenes de datos de texto con rapidez, lo que lo hace adecuado para aplicaciones en tiempo real y datasets grandes. Este algoritmo es particularmente beneficioso en tareas como la similitud semántica, el análisis de sentimiento y la clasificación de documentos, donde entender las relaciones entre las palabras es crucial.

> [!note]
> Un _embedding_, en el contexto del NLP y del ML, es una representación vectorial densa de los datos, normalmente palabras, que captura su significado, sus propiedades sintácticas y sus relaciones con otras palabras. Estos _embeddings_ se aprenden de los datos y se usan para convertir datos categóricos en datos numéricos continuos, que después pueden procesar los modelos de ML. Por ejemplo, en los _word embeddings_, cada palabra de un vocabulario se asigna a un espacio vectorial de alta dimensión. Las palabras con significados o usos similares quedan cerca unas de otras en ese espacio, lo que permite a los modelos entender la similitud semántica y las relaciones entre palabras. Dicho de otro modo, los _embeddings_ traducen la similitud semántica, tal como la perciben los humanos, a proximidad en un espacio vectorial. Entre las técnicas comunes de _embedding_ están Word2Vec, GloVe y FastText, que han mejorado significativamente el rendimiento de diversas tareas de NLP, como la clasificación de textos, el análisis de sentimiento y la traducción automática.

BlazingText opera con dos técnicas principales: Word2Vec y la clasificación de textos. El modelo Word2Vec crea representaciones vectoriales densas de las palabras, también conocidas como _word embeddings_, analizando grandes corpus de texto y capturando las relaciones semánticas y sintácticas entre las palabras. Estos _word embeddings_ se usan después para entender similitudes y analogías entre palabras. Para la clasificación de textos, BlazingText emplea implementaciones eficientes de modelos de deep learning que pueden categorizar el texto en clases predefinidas a partir de los _word embeddings_ aprendidos. El algoritmo aprovecha el multihilo (_multithreading_) y la aceleración por hardware para agilizar el proceso de entrenamiento, asegurando un rendimiento rápido y escalable. El **multihilo** significa que el programa reparte el trabajo entre todos los núcleos de la CPU de la instancia a la vez, en lugar de usar uno solo; la **aceleración por hardware** se refiere aquí a las GPU, para las que BlazingText incluye código escrito en CUDA, el lenguaje de programación de las GPU de NVIDIA. Por eso el tipo de instancia importa tanto en este algoritmo, como se detalla más abajo.

Para usar BlazingText en Amazon SageMaker, empiezas preparando tus datos de texto y subiéndolos a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Después creas un _estimator_, especificando BlazingText como algoritmo, y configuras los hiperparámetros necesarios para tu trabajo de entrenamiento. Una vez configurado el _estimator_, lanzas el trabajo de entrenamiento apuntándolo a la ubicación de tus datos en Amazon S3. Una vez completado el entrenamiento, puedes desplegar el modelo en un endpoint para predicciones en tiempo real o usarlo para inferencia por lotes. El siguiente código en Python puede usarse como referencia:

```python
import sagemaker

# Initialize the SageMaker session
sagemaker_session = sagemaker.Session()
role = '<your-iam-role>'

# Get the BlazingText container image
container = sagemaker.image_uris.retrieve(
    'blazingtext', sagemaker_session.boto_region_name
)

# Create the BlazingText estimator
bt_estimator = sagemaker.estimator.Estimator(
    container,
    role,
    instance_count=1,
    instance_type='ml.c4.2xlarge',
    output_path='s3://<your-bucket>/output',
    sagemaker_session=sagemaker_session
)

# Set hyperparameters
bt_estimator.set_hyperparameters(
    mode='supervised',
    epochs=10,
    min_count=2,
    learning_rate=0.05
)

# Launch the training job
bt_estimator.fit({'train': 's3://<your-bucket>/train'})

# Deploy the model
predictor = bt_estimator.deploy(
    initial_instance_count=1,
    instance_type='ml.m4.xlarge'
)
```

Observa cómo puedes especificar el modo de operación del _estimator_ en la instrucción `bt_estimator.set_hyperparameters()`.

> [!note]
> Este método recibe un objeto diccionario como único parámetro, cuyas claves están definidas en la documentación de Amazon SageMaker.
>
> Los hiperparámetros del algoritmo BlazingText dependen del modo que uses: Word2Vec (no supervisado) o clasificación de textos (supervisado). Para más información sobre este método, visita https://docs.aws.amazon.com/sagemaker/latest/dg/blazingtext_hyperparameters.html.

> [!warning] Nota de precisión: cómo se pasan los hiperparámetros
> En el SDK v2, `set_hyperparameters()` no recibe un diccionario como argumento posicional, sino argumentos con nombre (`clave=valor`), que es justo lo que hace el fragmento. Si tienes los hiperparámetros en un diccionario, se pasan desempaquetados: `bt_estimator.set_hyperparameters(**mis_hiperparametros)`. Lo que sí es un diccionario es cómo viajan finalmente a la API `CreateTrainingJob`: un mapa de cadena a cadena.

A diferencia de los fragmentos anteriores, aquí el rol se escribe a mano (`'<your-iam-role>'`): hay que sustituirlo por el ARN completo de un rol de IAM, con la forma `arn:aws:iam::<id de cuenta>:role/<nombre>`, que SageMaker pueda asumir y que tenga permisos sobre los buckets de entrada y salida. Esta es la forma de trabajar fuera de Studio, donde `get_execution_role()` no funciona.

> [!warning] Nota de precisión: configuración de BlazingText (verificado el 25-09-2026)
>
> - **Modos.** El hiperparámetro `mode` elige el algoritmo: `cbow`, `skipgram` o `batch_skipgram` para Word2Vec, y `supervised` para clasificación de textos.
> - **Formato de los datos.** BlazingText espera **un solo archivo de texto preprocesado**, con una oración por línea y los tokens separados por espacios; si tienes varios archivos, hay que concatenarlos. En modo `supervised`, cada línea lleva además su etiqueta con el prefijo `__label__` (p. ej., `__label__2 el producto llegó roto`). En CPU, el modo supervisado admite también un manifiesto aumentado en JSON Lines, que permite entrenar en modo Pipe.
> - **Instancias por modo.** `cbow` y `skipgram` usan una sola instancia de CPU o de GPU (la documentación recomienda `ml.p3.2xlarge`). `batch_skipgram` es el único modo que puede repartirse entre **varias instancias de CPU**, aunque la tabla general de parámetros diga «una sola instancia». Para `supervised`, se recomienda una instancia C5 si el dataset de entrenamiento pesa menos de 2 GB y una instancia con una GPU si es mayor. Las `ml.c4.2xlarge` y `ml.m4.xlarge` del fragmento son de generación anterior.
> - **Inferencia.** El endpoint recibe JSON con la forma `{"instances": ["oración 1", "oración 2"]}` y devuelve etiquetas y probabilidades. El modelo resultante es compatible con fastText, la biblioteca de código abierto en la que se basa.

###### Casos de uso (_Use Cases_)

BlazingText es ideal para escenarios que requieren clasificación de textos o creación de _word embeddings_ eficientes y escalables. Es particularmente útil en aplicaciones con grandes volúmenes de datos de texto, como el procesamiento de reseñas de clientes, el análisis de redes sociales y la categorización de documentos. El modelo Word2Vec de BlazingText es excelente para tareas como la similitud semántica, el clustering de palabras y la construcción de representaciones de características para tareas de NLP posteriores. Además, su velocidad y escalabilidad lo hacen adecuado para aplicaciones en tiempo real y datasets grandes, donde el procesamiento rápido y el alto rendimiento son esenciales.

Sin embargo, BlazingText puede no ser la mejor opción para tareas más complejas de comprensión del lenguaje natural que requieren contexto más allá de los _word embeddings_, como el análisis contextual profundo o la generación de lenguaje con matices. Para tareas que implican entender el contexto de oraciones o párrafos, como responder preguntas o la IA conversacional avanzada, FMs como `cohere.embed-english-v3` y `cohere.embed-multilingual-v3` podrían ser más apropiados, por su capacidad de capturar la información contextual con más eficacia. Estos FMs producen _embeddings_ y están disponibles en Amazon Bedrock. BlazingText también es menos adecuado para tareas que requieren una personalización extensa más allá de la clasificación de textos y los _word embeddings_, donde modelos más especializados o algoritmos construidos a medida pueden ofrecer mejores resultados.

La diferencia de infraestructura entre ambas opciones es la misma que en todo el capítulo. Con BlazingText entrenas tu propio modelo en SageMaker y mantienes un endpoint que se cobra por hora. Con los modelos de _embeddings_ de Bedrock no entrenas nada: llamas a `InvokeModel` con el texto y pagas por token de entrada. En ambos casos, los vectores resultantes se suelen guardar en una base de datos que permite buscar por cercanía entre vectores (en AWS, por ejemplo, Amazon OpenSearch Service o Amazon Aurora PostgreSQL con la extensión pgvector). El catálogo actual de Bedrock incluye además otros modelos de _embeddings_, como Cohere Embed v4, Titan Text Embeddings V2 y Amazon Nova Multimodal Embeddings.

#### Sequence-to-Sequence

El algoritmo Sequence-to-Sequence (Seq2Seq) de Amazon SageMaker es un algoritmo de aprendizaje supervisado diseñado para tareas en las que la entrada es una secuencia de tokens (como texto o audio) y la salida es otra secuencia de tokens. Esto lo hace adecuado para aplicaciones como la traducción automática (traducir texto de un idioma a otro), el resumen de textos (crear un resumen conciso de un texto más largo) y la conversión de voz a texto (convertir el lenguaje hablado en texto escrito). Seq2Seq aprovecha arquitecturas avanzadas de redes neuronales, incluidas las RNN y las redes neuronales convolucionales (CNN) con mecanismos de atención, para modelar y generar secuencias de forma eficaz.

Seq2Seq es el algoritmo integrado con requisitos de infraestructura más estrictos del capítulo. Según la tabla de parámetros, **solo entrena en GPU y en una sola instancia** (puede usar varias GPU dentro de ella), solo admite el modo File y exige **tres canales**: `train`, `validation` y `vocab`. Los datos no se entregan como texto, sino en recordIO-protobuf con las palabras ya convertidas a identificadores enteros, y el canal `vocab` contiene los archivos de vocabulario que traducen esos identificadores a palabras. Para casi todos los casos que describe esta sección, la alternativa dentro de AWS que no exige entrenar ni operar GPU es un servicio de IA: Amazon Translate para traducir, Amazon Transcribe para voz a texto o un modelo de Bedrock para resumir.

###### Casos de uso (_Use Cases_)

El algoritmo Seq2Seq es muy eficaz para tareas donde la entrada y la salida son secuencias, lo que lo convierte en una excelente opción para aplicaciones como la traducción automática, el resumen de textos y la respuesta a preguntas. Por ejemplo, en la traducción automática, Seq2Seq puede traducir texto de un idioma a otro entendiendo el contexto de la secuencia de entrada y generando la secuencia correspondiente en el idioma de destino. De forma similar, en el resumen de textos, los modelos Seq2Seq pueden condensar documentos largos en resúmenes concisos identificando y conservando la información más relevante. Además, Seq2Seq es útil en el desarrollo de chatbots para generar respuestas coherentes y adecuadas al contexto, lo que lo convierte en una herramienta versátil para una amplia gama de tareas de NLP que requieren la transformación o la generación de secuencias.

No obstante, Seq2Seq puede no ser la mejor opción para tareas que no implican principalmente la generación o transformación de secuencias. Por ejemplo, si el objetivo es realizar una clasificación de textos simple, como identificar el sentimiento de un tuit (positivo o negativo), otros modelos, como los clasificadores tradicionales o incluso las CNN, podrían ser más eficientes y consumir menos recursos. De forma similar, las tareas que requieren el análisis de características estáticas, como la clasificación de imágenes y la predicción de series de tiempo sin necesidad de una salida secuencial, se resuelven mejor con otros algoritmos especializados. En esos casos, las capacidades de Seq2Seq pueden ser excesivas y no aportar beneficios adicionales frente a modelos más simples diseñados para esas tareas específicas.

### Algoritmos de procesamiento de imágenes (_Image Processing Algorithms_)

Amazon SageMaker ofrece un conjunto de algoritmos integrados diseñados para agilizar diversas tareas de procesamiento de imágenes, que aprovechan técnicas avanzadas de ML para ofrecer alta exactitud y eficiencia. Entre ellos están algoritmos como Image Classification, que categoriza imágenes en clases predefinidas; Object Detection, que identifica y localiza objetos dentro de las imágenes; Semantic Segmentation, que clasifica cada píxel de una imagen para distinguir objetos distintos, e Image Embeddings, que transforma imágenes en vectores de tamaño fijo para usarlos en diversas tareas posteriores. Cada uno de estos algoritmos ofrece herramientas potentes para automatizar y mejorar los flujos de trabajo de procesamiento de imágenes, lo que los hace esenciales para desarrollar aplicaciones sofisticadas de visión por computadora.

Frente a Amazon Rekognition, visto al principio del capítulo, estos algoritmos tienen sentido cuando las clases o los objetos que te interesan no están entre los que Rekognition reconoce de serie (un defecto de fabricación específico, un tipo de tejido en una imagen médica) y tienes imágenes etiquetadas para entrenar. El precio es operativo: todos ellos entrenan **solo en GPU**, que es el tipo de instancia más caro por hora, y te corresponde preparar los datos en su formato, lanzar los trabajos y operar los endpoints.

> [!warning] Nota de precisión: «Image Embeddings» no es un algoritmo integrado
> La lista oficial de algoritmos integrados de visión (verificada el 25-09-2026) incluye Image Classification (variantes MXNet y TensorFlow), Object Detection (variantes MXNet y TensorFlow) y Semantic Segmentation, pero no un algoritmo de «Image Embeddings». Los modelos preentrenados de _embeddings_ de imagen se encuentran en **SageMaker JumpStart**, el catálogo de modelos preentrenados de SageMaker, y en Bedrock (p. ej., Titan Multimodal Embeddings o Amazon Nova Multimodal Embeddings).

#### Clasificación de imágenes (_Image Classification_)

La clasificación de imágenes es una tarea fundamental de la visión por computadora, cuyo objetivo es asignar una etiqueta o categoría a una imagen de entrada. Este proceso implica analizar el contenido de la imagen y categorizarla en una de varias clases predefinidas. Por ejemplo, en un dataset de fotos de animales, un modelo de clasificación de imágenes puede etiquetar cada imagen como «gato», «perro», «pájaro», etc. La clasificación de imágenes se usa ampliamente en diversas aplicaciones, como las imágenes médicas, la conducción autónoma, la videovigilancia de seguridad y muchas otras, donde la identificación exacta de los objetos dentro de las imágenes es clave.

La clasificación de imágenes es un algoritmo de ML supervisado que aprovecha técnicas de deep learning, en particular las CNN, para procesar y analizar imágenes. Las CNN están diseñadas para aprender automática y adaptativamente jerarquías espaciales de características a partir de las imágenes. El proceso implica varias capas de operaciones convolucionales, _pooling_ y capas totalmente conectadas, que trabajan juntas para extraer características y hacer predicciones. Durante el entrenamiento, el modelo aprende a reconocer patrones y características dentro de las imágenes minimizando el error de clasificación sobre un dataset grande de imágenes etiquetadas. Una vez entrenado, el modelo puede predecir con exactitud la clase de imágenes nuevas, no vistas, a partir de las características que aprendió.

> [!note]
> El algoritmo integrado Image Classification viene en tres «sabores»: el primero está implementado con la popular plataforma de ML TensorFlow, el segundo con el framework de ML PyTorch y el tercero con la biblioteca de deep learning MXNet.

> [!warning] Nota de precisión: solo hay dos variantes integradas
> La lista oficial de algoritmos integrados (verificada el 25-09-2026) incluye **Image Classification – MXNet** e **Image Classification – TensorFlow**; no hay una variante integrada de PyTorch. Los modelos de clasificación de imágenes basados en PyTorch están disponibles como modelos preentrenados en SageMaker JumpStart. Las dos variantes integradas difieren en infraestructura: la de MXNet **entrena solo en GPU** (familias P2, P3, G4dn y G5), admite varias instancias y acepta datos en formato RecordIO de MXNet (un archivo `.rec` por canal), imágenes `.jpg`/`.png` acompañadas de archivos de lista `.lst`, o un manifiesto aumentado para usar el modo Pipe; la de TensorFlow parte de modelos preentrenados, admite CPU o GPU y solo reparte el trabajo entre varias GPU de una misma instancia. Ambas admiten CPU o GPU para la inferencia.

Para usar Image Classification en Amazon SageMaker, empiezas preparando tu dataset etiquetado y subiéndolo a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Después creas un modelo de clasificación de imágenes con un _estimator_ de Amazon SageMaker, especificando el algoritmo y los hiperparámetros necesarios y apuntando a tu dataset. El _estimator_ se encarga del proceso de entrenamiento, usando la potente infraestructura de Amazon SageMaker para entrenar tu modelo de forma eficiente. Una vez completado el entrenamiento, puedes desplegar el modelo como un endpoint para predicciones en tiempo real o para procesamiento por lotes. Amazon SageMaker también ofrece herramientas de evaluación y monitoreo de modelos para asegurar que tu modelo de clasificación de imágenes rinda de forma óptima. Estas herramientas se tratarán en el próximo capítulo.

###### Casos de uso (_Use Cases_)

Los algoritmos de clasificación de imágenes son ideales cuando necesitas categorizar imágenes en clases específicas según su contenido. Estos algoritmos son particularmente útiles en escenarios donde se requieren una identificación y un etiquetado exactos de las imágenes. Por ejemplo, en las imágenes médicas, pueden usarse para clasificar radiografías o resonancias magnéticas y detectar enfermedades. En la industria minorista, la clasificación de imágenes puede ayudar a organizar productos identificando artículos en imágenes. Además, estos algoritmos son valiosos en los sistemas de seguridad y vigilancia para reconocer rostros, placas vehiculares u otros objetos de interés. En general, son beneficiosos en cualquier aplicación cuyo objetivo sea categorizar de forma automática y exacta los datos visuales.

Los algoritmos de clasificación de imágenes pueden no ser adecuados para tareas que requieren entender el contexto o las relaciones entre múltiples objetos dentro de una imagen. Por ejemplo, si el objetivo es detectar y localizar varios objetos en una imagen (detección de objetos) o segmentar distintas regiones (segmentación semántica), se necesitarían algoritmos más especializados. Además, las tareas que implican analizar imágenes para extraer información detallada y estructurada, como el OCR, pueden requerir enfoques distintos. Asimismo, en los casos donde las imágenes carecen de distinciones claras entre categorías o donde el objetivo principal no es la clasificación sino otras formas de análisis, los algoritmos de clasificación de imágenes podrían no ser la opción más eficaz.

#### Detección de objetos (_Object Detection_)

Con la detección de objetos, el objetivo no es solo identificar objetos dentro de una imagen, sino también señalar sus ubicaciones precisas mediante cuadros delimitadores (_bounding boxes_). Esta doble capacidad de clasificación y localización es parte integral de aplicaciones como la conducción autónoma, los sistemas de seguridad, la inteligencia, la vigilancia, el reconocimiento, el sector minorista y muchas otras. Como algoritmo de aprendizaje supervisado, la detección de objetos requiere datasets etiquetados para el entrenamiento, en los que cada imagen incluye anotaciones detalladas que indican los objetos y sus ubicaciones.

El algoritmo Object Detection de Amazon SageMaker emplea técnicas sofisticadas de deep learning, con arquitecturas populares como Single Shot MultiBox Detector (SSD), las redes neuronales convolucionales basadas en regiones (R-CNN) y You Only Look Once (YOLO). El enfoque SSD integra como red base una CNN preentrenada para tareas de clasificación de imágenes. Arquitecturas de CNN populares como VGG-16 y ResNet-50 suelen usarse como columna vertebral (_backbone_). El algoritmo divide la imagen de entrada en una cuadrícula y, para cada celda, predice múltiples cuadros delimitadores y las probabilidades de clase de los objetos dentro de esos cuadros. Durante el entrenamiento, el modelo recibe un dataset de imágenes junto con sus anotaciones de cuadros delimitadores y sus etiquetas de clase correspondientes. Aprende a minimizar el error entre sus predicciones y los datos de referencia (_ground truth_), refinando eficazmente su capacidad de detectar y clasificar objetos con exactitud. La implementación de SSD de Amazon SageMaker también incluye técnicas de aumento de datos, como el volteo, el reescalado y la variación aleatoria (_jittering_), para mejorar la robustez del modelo y evitar el sobreajuste.

> [!warning] Nota de precisión: arquitecturas disponibles en el algoritmo integrado
> Según la documentación (verificada el 25-09-2026), **Object Detection – MXNet usa solo el marco SSD**, con dos redes base posibles: VGG y ResNet. R-CNN y YOLO **no** son opciones de ese algoritmo integrado; modelos como YOLO y Faster R-CNN aparecen entre los modelos preentrenados de SageMaker JumpStart, y la variante **Object Detection – TensorFlow** parte de modelos preentrenados de TensorFlow.

Como con otros algoritmos integrados, para usar la detección de objetos en Amazon SageMaker empiezas preparando y etiquetando tu dataset, asegurándote de que cada imagen contenga anotaciones de los objetos de interés. Este dataset etiquetado se sube después a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Puedes crear un modelo de detección de objetos con un _estimator_ de Amazon SageMaker, especificando el algoritmo (MXNet o TensorFlow) y los hiperparámetros necesarios y apuntando a tu dataset. El _estimator_ gestiona el proceso de entrenamiento, aprovechando la infraestructura administrada de Amazon SageMaker para optimizar el modelo a partir de tus datos. Una vez completado el entrenamiento, puedes desplegar el modelo como un endpoint para predicciones en tiempo real o para procesamiento por lotes.

En la variante MXNet, las anotaciones tienen un formato concreto: si entrenas con imágenes sueltas en lugar de RecordIO, necesitas **cuatro canales** (`train`, `validation`, `train_annotation` y `validation_annotation`), con un archivo JSON por imagen, del mismo nombre, que lista la clase y las coordenadas de cada cuadro delimitador (`left`, `top`, `width`, `height`). También admite un manifiesto aumentado en JSON Lines, que es el formato que producía SageMaker Ground Truth y permite entrenar en modo Pipe. Entrena **solo en GPU** (familias P2, P3, G4dn y G5), con posibilidad de varias GPU y varias instancias, y para la inferencia admite CPU (p. ej., C5 o M5) o GPU.

###### Casos de uso (_Use Cases_)

La detección de objetos es muy eficaz en escenarios donde tanto la identificación como la localización precisa de los objetos dentro de las imágenes son críticas. Esto la hace ideal para aplicaciones como la conducción autónoma, donde los vehículos necesitan detectar y seguir peatones, otros autos y obstáculos para navegar con seguridad. También es valiosa en los sistemas de seguridad y vigilancia, que dependen de identificar amenazas potenciales o actividades sospechosas reconociendo y siguiendo en tiempo real objetos como rostros, vehículos y bolsos. En el sector minorista, la detección de objetos puede ayudar a gestionar el inventario reconociendo y contando automáticamente los productos en los anaqueles, así como a analizar el comportamiento de los clientes siguiendo sus movimientos dentro de la tienda. Además, se usa en las imágenes médicas para localizar e identificar anomalías o enfermedades en los estudios, lo que ayuda al diagnóstico y a la planificación del tratamiento. En las aplicaciones militares, especialmente en inteligencia, vigilancia y reconocimiento (ISR), la detección de objetos es crucial para identificar y seguir vehículos, personal y equipo enemigos a partir de imágenes de vigilancia aérea y terrestre, lo que mejora la conciencia situacional y la toma de decisiones en el campo de batalla.

La detección de objetos puede no ser tu mejor opción en tareas que requieren entender el contexto o las relaciones entre objetos sin necesitar una localización precisa. Por ejemplo, si el objetivo es simplemente clasificar el contenido general de una imagen, como determinar si una imagen contiene un gato o un perro, los algoritmos de clasificación de imágenes son más apropiados. De forma similar, en tareas que implican segmentar imágenes en regiones según los bordes de los objetos, como distinguir entre distintos tipos de tejido en un estudio médico, los algoritmos de segmentación semántica serían más eficaces. Además, si el objetivo principal es analizar características o patrones estáticos en datos no visuales, como texto o series de tiempo, deben usarse otros algoritmos especializados. La fortaleza de la detección de objetos radica en su capacidad no solo de reconocer objetos, sino también de dar su ubicación exacta, lo que puede ser innecesario en tareas más simples de clasificación o segmentación.

#### Segmentación semántica (_Semantic Segmentation_)

La segmentación semántica es una técnica potente de visión por computadora que consiste en etiquetar cada píxel de una imagen con una etiqueta de clase de un conjunto predefinido de clases. A diferencia de la detección de objetos, que identifica objetos y sus cuadros delimitadores, y de la clasificación de imágenes, que analiza solo imágenes completas clasificándolas en una de varias categorías de salida, la segmentación semántica ofrece una comprensión detallada de la imagen al etiquetar cada píxel según su categoría. Esto la hace invaluable en aplicaciones que requieren una localización y diferenciación precisas de los objetos dentro de una imagen.

Los algoritmos de segmentación semántica suelen usar modelos de deep learning, en particular CNN con arquitecturas especializadas como las redes totalmente convolucionales (FCN), el análisis de escenas piramidal (PSP) y DeepLabV3. Estos modelos parten de una red de clasificación preentrenada, que se modifica para producir mapas de segmentación en lugar de probabilidades de clase. La red procesa la imagen de entrada a través de múltiples capas convolucionales para extraer características en distintos niveles de abstracción. Durante el entrenamiento, el modelo aprende a asignar estas características a etiquetas a nivel de píxel usando una función de pérdida que mide la diferencia entre las etiquetas predichas y las verdaderas. A menudo se emplean técnicas como el sobremuestreo espacial (_upsampling_) y las conexiones de salto (_skip connections_) para mejorar la resolución espacial de la salida, asegurando que el mapa de segmentación represente con exactitud los detalles finos y los bordes de los objetos dentro de la imagen.

Para usar la segmentación semántica en Amazon SageMaker, primero necesitas preparar y etiquetar tu dataset, asegurándote de que cada imagen tenga una máscara de segmentación correspondiente que etiquete cada píxel. Este dataset etiquetado se sube después a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Puedes crear un modelo de segmentación semántica con un _estimator_ de Amazon SageMaker, especificando el algoritmo (como U-Net o DeepLab) y los hiperparámetros necesarios y apuntando a tu dataset. El _estimator_ se encarga del proceso de entrenamiento, aprovechando la infraestructura robusta de Amazon SageMaker para optimizar el modelo a partir de tus datos. Una vez completado el entrenamiento, puedes desplegar el modelo como un endpoint para predicciones en tiempo real o para procesamiento por lotes.

> [!warning] Nota de precisión: U-Net no es una opción (verificado el 25-09-2026)
> El hiperparámetro `algorithm` del algoritmo integrado Semantic Segmentation solo admite `fcn` (el valor por defecto), `psp` y `deeplab` (DeepLabV3), con `backbone` `resnet-50` o `resnet-101`. **U-Net no está disponible.** Los hiperparámetros obligatorios son `num_classes` y `num_training_samples`. En infraestructura, este algoritmo **entrena solo en GPU y en una sola instancia**, y requiere cuatro canales (`train`, `validation`, `train_annotation` y `validation_annotation`, más un `label_map` opcional), donde las máscaras son imágenes PNG en las que el valor de cada píxel es el número de su clase.

###### Casos de uso (_Use Cases_)

La segmentación semántica se aprovecha mejor en escenarios que requieren una comprensión detallada de la imagen, hasta el nivel de píxel. Esto la hace ideal para aplicaciones en imágenes médicas, como segmentar distintos tipos de tejidos, órganos o tumores en resonancias magnéticas o tomografías, lo cual es crucial para un diagnóstico y una planificación del tratamiento exactos. En la conducción autónoma, la segmentación semántica se usa para entender el entorno identificando y diferenciando objetos como carreteras, banquetas, vehículos y peatones. Esta segmentación detallada ayuda a la navegación segura y a la toma de decisiones. Además, en el monitoreo agrícola, la segmentación semántica puede usarse para diferenciar entre cultivos y malezas, evaluar la salud de las plantas y seguir las etapas de crecimiento, lo que contribuye a prácticas agrícolas eficientes. Esta técnica también es valiosa en las imágenes satelitales, donde se crean mapas detallados para identificar distintos tipos de cobertura del suelo, como bosques, áreas urbanas y cuerpos de agua, para el monitoreo ambiental y la planificación urbana.

La segmentación semántica no se recomienda en tareas que no requieren precisión a nivel de píxel o donde el objetivo principal es simplemente clasificar una imagen completa o detectar objetos con cuadros delimitadores. Por ejemplo, si el objetivo es categorizar imágenes en clases amplias, como identificar si una imagen contiene un gato o un perro, los algoritmos de clasificación de imágenes son más apropiados. De forma similar, en aplicaciones que necesitan localizar y clasificar objetos dentro de una imagen, como detectar rostros o vehículos, los algoritmos de detección de objetos son más adecuados, porque proporcionan cuadros delimitadores alrededor de los objetos detectados, pero no requieren anotaciones detalladas a nivel de píxel. Además, la segmentación semántica puede ser menos eficaz en casos donde el costo computacional y la complejidad son prohibitivos, o donde hay pocos datos etiquetados disponibles para el entrenamiento, porque requiere un número significativo de imágenes anotadas para lograr una alta exactitud.

## Criterios para la selección de modelos (_Criteria for Model Selection_)

Elegir el modelo correcto para tu problema de ML implica considerar varios factores críticos. Esta sección describe los criterios clave que te guiarán para seleccionar el mejor modelo para tu caso de uso específico e incluye ejemplos de algoritmos que se ajustan a cada criterio.

- **Exactitud (_Accuracy_).** El objetivo principal de cualquier modelo de ML es lograr una alta exactitud en la tarea dada. Algoritmos como XGBoost y las SVM son conocidos por su alta exactitud en diversas aplicaciones, incluidas la clasificación y la regresión.
- **Interpretabilidad (_Interpretability_).** La interpretabilidad es tan importante como la exactitud, especialmente en los dominios donde entender el proceso de toma de decisiones es esencial. Los modelos Linear Learner y de regresión logística son muy interpretables y ofrecen una visión clara de cómo se hacen las predicciones. Los árboles de decisión también son interpretables, porque representan visualmente el proceso de decisión. Estos modelos equilibran la exactitud con la interpretabilidad, asegurando que sus decisiones puedan ser entendidas fácilmente por las partes interesadas y generen confianza.
- **Escalabilidad (_Scalability_).** La capacidad de un modelo de manejar volúmenes y complejidad de datos crecientes es crítica en las aplicaciones a gran escala. Algoritmos como Linear Learner y el clustering K-means escalan de forma eficiente y pueden manejar datasets grandes. Random Cut Forest también es escalable, lo que lo hace adecuado para la detección de anomalías en escenarios de big data. En términos de SageMaker, esto se traduce en que los tres admiten el modo Pipe (leen los datos en flujo desde S3 sin copiarlos antes) y que Linear Learner y RCF pueden repartir el entrenamiento entre varias instancias.
- **Latencia y velocidad (_Latency and speed_).** En las aplicaciones en tiempo real, la velocidad a la que un modelo puede dar predicciones es crucial. Algoritmos como el _random forest_ ofrecen tiempos de inferencia rápidos, lo que los hace ideales en escenarios donde las decisiones oportunas son críticas, como la detección de fraude y los sistemas de recomendación. La latencia que percibe la aplicación no depende solo del algoritmo: también suman el tipo de instancia del endpoint, el tamaño del modelo, el tiempo de serializar y enviar los datos por la red y la distancia entre la aplicación y el endpoint, por lo que conviene que ambos estén en la misma región. (Recuerda que el _random forest_ no es un algoritmo integrado de SageMaker; se desplegaría con el contenedor de scikit-learn.)
- **Requisitos de recursos (_Resource requirements_).** Los recursos computacionales necesarios para el entrenamiento y la inferencia pueden afectar la viabilidad de usar ciertos modelos. Los _linear learners_, k-NN y PCA generalmente requieren menos recursos que las redes neuronales profundas. Evaluar los requisitos de recursos asegura que los modelos se ajusten a tu infraestructura y tu presupuesto disponibles. En la práctica, la frontera más importante es la que separa los algoritmos que pueden entrenar en CPU de los que exigen GPU (los de imagen y Seq2Seq): una instancia con GPU cuesta por hora del orden de varias veces a más de diez veces lo que una instancia de CPU de propósito general, según la familia y el tamaño (consulta la página de precios vigente de SageMaker AI).
- **Disponibilidad y calidad de los datos (_Data availability and quality_).** La disponibilidad y la calidad de los datos influyen significativamente en el rendimiento del modelo. Algoritmos como LDA y BlazingText rinden bien con datasets de texto grandes, y K-means puede funcionar eficazmente con datasets más pequeños en tareas de clustering. Asegurar datos de alta calidad para el entrenamiento es crítico para desarrollar modelos exactos y confiables.
- **Consideraciones regulatorias y éticas (_Regulatory and ethical considerations_).** En muchas industrias, los modelos deben cumplir requisitos regulatorios y estándares éticos. La regresión logística y los árboles de decisión suelen preferirse en las industrias reguladas por su transparencia y facilidad de explicación. Las consideraciones éticas son particularmente importantes en campos como la salud y las finanzas, donde las implicaciones de las predicciones del modelo pueden tener consecuencias significativas.
- **Costo (_Cost_).** El costo de implementar y mantener distintos modelos puede variar mucho. Modelos como Linear Learner y el _random forest_ suelen ser rentables, mientras que los modelos de deep learning como las CNN pueden generar costos más altos por sus demandas computacionales. Considerar las implicaciones de costo, incluidos los recursos de la nube y el etiquetado de datos, es esencial en los proyectos de largo plazo. En la nube, el costo de un modelo tiene tres componentes que conviene separar: el **entrenamiento** (horas de instancia de cada trabajo, multiplicadas por cuántas veces reentrenas), la **inferencia** (horas de instancia de los endpoints mientras existan, o duración de los trabajos por lotes) y el **almacenamiento** (datos y modelos en S3). En un modelo que sirve predicciones todo el año, la inferencia suele pesar más que el entrenamiento. El capítulo 5 muestra formas de abaratar el entrenamiento, como las instancias Spot administradas.

Suele haber un compromiso entre estos factores; por ejemplo, entre la exactitud y la interpretabilidad, donde los modelos complejos que logran una alta exactitud pueden ser difíciles de entender, mientras que los modelos más simples, con mejor interpretabilidad, pueden tener menor exactitud. Tu trabajo como ingeniero de ML en AWS es equilibrar estos compromisos para satisfacer las necesidades y restricciones específicas de tu problema de ML, asegurando que el modelo elegido se alinee con los objetivos de negocio y los requisitos operativos. Esto implica evaluar y ajustar continuamente el modelo en función de datos y retroalimentación nuevos, para lograr el mejor rendimiento posible manteniendo la transparencia y la confianza.

Al considerar estos criterios y entender qué algoritmos se alinean con ellos, estarás bien preparado para seleccionar el modelo más apropiado para tu problema de ML. Este enfoque integral asegura que tus modelos no solo tengan un alto rendimiento, sino que también sean interpretables, escalables y estén alineados con tus requisitos específicos.

## Escenarios donde este servicio es la opción obligada

Este capítulo no trata de un solo servicio, sino de nueve servicios de IA y diecisiete algoritmos integrados. Por eso cada escenario se centra en uno distinto de los que aparecen en el capítulo, elegido porque, dadas las restricciones del caso, es la única opción razonable dentro de AWS. Los tres cubren las tres formas de consumir ML que recorre el capítulo: un modelo fundacional por API (Bedrock), un servicio de IA preentrenado (Rekognition) y un algoritmo integrado que entrenas tú (IP Insights). Las empresas son ficticias; las restricciones son las que aparecen en proyectos reales.

### Escenario 1: resúmenes de siniestros en una aseguradora → Amazon Bedrock

**Contexto.** Una aseguradora de autos de Estados Unidos recibe unas 5 000 reclamaciones al día. Cada expediente incluye la narración del ajustador, el reporte policial y la póliza, y el equipo de siniestros quiere que un modelo genere un resumen del caso, extraiga los datos clave a campos estructurados y redacte un borrador de carta al asegurado. Un piloto de seis meses con un modelo Claude de Anthropic, usado mediante la API pública de Anthropic, fue validado por las áreas legal y de cumplimiento. Ahora hay que llevarlo a producción dentro de la plataforma de la empresa: la aplicación de siniestros corre en contenedores sobre **Amazon ECS** (el servicio de AWS que ejecuta y reinicia contenedores en un grupo de máquinas) en **subredes privadas** de una VPC, es decir, en segmentos de la red cuyos recursos no son accesibles desde internet ni tienen salida directa hacia él. El uso es muy irregular: picos los lunes por la mañana, casi nada de noche y en fines de semana.

**Restricciones clave.**

1. **Modelo:** debe ser un modelo Claude, la familia validada en el piloto. Cambiar de familia obligaría a repetir tres meses de validación.
2. **Red:** la aplicación vive en subredes privadas sin salida a internet, y la política de seguridad prohíbe abrirle una (por ejemplo, con una puerta de enlace NAT, el componente que permite a máquinas privadas iniciar conexiones hacia internet).
3. **Datos:** los expedientes contienen datos personales. El contrato con los reaseguradores exige que no se usen para entrenar modelos de terceros y que se procesen dentro de Estados Unidos.
4. **Operación:** no hay equipo para operar GPU, y la dirección no quiere costos fijos en las horas en que nadie usa el sistema.
5. **Auditoría:** cada invocación debe poder atribuirse a la aplicación que la hizo, y el texto de las peticiones y respuestas debe conservarse para auditoría.

**Por qué este servicio.**

- (1) → Dentro de AWS, los modelos Claude se ofrecen en Bedrock como modelos serverless; Anthropic no distribuye los pesos de sus modelos, así que no hay forma de alojarlos por cuenta propia. Solo hay que enviar una vez el formulario de caso de uso de Anthropic antes de la primera invocación.
- (2) → Bedrock admite un **endpoint de VPC de interfaz** para `bedrock-runtime`, basado en **AWS PrivateLink**: AWS coloca dentro de la subred privada una interfaz de red con una IP privada que da acceso al servicio, de modo que el tráfico entre la aplicación y Bedrock nunca sale a internet ni necesita NAT.
- (3) → Según la documentación de Bedrock, las peticiones y respuestas no se usan para entrenar modelos ni se comparten con el proveedor del modelo. Un **perfil de inferencia geográfico** de Estados Unidos (identificadores con el prefijo `us.`) reparte los picos de los lunes entre varias regiones sin salir de ese país.
- (4) → En la modalidad bajo demanda se paga por token procesado: sin instancias, sin capacidad que aprovisionar y sin costo en las horas sin uso.
- (5) → Una política de IAM sobre el rol de las tareas de ECS limita `bedrock:InvokeModel` a los ARN concretos del modelo y del perfil de inferencia. **AWS CloudTrail**, el servicio que registra el historial de llamadas a la API de AWS en la cuenta, deja constancia de quién invocó qué y cuándo, y el **registro de invocaciones de modelos** (_model invocation logging_) de Bedrock, que es opcional, guarda el texto completo de peticiones y respuestas en S3 o en CloudWatch Logs.

**Por qué no las alternativas.**

- **Endpoints de SageMaker AI (incluido JumpStart):** solo alojan modelos cuyos pesos se pueden desplegar, y Claude no está entre ellos (restricción 1); además, las instancias se cobran por hora aunque no haya peticiones (restricción 4).
- **Amazon EC2 con GPU**, alojando un modelo por cuenta propia: tiene el mismo problema de disponibilidad del modelo y suma toda la operación de las GPU (restricciones 1 y 4).
- **Amazon Comprehend:** extrae entidades, frases clave y sentimiento, pero no genera resúmenes ni redacta cartas; no resuelve la tarea.
- **La API pública de Anthropic**, como en el piloto: exige salida a internet desde las subredes privadas (restricción 2) y deja el tratamiento de los datos fuera de los controles y registros de la cuenta de AWS (restricción 5).

### Escenario 2: alta de clientes en un banco digital → Amazon Rekognition

**Contexto.** Un banco digital abre cuentas desde su app para iOS y Android, sin sucursales. El regulador exige verificar a distancia que quien abre la cuenta es una persona real y es el titular del documento de identidad que presenta. Además, el área de fraude detectó personas que abren varias cuentas con identidades distintas (el mismo rostro con nombres diferentes) y quiere detectarlas en el momento del alta. El banco tiene unos 3 millones de clientes.

**Restricciones clave.**

1. **Prueba de vida:** la normativa exige comprobar que frente a la cámara hay una persona presente, y no una foto impresa, un video reproducido en otra pantalla o una máscara.
2. **Comparaciones:** 1:1, la selfie contra la foto del documento; y 1:N, la selfie contra los rostros de los 3 millones de clientes existentes, con respuesta en pocos segundos durante el alta.
3. **Sin equipo de visión:** el banco no tiene imágenes de rostros etiquetadas ni un equipo capaz de entrenar y mantener modelos biométricos, y la dirección exige una solución administrada.
4. **Privacidad:** debe poder borrar los datos biométricos de un cliente cuando este lo solicite, y las imágenes deben quedarse en almacenamiento del propio banco, cifradas con sus claves.
5. **Plazo:** la integración en las apps móviles debe estar lista en semanas.

**Por qué este servicio.**

- (1) → **Rekognition Face Liveness**: el backend del banco crea una sesión (`CreateFaceLivenessSession`), la app captura un video corto con un componente de interfaz ya hecho y el backend recibe una puntuación de confianza y una imagen de referencia (`GetFaceLivenessSessionResults`). El umbral de aceptación lo fija el banco.
- (2) → `CompareFaces` resuelve la comparación 1:1. Para la 1:N, los rostros de los clientes se indexan en una **colección** (`IndexFaces`) y cada selfie nueva se busca en ella (`SearchFacesByImage`). Según las cuotas vigentes, una colección admite hasta 20 millones de vectores de rostro, holgura suficiente para 3 millones de clientes.
- (3) → Es un servicio preentrenado que se consume por API: no hay datos que etiquetar, ni modelos que entrenar, ni endpoints que operar.
- (4) → La colección guarda vectores de rasgos faciales, no fotos, y `DeleteFaces` elimina el vector de un cliente. Las imágenes originales se quedan en el bucket de S3 del banco, cifradas con sus claves de KMS. Con la política de exclusión de servicios de IA de AWS Organizations, el banco impide además que AWS use ese contenido para mejorar el servicio.
- (5) → **AWS Amplify** ofrece el componente de captura de Face Liveness ya construido para web, Android e iOS.

**Por qué no las alternativas.**

- **Algoritmos integrados de SageMaker (Image Classification, Object Detection):** exigirían imágenes de rostros etiquetadas y construir un modelo biométrico propio, y no incluyen prueba de vida (restricciones 1 y 3).
- **Amazon Textract (`AnalyzeID`):** extrae los datos del documento de identidad (nombre, fecha de nacimiento, número), pero no compara rostros. Es un complemento, no un sustituto.
- **Modelos multimodales de Amazon Bedrock:** son de propósito general; no ofrecen colecciones de rostros para búsquedas 1:N entre millones de personas ni sesiones de prueba de vida (restricciones 1 y 2).
- **Amazon Cognito:** gestiona el registro, el inicio de sesión y el segundo factor de los usuarios, pero no verifica un rostro contra un documento. También es un complemento.

Una advertencia que el escenario no puede omitir: el reconocimiento facial está regulado en muchas jurisdicciones (consentimiento expreso, finalidad limitada, plazos de conservación). AWS publica una **AI Service Card** para la comparación de rostros de Rekognition con recomendaciones de uso responsable, y lo prudente es enviar a revisión humana los casos con puntuaciones cercanas al umbral.

### Escenario 3: robo de cuentas en un videojuego en línea → Amazon SageMaker IP Insights

**Contexto.** Un estudio de videojuegos tiene 8 millones de cuentas en un juego en línea con artículos virtuales que se venden por dinero real. Sufre ataques de robo de cuentas: los atacantes prueban combinaciones de usuario y contraseña filtradas en otros sitios (_credential stuffing_) y, cuando aciertan, vacían el inventario. El estudio quiere asignar a cada inicio de sesión una puntuación de riesgo y pedir verificación por correo o un segundo factor solo cuando el acceso sea inusual para esa cuenta. Tiene 18 meses de registros de inicio de sesión en S3 (identificador de cuenta, dirección IP del cliente y fecha), pero ninguna marca de cuáles fueron fraudulentos. Su servicio de autenticación es propio, corre en contenedores detrás de un balanceador de carga y está integrado con las plataformas de consolas.

**Restricciones clave.**

1. **Sin etiquetas:** no hay ejemplos marcados de inicios de sesión fraudulentos.
2. **Riesgo por cuenta, no por IP:** una IP de Brasil es normal para un jugador brasileño y sospechosa para una cuenta que siempre entra desde Madrid. Una lista global de «IP malas» no distingue esos casos.
3. **Identidad propia:** el sistema de autenticación no se puede migrar a un servicio de identidad administrado, por los contratos y las integraciones con las plataformas de consolas.
4. **Latencia:** la puntuación debe calcularse dentro del flujo de inicio de sesión, con un presupuesto de decenas de milisegundos.
5. **Red solo IPv4:** el balanceador de carga y los servidores de juego solo aceptan IPv4, así que todos los inicios de sesión llegan con una dirección IPv4.
6. **Equipo:** un equipo de ML pequeño que ya usa SageMaker y quiere reentrenar el modelo cada semana.

**Por qué este servicio.**

- (1) → IP Insights es **no supervisado**: se entrena directamente con pares (cuenta, IPv4) extraídos de los registros, en un CSV de dos columnas sin encabezado, sin necesidad de etiquetas.
- (2) → Aprende la asociación entre **cada entidad** y las direcciones IP, y devuelve una puntuación por par (cuenta, IP): justo la pregunta de si esa IP es habitual para esa cuenta.
- (3) → No depende del proveedor de identidad: el servicio de autenticación propio solo tiene que llamar al endpoint con el par antes de aceptar el acceso.
- (4) → Se despliega en un endpoint de SageMaker en tiempo real, sobre instancias de CPU, como recomienda la documentación para la inferencia, y en la misma región y VPC que el servicio de autenticación.
- (5) → La limitación de IP Insights a IPv4 no afecta a esta red.
- (6) → El reentrenamiento semanal es un trabajo de entrenamiento más (con GPU, como recomienda la documentación), que se puede automatizar con el resto de la canalización de SageMaker.

**Por qué no las alternativas.**

- **Amazon GuardDuty:** detecta amenazas contra la **cuenta de AWS** y sus recursos, analizando CloudTrail, VPC Flow Logs y registros de DNS; no ve los inicios de sesión de los jugadores en la aplicación (restricción 2).
- **Protección contra amenazas de Amazon Cognito** (autenticación adaptativa, en el plan Plus): puntúa el riesgo de cada inicio de sesión, pero solo para usuarios de un grupo de usuarios de Cognito (restricción 3).
- **Amazon Fraud Detector:** tenía un tipo de modelo para robo de cuentas, pero no admite clientes nuevos desde el 07-11-2025.
- **AWS WAF**, el firewall de aplicaciones web de AWS, incluida su regla administrada de prevención de robo de cuentas: inspecciona las peticiones de inicio de sesión con reglas (credenciales filtradas conocidas, volumen de intentos por IP), pero no aprende qué IP son habituales para cada cuenta (restricción 2). Complementa a IP Insights, no lo sustituye.
- **Random Cut Forest:** detecta anomalías en vectores numéricos, pero no modela de forma nativa la asociación entre entidad e IP; habría que diseñar a mano variables por cuenta, que es justo lo que IP Insights hace por sí mismo (restricción 2).

## Resumen (_Summary_)

En este capítulo exploramos diversos servicios de IA de AWS y los algoritmos integrados de Amazon SageMaker que pueden aprovecharse en distintos casos de uso de ML.

Los servicios de IA de AWS ofrecen una gama de modelos de ML preconstruidos y preentrenados, diseñados para simplificar y acelerar el despliegue de la IA en tus aplicaciones. Estos servicios incluyen Amazon Rekognition para el análisis de imágenes y video, Amazon Comprehend para el NLP, Amazon Polly para la conversión de texto a voz, Amazon Lex para crear interfaces conversacionales, Amazon Textract para extraer texto y datos de documentos, Amazon Transcribe para convertir voz a texto, Amazon Translate para la traducción en tiempo real y Amazon Personalize para crear experiencias de usuario personalizadas. Amazon Bedrock, con los nuevos FMs Nova, también ofrece capacidades robustas para tareas de IA generativa, lo que mejora la flexibilidad y la potencia de tus aplicaciones de IA. Estos servicios de IA son totalmente administrados, requieren una experiencia mínima en ML y son ideales para los desarrolladores que buscan integrar capacidades de IA rápidamente en sus aplicaciones.

Para una mayor flexibilidad y tareas de ML más especializadas, Amazon SageMaker ofrece una amplia gama de algoritmos integrados y soporte para frameworks populares como TensorFlow, PyTorch y MXNet, lo que te permite adaptar tus modelos de ML para cumplir requisitos específicos y lograr un rendimiento optimizado.

Exploramos los algoritmos de ML supervisado de Amazon SageMaker, diseñados para tareas que usan datos etiquetados para el entrenamiento. Los ejemplos incluyeron Linear Learner para regresión y clasificación, XGBoost para _gradient boosting_ sobre árboles de decisión, k-NN para clasificación y regresión, y Factorization Machines para sistemas de recomendación y tareas de clasificación a gran escala. Estos modelos aprenden de los pares de entrada y salida de los datos de entrenamiento para hacer predicciones con datos nuevos, no vistos.

Después conocimos los algoritmos integrados de ML no supervisado de Amazon SageMaker, diseñados para descubrir patrones y estructuras ocultos en datos no etiquetados. Estos algoritmos incluyen K-Means para agrupar puntos de datos similares en grupos distintos, PCA para la reducción de dimensionalidad, LDA y NTM para el modelado de temas, y RCF e IP Insights para la detección de anomalías.

Más adelante en el capítulo se presentaron algoritmos más avanzados que aprovechan redes neuronales profundas para el análisis textual y el procesamiento de imágenes. Entre ellos estaban los algoritmos integrados de Amazon SageMaker BlazingText, para _word embeddings_ y clasificación de textos eficientes; Seq2Seq, para tareas de secuencia a secuencia como la traducción automática y el resumen de textos; Image Classification, para categorizar imágenes en clases predefinidas; Object Detection, para identificar y localizar múltiples objetos dentro de una imagen, y Semantic Segmentation, para la clasificación a nivel de píxel del contenido de una imagen. Estos modelos de deep learning están diseñados para manejar tareas complejas con alta exactitud, lo que permite soluciones de IA robustas en diversos dominios, como el NLP, la visión por computadora y los sistemas autónomos.

El capítulo concluyó con una lista seleccionada de los criterios que debes considerar al seleccionar tu modelo, incluidos la exactitud, la interpretabilidad, la escalabilidad, la latencia, los requisitos de recursos, la disponibilidad de datos, los aspectos regulatorios y el costo.

## Puntos esenciales para el examen (_Exam Essentials_)

**Conoce los servicios de IA de AWS para visión.** AWS ofrece dos servicios de IA orientados a tareas de visión. Amazon Rekognition permite el análisis de imágenes y video, lo que permite a los usuarios detectar objetos, rostros, puntos de referencia y texto, así como reconocer celebridades y analizar sentimientos. Para el procesamiento de documentos, Amazon Textract extrae texto, tablas y otros datos de documentos escaneados, lo que facilita digitalizar y gestionar los flujos de trabajo documentales de forma eficiente.

> [!warning] Nota de precisión: Rekognition no analiza «sentimientos»
> Lo que Rekognition devuelve es una predicción de la **emoción aparente según la expresión facial** (el campo `Emotions` de `DetectFaces`), con su nivel de confianza. La documentación de AWS advierte expresamente que no es una determinación del estado emocional interno de la persona y que no debe usarse así. El análisis de sentimiento de un **texto** es tarea de Comprehend.

**Conoce los servicios de IA de AWS para voz y chatbots.** AWS ofrece dos servicios de IA para tareas relacionadas con la voz. Amazon Polly ofrece capacidades de texto a voz, transformando el contenido escrito en voz de sonido natural en múltiples idiomas. Amazon Transcribe ofrece reconocimiento automático de voz (ASR), convirtiendo grabaciones de audio en transcripciones de texto exactas. Amazon Lex permite crear interfaces conversacionales, combinando el ASR y la comprensión del lenguaje natural (NLU) para impulsar chatbots y asistentes de voz.

**Conoce los servicios de IA de AWS para lenguaje.** AWS ofrece dos servicios de IA para tareas de procesamiento de lenguaje. Amazon Comprehend permite el procesamiento de lenguaje natural (NLP) extrayendo información del texto, como el análisis de sentimiento, el reconocimiento de entidades y la extracción de frases clave. Amazon Translate ofrece servicios de traducción en tiempo real y por lotes, lo que permite una comunicación multilingüe fluida.

**Conoce los servicios de IA de AWS para IA generativa.** AWS ofrece Amazon Bedrock, con una amplia selección de modelos fundacionales (FMs), incluidos los recién lanzados FMs Nova. Amazon Bedrock ofrece una plataforma robusta para crear y desplegar aplicaciones de IA generativa, lo que permite a los desarrolladores construir soluciones sofisticadas como la generación de texto, la síntesis de imágenes, los _embeddings_ y más. Estos servicios permiten a los usuarios aprovechar con facilidad modelos de IA generativa de última generación, habilitando aplicaciones innovadoras y creativas en diversos dominios.

**Conoce la diferencia entre los algoritmos de clasificación y de regresión.** Los algoritmos de clasificación se usan para categorizar datos en clases o etiquetas distintas, como determinar si un correo electrónico es spam. Los algoritmos de regresión, en cambio, predicen valores numéricos continuos, como pronosticar precios de acciones o estimar el valor de viviendas a partir de diversas características.

**Conoce los algoritmos lineales que ofrece Amazon SageMaker.** Amazon SageMaker ofrece algoritmos Linear Learner para tareas tanto de clasificación como de regresión. Entre ellos están la regresión lineal, la regresión logística y las máquinas de vectores de soporte. Estos algoritmos son muy interpretables, lo que los hace ideales en aplicaciones donde entender la contribución de cada característica a las predicciones es crítico.

> [!warning] Nota de precisión
> Linear Learner es un único algoritmo integrado; la regresión lineal, la logística y la SVM lineal se eligen con `predictor_type` y `loss` (ver la nota en la sección de Linear Learner). No hay SVM con kernels.

**Conoce el algoritmo k-NN y cuándo usarlo.** El algoritmo de los k vecinos más cercanos es un algoritmo de ML supervisado simple pero potente para tareas de clasificación y regresión, en el que las predicciones se hacen a partir de los puntos de datos más cercanos en el espacio de características. Es un algoritmo integrado de Amazon SageMaker, particularmente útil cuando la frontera de decisión es compleja y no lineal, y cuando la interpretabilidad es importante, porque el razonamiento detrás de cada predicción queda claro al examinar los vecinos más cercanos. k-NN se usa mejor con datasets más pequeños, por su complejidad computacional, y en casos donde tienes una comprensión clara de la estructura y las relaciones de los datos.

> [!note]
> En la sección de k-NN, el libro decía que es más adecuado cuando «las fronteras de decisión no son demasiado complejas»; este punto dice lo contrario. Se conservan ambas afirmaciones tal como aparecen en el original.

**Conoce los algoritmos supervisados basados en árboles de decisión que ofrece Amazon SageMaker y cuándo usarlos.** Amazon SageMaker ofrece potentes algoritmos supervisados basados en árboles de decisión, como Random Forest y XGBoost. Random Forest es valioso por su capacidad de reducir el sobreajuste y mejorar la exactitud creando un ensamble de árboles de decisión y promediando sus predicciones, lo que lo hace adecuado para tareas con un gran número de características de entrada. XGBoost es conocido por su eficiencia computacional y su rendimiento, en particular con datasets grandes y modelos complejos, y suele usarse en tareas de clasificación y regresión que requieren alta exactitud predictiva y velocidad.

> [!warning] Nota de precisión
> Random Forest **no** es un algoritmo integrado de SageMaker (ver la nota en la sección de Random Forest). Los algoritmos integrados basados en árboles son XGBoost, LightGBM y CatBoost. No confundas Random Forest con Random Cut Forest, que es de detección de anomalías.

**Conoce los algoritmos de clustering que ofrece Amazon SageMaker.** Amazon SageMaker ofrece el algoritmo no supervisado K-Means para clustering, que no debe confundirse con el algoritmo supervisado k-NN, pensado para tareas de clasificación y regresión.

**Conoce los algoritmos de reducción de dimensionalidad que ofrece Amazon SageMaker.** Amazon SageMaker ofrece el algoritmo no supervisado de análisis de componentes principales (PCA) para la reducción de dimensionalidad, útil para reducir el número de características de un dataset conservando la mayor variabilidad posible.

**Conoce los algoritmos de modelado de temas que ofrece Amazon SageMaker.** Amazon SageMaker ofrece los algoritmos no supervisados de asignación latente de Dirichlet (LDA) y el modelo neuronal de temas (NTM) para casos de uso de modelado de temas, lo que permite a los usuarios descubrir temas ocultos en grandes colecciones de datos textuales sin necesidad de datos etiquetados.

**Conoce los algoritmos de detección de anomalías que ofrece Amazon SageMaker.** Amazon SageMaker ofrece los algoritmos no supervisados Random Cut Forest e IP Insights para casos de uso de detección de anomalías, particularmente útiles para identificar patrones o comportamientos inusuales en los datos, como la detección de actividades fraudulentas o de intrusiones en la red.

**Conoce los algoritmos de análisis textual que ofrece Amazon SageMaker.** Amazon SageMaker ofrece los algoritmos BlazingText y Sequence-to-Sequence para casos de uso de análisis textual, lo que permite manejar eficientemente datos textuales a gran escala y da soporte a diversas tareas de procesamiento de lenguaje natural, como la clasificación de textos, la traducción y el resumen.

**Conoce los algoritmos de procesamiento de imágenes que ofrece Amazon SageMaker.** Amazon SageMaker ofrece los algoritmos Image Classification, Object Detection y Semantic Segmentation para casos de uso de procesamiento de imágenes, lo que permite a los desarrolladores construir, entrenar y desplegar modelos de machine learning capaces de analizar y entender datos visuales, como reconocer objetos, clasificar imágenes en categorías y segmentar imágenes a nivel de píxel para un análisis detallado.

> [!tip] Lo que añaden estas notas para el examen
> Además de «qué algoritmo resuelve qué problema», que es lo que resumen los puntos anteriores, las preguntas de configuración suelen girar en torno a cuatro datos de infraestructura, reunidos en la tabla del inicio de la sección de algoritmos integrados: el **formato de entrada** (Factorization Machines solo recordIO-protobuf; Object2Vec, JSON Lines; DeepAR, JSON Lines o Parquet; BlazingText, texto con una oración por línea; CSV siempre sin encabezado y con la etiqueta primero), el **modo de entrada** (Pipe frente a File), el **tipo de instancia** (los de imagen y Seq2Seq solo con GPU; RCF solo con CPU; DeepAR solo CPU para inferencia) y si el algoritmo **admite varias instancias** (LDA, Seq2Seq, Object2Vec y Semantic Segmentation no; BlazingText solo en el modo `batch_skipgram`).

## Preguntas de repaso (_Review Questions_)

1. ¿Qué servicio de IA de AWS usarías para analizar grandes volúmenes de texto no estructurado y extraer información como el reconocimiento de entidades y el análisis de sentimiento?
   - A. Amazon Textract
   - B. Amazon Lex
   - C. Amazon Comprehend
   - D. Amazon Polly

2. ¿Qué servicio de IA de AWS está diseñado específicamente para aplicaciones de IA generativa y ofrece una plataforma robusta para crear texto, imágenes y otros productos creativos?
   - A. Amazon Rekognition
   - B. Amazon Bedrock
   - C. Amazon Translate
   - D. Amazon Transcribe

3. Para detectar objetos y personas en transmisiones de video en tiempo real, ¿qué servicio de IA de AWS sería el más apropiado?
   - A. Amazon Textract
   - B. Amazon Lex
   - C. Amazon Rekognition
   - D. Amazon Comprehend

4. ¿Qué servicio de AWS ofrece una solución altamente eficiente y escalable para la reducción de dimensionalidad en datasets grandes?
   - A. Amazon Translate
   - B. Amazon Polly
   - C. Principal Component Analysis (PCA) en Amazon SageMaker
   - D. K-Means en Amazon SageMaker

5. ¿Qué algoritmo de Amazon SageMaker es particularmente útil para tareas de clasificación y regresión por su eficiencia y su alto rendimiento?
   - A. Random Forest
   - B. XGBoost
   - C. k-Nearest Neighbors
   - D. Principal Component Analysis

6. ¿Qué algoritmo de ML supervisado que ofrece Amazon SageMaker es ideal para reducir el sobreajuste promediando múltiples árboles de decisión?
   - A. Linear Learner
   - B. BlazingText
   - C. Random Forest
   - D. Latent Dirichlet Allocation

7. ¿Para qué tipo de tareas es particularmente adecuado el algoritmo Linear Learner de Amazon SageMaker?
   - A. Clasificación de textos
   - B. Clustering
   - C. Clasificación y regresión
   - D. Detección de anomalías

8. ¿Qué algoritmo usarías en Amazon SageMaker si necesitas resultados muy interpretables en un modelo de relación lineal?
   - A. Random Cut Forest
   - B. Linear Learner
   - C. Neural Topic Model
   - D. DeepAR

9. ¿Qué algoritmo de ML supervisado de Amazon SageMaker está diseñado específicamente para el pronóstico de series de tiempo?
   - A. IP Insights
   - B. DeepAR
   - C. Neural Topic Model
   - D. Sequence-to-Sequence

10. ¿Qué algoritmo de ML supervisado de Amazon SageMaker es conocido por potenciar (_boosting_) aprendices débiles para crear un modelo predictivo fuerte?
    - A. K-Means
    - B. Latent Dirichlet Allocation
    - C. XGBoost
    - D. Random Cut Forest

11. Para tu problema de ML necesitas un algoritmo no supervisado y muy interpretable de Amazon SageMaker para reducir la dimensionalidad de los datos preservando la varianza máxima. ¿Qué algoritmo integrado usarías?
    - A. K-Means
    - B. Random Cut Forest
    - C. Principal Component Analysis
    - D. Neural Topic Model

12. Para detectar eventos raros y anomalías con alta exactitud en flujos de datos, ¿qué algoritmo no supervisado de Amazon SageMaker elegirías?
    - A. Latent Dirichlet Allocation
    - B. IP Insights
    - C. Random Cut Forest
    - D. Factorization Machines

13. Cuando necesitas descubrir temas ocultos en datasets de texto grandes con alta interpretabilidad y exactitud, ¿qué algoritmo de Amazon SageMaker seleccionarías?
    - A. K-Means
    - B. Latent Dirichlet Allocation
    - C. Principal Component Analysis
    - D. Random Cut Forest

14. Para agrupar con exactitud datasets grandes en grupos predefinidos según la similitud de las características, ¿qué algoritmo muy interpretable de Amazon SageMaker usarías?
    - A. Random Cut Forest
    - B. Principal Component Analysis
    - C. K-Means
    - D. Neural Topic Model

15. ¿Qué algoritmo de Amazon SageMaker es conocido por su alta exactitud y rendimiento en tareas de clasificación de textos a gran escala, siendo además rentable?
    - A. BlazingText
    - B. Sequence-to-Sequence
    - C. Latent Dirichlet Allocation
    - D. IP Insights

16. Identifica el algoritmo de Amazon SageMaker que ofrece alto rendimiento y exactitud en tareas de traducción de textos, siendo además rentable e interpretable.
    - A. Random Cut Forest
    - B. Sequence-to-Sequence
    - C. BlazingText
    - D. Principal Component Analysis

17. Para la identificación y localización de alta exactitud de múltiples objetos dentro de una imagen, ¿qué algoritmo de Amazon SageMaker destaca en rendimiento y eficiencia de costos?
    - A. Image Classification
    - B. Object Detection
    - C. Semantic Segmentation
    - D. Factorization Machines

18. ¿Qué algoritmo de Amazon SageMaker ofrece alta exactitud y rentabilidad para clasificar imágenes en categorías predefinidas, asegurando interpretabilidad y rendimiento?
    - A. Latent Dirichlet Allocation
    - B. Image Classification
    - C. Object Detection
    - D. IP Insights

19. Identifica el algoritmo de Amazon SageMaker que permite un análisis detallado de las imágenes a nivel de píxel, ofreciendo alta exactitud e interpretabilidad y siendo eficiente en rendimiento.
    - A. Random Cut Forest
    - B. Image Classification
    - C. Semantic Segmentation
    - D. BlazingText

20. ¿Qué algoritmo de Amazon SageMaker utiliza _word embeddings_ para tareas de procesamiento de lenguaje natural, equilibrando exactitud, rentabilidad, interpretabilidad y rendimiento?
    - A. Random Cut Forest
    - B. Principal Component Analysis
    - C. Latent Dirichlet Allocation
    - D. BlazingText

> [!warning] Nota sobre las preguntas 5 y 6
> Ambas incluyen Random Forest como algoritmo de SageMaker, y la 6 lo presenta como «algoritmo de ML supervisado que ofrece Amazon SageMaker». Como se explicó en la sección correspondiente, no existe un Random Forest integrado; en SageMaker se entrenaría en modo script con scikit-learn. En un examen oficial actualizado no deberías encontrar esa premisa, pero si aparece, ten presente la diferencia con Random Cut Forest.

## Glosario

- **Aceleración por hardware.** Uso de GPU para ejecutar partes del cálculo mucho más rápido que en CPU; en BlazingText, mediante código escrito en CUDA.
- **AI Service Card.** Documento de AWS que describe los usos previstos, las limitaciones y las recomendaciones de uso responsable de un servicio de IA.
- **Amplify (AWS Amplify).** Conjunto de bibliotecas y componentes de AWS para aplicaciones web y móviles; incluye el componente de captura de Face Liveness.
- **API.** Punto de acceso HTTPS regional de un servicio, al que una aplicación envía peticiones firmadas y del que recibe respuestas en JSON.
- **API síncrona / asíncrona.** En la síncrona, la respuesta llega en la misma conexión; en la asíncrona, se lanza un trabajo que procesa en segundo plano y deja los resultados en S3.
- **Aprovisionar.** Reservar y arrancar los recursos (instancias, capacidad) que necesita un trabajo o un servicio.
- **ARN (Amazon Resource Name).** Identificador único de un recurso de AWS, usado en las políticas de IAM para indicar sobre qué recurso se concede un permiso.
- **Athena (Amazon Athena).** Motor SQL serverless que consulta archivos directamente en S3 y cobra por los datos escaneados.
- **Backend.** Parte de una aplicación que el usuario no ve: bases de datos, lógica de negocio e integraciones con otros sistemas.
- **Bedrock Marketplace (Amazon Bedrock Marketplace).** Catálogo de modelos especializados de Bedrock que se despliegan en endpoints de SageMaker y se cobran por hora de instancia.
- **boto3 / botocore.** SDK de AWS para Python y la biblioteca de bajo nivel sobre la que funciona; botocore espera 60 segundos por defecto una respuesta.
- **Bucket predeterminado de SageMaker.** Bucket `sagemaker-<región>-<id de cuenta>` que el SDK crea para guardar datos y modelos de ejemplo.
- **Campaña (Personalize).** Endpoint en tiempo real de una versión de solución de Personalize, con capacidad mínima aprovisionada que se cobra por hora.
- **Canal (channel).** Nombre, como `train` o `validation`, asociado a una ubicación de datos y un tipo de contenido dentro de un trabajo de entrenamiento.
- **Centro de contacto (contact center).** Infraestructura con la que una empresa atiende llamadas y chats de clientes: números, colas, enrutamiento a agentes y grabación.
- **CGNAT (carrier-grade NAT).** Técnica de los operadores para que muchos clientes compartan una misma dirección IPv4 pública.
- **Ciclo de vida de modelos (Bedrock).** Estados Active, Legacy y EOL de un modelo; en Legacy no admite clientes nuevos y después del EOL deja de responder.
- **CloudTrail (AWS CloudTrail).** Servicio que registra el historial de llamadas a la API de AWS en una cuenta, útil para auditoría.
- **CloudWatch Logs (Amazon CloudWatch Logs).** Servicio que centraliza los registros de las aplicaciones y de los servicios de AWS.
- **Cognito (Amazon Cognito).** Servicio de registro e inicio de sesión de usuarios para aplicaciones, con segundo factor y protección contra amenazas en su plan Plus.
- **Cold start.** Retraso de la primera invocación de una función Lambda después de un periodo inactivo.
- **Colección de rostros (Rekognition).** Contenedor del lado de AWS con vectores de rasgos faciales indexados, sobre el que se hacen búsquedas 1:N.
- **Compliance (cumplimiento normativo).** Capacidad de demostrar ante auditores o reguladores que se cumplen normas legales o contractuales.
- **Comprehend Medical (Amazon Comprehend Medical).** Servicio aparte de Comprehend, especializado en extraer entidades médicas de texto clínico.
- **Connect (Amazon Connect).** Servicio de centro de contacto en la nube de AWS, en el que un bot de Lex puede atender el primer tramo de las llamadas.
- **Contenedor / imagen de contenedor.** Paquete con un programa y todas sus dependencias que se ejecuta igual en cualquier máquina; la imagen es la plantilla desde la que se arranca.
- **Converse API / InvokeModel.** Formas de invocar un modelo en Bedrock: Converse, con un formato de mensajes común a todos los modelos; InvokeModel, con el formato propio de cada proveedor.
- **CreateTrainingJob.** Operación de la API de SageMaker que crea un trabajo de entrenamiento; los hiperparámetros viajan en ella como un mapa de cadena a cadena.
- **Credential stuffing.** Ataque que prueba en masa combinaciones de usuario y contraseña filtradas en otros sitios.
- **CSV en SageMaker.** Archivo sin fila de encabezado y con la etiqueta en la primera columna; para algoritmos no supervisados se declara `text/csv;label_size=0`.
- **Cuota de servicio (service quota).** Límite por cuenta y región, como las transacciones por segundo de una API; muchas se amplían desde la consola de Service Quotas.
- **Dirección IP / IPv4 / IPv6.** Identificador de un dispositivo en la red; IPv4 usa 32 bits (p. ej., `203.0.113.25`) e IPv6, 128 bits (p. ej., `2001:db8::1`).
- **Dominio de SageMaker Studio.** Configuración de Studio para un grupo de usuarios en una región: red, autenticación, perfiles de usuario y rol de ejecución por defecto.
- **EBS (Amazon Elastic Block Store).** Almacenamiento de bloques: un disco virtual que se conecta a una sola instancia; es el almacenamiento de los espacios de Studio.
- **ECR (Amazon Elastic Container Registry).** Registro de imágenes de contenedor de AWS; aloja las imágenes de los algoritmos integrados, con una copia por región.
- **ECS (Amazon Elastic Container Service).** Servicio que ejecuta, reinicia y distribuye contenedores sobre un grupo de máquinas.
- **EFS (Amazon Elastic File System).** Almacenamiento de archivos compartido por NFS, que varias máquinas montan a la vez y que crece solo; era el almacenamiento de Studio Classic.
- **Endpoint de SageMaker.** Servicio HTTPS persistente sobre instancias administradas que sirve predicciones en tiempo real y se cobra por hora mientras exista.
- **Endpoint de VPC de interfaz (AWS PrivateLink).** Interfaz de red con una IP privada dentro de una subred que da acceso a un servicio de AWS sin pasar por internet.
- **Entrenamiento distribuido.** Reparto de un mismo entrenamiento entre varias instancias; solo algunos algoritmos integrados lo admiten.
- **Espacio (SageMaker Studio).** Instancia administrada más un volumen EBS en el que corren JupyterLab o Code Editor; se cobra por hora mientras está encendido.
- **Estimator.** Objeto del SDK de SageMaker (v2) que describe un trabajo de entrenamiento: imagen, rol, instancias, hiperparámetros y ruta de salida.
- **Face Liveness.** Función de Rekognition que comprueba que frente a la cámara hay una persona real y presente, y no una foto, una pantalla o una máscara.
- **Forecast (Amazon Forecast).** Servicio administrado de pronósticos de AWS; no admite clientes nuevos desde el 29-07-2024.
- **FSx for Lustre (Amazon FSx for Lustre).** Sistema de archivos paralelo administrado, basado en Lustre, que se usa como caché de alta velocidad delante de S3 en entrenamientos intensivos.
- **Fulfillment (Lex).** Función Lambda que ejecuta la acción que pidió el usuario y devuelve la respuesta del bot.
- **Fully managed (totalmente administrado).** Servicio cuya infraestructura opera AWS; el cliente solo lo consume y paga por uso.
- **Grupo de datasets (Personalize).** Contenedor de los datasets, soluciones y campañas de un caso de uso de Personalize.
- **Grupo de seguridad.** Conjunto de reglas de firewall que define qué tráfico puede entrar y salir de un recurso dentro de una VPC.
- **GuardDuty (Amazon GuardDuty).** Servicio de detección de amenazas sobre la cuenta de AWS, que analiza CloudTrail, VPC Flow Logs y registros de DNS.
- **IAM (AWS Identity and Access Management).** Servicio donde se define, mediante políticas, qué identidad puede hacer qué acción sobre qué recurso.
- **Inferencia entre regiones / perfil de inferencia (Bedrock).** Enrutamiento de las peticiones entre varias regiones mediante un identificador de perfil; los perfiles geográficos no salen de su geografía.
- **Instancia de rendimiento ampliable (familia T).** Instancia barata que acumula créditos de CPU y los gasta en ráfagas; pensada para trabajo interactivo ligero.
- **Intención / slot / utterance (Lex).** Acción que quiere el usuario, dato que el bot debe obtener para completarla y frase de ejemplo que la activa.
- **IOPS.** Operaciones de lectura o escritura por segundo que admite un disco.
- **IVR (interactive voice response).** Sistema telefónico automático que atiende y enruta llamadas con menús de voz.
- **JSON Lines.** Formato de texto con un objeto JSON completo por línea; lo usan DeepAR, Object2Vec y los manifiestos aumentados.
- **JumpStart (Amazon SageMaker JumpStart).** Catálogo de modelos preentrenados de SageMaker que se despliegan o ajustan con pocos pasos.
- **Kinesis Video Streams (Amazon Kinesis Video Streams).** Servicio que recibe y almacena flujos de video en vivo, como los de cámaras, para procesarlos.
- **KMS (AWS Key Management Service).** Servicio de gestión de las claves con las que se cifran los datos en AWS.
- **Knowledge Bases (Amazon Bedrock Knowledge Bases).** Función de Bedrock que conecta un modelo con documentos propios a través de una base de datos vectorial para implementar RAG.
- **Lambda (AWS Lambda).** Servicio que ejecuta código bajo demanda sin servidores que administrar, cobrado por milisegundo, con un máximo de 15 minutos por ejecución.
- **Latencia.** Tiempo que transcurre entre el envío de una petición y el inicio de la respuesta.
- **Léxico (Polly).** Diccionario de pronunciación que se carga en la cuenta y se aplica a todas las peticiones de síntesis.
- **Modo File / Pipe / FastFile.** Formas de leer los datos de S3 en un entrenamiento: copiarlos antes al disco, transmitirlos en flujo o descargarlos a medida que se leen.
- **Modo script / traer tu propio contenedor.** Entrenar con tu propio código dentro de un contenedor de framework mantenido por AWS, o con una imagen construida por ti.
- **MTBF (mean time between failures).** Tiempo medio entre fallas; métrica de confiabilidad de un equipo o de una flota.
- **Multihilo (multithreading).** Reparto del trabajo de un programa entre varios núcleos de CPU a la vez.
- **NAT (puerta de enlace).** Componente que permite a los recursos de subredes privadas iniciar conexiones hacia internet.
- **NFS (Network File System).** Protocolo estándar, sobre todo en Linux, para montar una carpeta remota como si fuera local.
- **Niveles de servicio (Bedrock).** Priority, Standard, Flex y Reserved: variantes de la inferencia con distinto precio y prioridad.
- **Pago por uso (pay-as-you-go).** Modelo de cobro sin cuota fija, en el que solo se paga lo consumido.
- **Plano de control / plano de datos.** Operaciones que crean y configuran recursos frente a las que los usan; en Bedrock, los clientes `bedrock` y `bedrock-runtime`.
- **Política de exclusión de servicios de IA.** Política de AWS Organizations que impide que ciertos servicios de IA usen el contenido de la organización para mejorarse.
- **Protocol Buffers (protobuf).** Formato de serialización binaria con esquema fijo, creado por Google, más compacto y rápido de leer que el texto.
- **Provisioned Throughput (Bedrock).** Capacidad de inferencia reservada para la cuenta, que se paga por hora se use o no.
- **Rastreador de eventos (Personalize).** Recurso que recibe en tiempo real las interacciones que la aplicación envía con `PutEvents`.
- **re:Invent (AWS re:Invent).** Conferencia anual de AWS en Las Vegas, donde se anuncia buena parte de los servicios nuevos.
- **Receta (Personalize).** Algoritmo preconfigurado por AWS para un tipo de recomendación, como `User-Personalization` o `Similar-Items`.
- **Recomendador (Personalize).** Recurso ya configurado para un caso de uso de un dominio (comercio electrónico, video bajo demanda) en el que no se elige receta.
- **recordIO-protobuf.** Formato binario de muchos algoritmos integrados: registros protobuf concatenados en un archivo RecordIO, que se puede leer en flujo (modo Pipe).
- **Región predeterminada.** Región que usan las herramientas si no se indica otra; en boto3 se toma del parámetro `region_name`, de las variables de entorno o de `~/.aws/config`.
- **Registro de invocaciones de modelos (Bedrock).** Opción que guarda el texto completo de peticiones y respuestas en S3 o en CloudWatch Logs.
- **Residencia de datos (data residency).** Requisito de almacenar o procesar ciertos datos dentro de un país o jurisdicción concretos.
- **Rol de ejecución.** Rol de IAM con cuyos permisos actúan Studio, los trabajos de entrenamiento y los endpoints de SageMaker.
- **Rol de IAM.** Identidad con permisos que asumen temporalmente personas o servicios, sin credenciales permanentes.
- **S3 (Amazon Simple Storage Service).** Almacenamiento de objetos de AWS: los archivos se guardan como objetos dentro de contenedores llamados buckets.
- **SageMaker AI (Amazon SageMaker AI).** Nombre actual del servicio de ML que el libro llama Amazon SageMaker.
- **SDK (software development kit).** Biblioteca que firma y envía desde el código las llamadas a un servicio, como boto3 o el SDK de SageMaker para Python.
- **SDK de SageMaker para Python v2 / v3.** La v2 usa `Estimator`, `image_uris` y `Session`; la v3 los sustituye por `ModelTrainer` y `ModelBuilder`, así que el código del libro requiere la v2.
- **Serverless.** Modelo sin servidores que aprovisionar: el servicio asigna cómputo al vuelo y cobra por uso.
- **SNS (Amazon Simple Notification Service).** Servicio de mensajería que reenvía notificaciones a colas, funciones Lambda o correos; lo usan las APIs asíncronas para avisar que terminaron.
- **SSML (Speech Synthesis Markup Language).** Lenguaje de marcado basado en XML para controlar el tono, la velocidad, las pausas y la pronunciación en la síntesis de voz.
- **Streaming / WebSocket.** Envío continuo de datos por una conexión persistente; WebSocket es un protocolo que mantiene abierta una conexión bidireccional.
- **Subred privada.** Segmento de una VPC cuyos recursos no son accesibles desde internet ni tienen salida directa hacia él.
- **Terminología personalizada (Translate).** Glosario de pares origen-destino que fija cómo se traducen ciertos términos.
- **Throughput.** Cantidad de trabajo completada por unidad de tiempo: MB/s en un disco, tokens por minuto en Bedrock.
- **Tipo de contenido (tipo MIME).** Etiqueta estándar que declara el formato de los datos en una petición HTTP o en un canal de SageMaker, como `text/csv` o `application/json`.
- **Tipo de instancia (`ml.familia.tamaño`).** Nombre de una instancia de SageMaker: prefijo `ml.`, familia y generación (p. ej., `t3`, `m5`, `p3`) y tamaño (p. ej., `medium`, `xlarge`).
- **Token (unidad de cobro).** Fragmento de texto con el que Bedrock mide y cobra la entrada y la salida de los modelos; 1 000 tokens son unas 750 palabras en inglés.
- **Trabajo de entrenamiento (training job).** Ejecución administrada que aprovisiona instancias, entrena, deja `model.tar.gz` en S3 y apaga las instancias al terminar.
- **Trabajo de personalización de modelo (Bedrock).** Trabajo que ajusta un modelo fundacional con datos de S3 y produce un modelo privado de la cuenta.
- **Transformación por lotes (batch transform).** Trabajo que aplica un modelo a todos los archivos de una carpeta de S3, escribe los resultados en otra y apaga las instancias al terminar.
- **Unidad de inferencia (Comprehend).** Capacidad aprovisionada de un endpoint personalizado de Comprehend, que se cobra por segundo mientras existe.
- **Versión de solución (Personalize).** Modelo entrenado por Personalize a partir de una receta y de los datasets; su entrenamiento se cobra por hora.
- **Vocabulario personalizado (Transcribe).** Lista de términos propios (productos, apellidos, jerga) que se carga en la cuenta para que el servicio los reconozca.
- **VPC (Virtual Private Cloud).** Red privada virtual dentro de AWS, aislada de las de otros clientes, donde se colocan los recursos y se controla su tráfico.
- **VPC Flow Logs.** Registros con los metadatos de cada conexión de red dentro de una VPC (IP de origen y destino, puertos, bytes, aceptada o rechazada).
- **WAF (AWS WAF).** Firewall de aplicaciones web de AWS, que filtra las peticiones HTTP con reglas propias o administradas.
