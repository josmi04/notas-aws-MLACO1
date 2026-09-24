---
tema: "Capítulo 4 — Selección de modelos (versión explicada)"
fuente: Guia Oficial/04_model_selection.txt
guia-mla-c01:
  [
    Dominio 2,
    2.1 Choose a modeling approach,
    2.2 Train and refine models,
  ]
perfil-lector: profesional de datos/estadística sin experiencia administrando infraestructura
versiones-codigo:
  python: "3.12.6"
  scikit-learn: "1.5.2"
  numpy: "2.0.2"
  pandas: "2.2.2"
  matplotlib: "3.9.2"
  xgboost: "3.0.5"
verificado: 2026-09-24
tags:
  [
    aws,
    mla-c01,
    seleccion-de-modelos,
    servicios-de-ia,
    bedrock,
    sagemaker,
    algoritmos-integrados,
    rekognition,
    textract,
    transcribe,
    comprehend,
    personalize,
    xgboost,
    k-means,
    pca,
  ]
---

> [!info] Cómo leer esta versión
>
> - **Qué es.** Traducción íntegra al español del capítulo 4 de la guía oficial de estudio, con los mismos encabezados, casos de uso y afirmaciones. No se eliminó nada. Los encabezados conservan entre paréntesis el título original en inglés, porque el examen se presenta en inglés.
> - **Qué se añadió.** Explicaciones de los términos de AWS y de infraestructura integradas en el texto, aclaraciones de razonamientos que el libro da por sentados, **notas de precisión** (recuadros amarillos) cuando el original es impreciso o está desactualizado, las salidas reales del código, la sección «Escenarios donde este servicio es la opción obligada» y un glosario al final.
> - **Recuadros del libro.** Las notas al margen del libro aparecen como recuadros «Recuadro del libro».
> - **Figuras.** El `.txt` no incluye las imágenes, así que solo quedan los pies de figura. Cuando la figura mostraba la salida de un programa que corre en una computadora local, aquí va la salida real obtenida al ejecutar ese mismo código con las versiones del encabezado del archivo. Los programas que llaman a servicios de AWS (Bedrock, entrenamiento en SageMaker) **no** se ejecutaron: requieren una cuenta y generan costos.
> - **Código.** Se corrigieron defectos de la extracción del PDF (líneas partidas a la mitad, comillas tipográficas) para que el código se pueda ejecutar. La lógica no se modificó. Cuando el código del libro tiene errores, se señalan en una nota y se muestra la corrección aparte.
> - **Fórmulas.** En el `.txt` se perdieron las fórmulas y varios símbolos sueltos (como el valor de _k_). Se reconstruyeron a partir del contexto y se indica cuando la reconstrucción implica una suposición.
> - **Cifras y estado de los servicios.** Se verificaron en la documentación de AWS el 24 de septiembre de 2026. Cambian con frecuencia, así que confírmalos en la documentación oficial vigente antes de usarlos en una decisión real.

> [!warning] Cambios en AWS posteriores a la edición del libro (verificado el 24-09-2026)
>
> - **Amazon SageMaker** pasó a llamarse **Amazon SageMaker AI**. El propio libro lo advierte en un recuadro. En este texto se usa el nombre del libro.
> - **SDK de SageMaker para Python, versión 3.** La versión 3 eliminó las clases `Estimator`, `Model` y `Predictor` que usan todos los ejemplos de SageMaker de este capítulo, y no incluye una capa de compatibilidad. Para ejecutar esos ejemplos hay que instalar la versión 2: `pip install "sagemaker<3"`.
> - **Modelos de Amazon en Bedrock.** **Nova Canvas**, el modelo del ejemplo de código del capítulo, está en estado _Legacy_ desde el 30-03-2026 y llega a su fin de vida (_end of life_, EOL) el **30-09-2026**; los clientes nuevos ya no pueden usarlo. Lo mismo ocurre con Nova Reel (EOL 30-09-2026). Nova Premier llegó a su EOL el 14-09-2026. Los modelos **Titan Text G1 (Premier, Express y Lite)** que recomienda el libro ya no aparecen en el catálogo de Bedrock. Nova Micro, Nova Lite y Nova Pro siguen activos, junto con la familia Nova 2 (por ejemplo, Nova 2 Lite).
> - **Acceso a modelos de Bedrock.** Ya no hay que «solicitar acceso» a cada modelo como muestra la Figura 4.2: todos los modelos están habilitados por defecto en las regiones comerciales, siempre que tengas los permisos de AWS Marketplace adecuados. Los modelos de Anthropic piden, además, llenar una vez un formulario de caso de uso.
> - **Amazon Rekognition.** Las funciones de **video en streaming** y de **análisis masivo de imágenes** (_Bulk Image Analysis_) ya no admiten clientes nuevos desde el 30-04-2026. El resto de Rekognition sigue disponible.
> - **Amazon Comprehend.** El **modelado de temas** (_topic modeling_), la detección de eventos y la clasificación de seguridad de prompts ya no admiten clientes nuevos desde el 30-04-2026.
> - **Amazon Lex V1** dejó de tener soporte el 15-09-2025. La versión vigente es Lex V2.
> - **Funciones de SageMaker AI en mantenimiento.** Desde el 30-07-2026 ya no admiten clientes nuevos, entre otras, Augmented AI (A2I), Clarify, Debugger, Ground Truth y Model Monitor. No son protagonistas de este capítulo, pero el libro las anuncia para los siguientes.
> - «Ya no admite clientes nuevos» significa que las cuentas que ya usaban la función pueden seguir usándola, pero una cuenta nueva no puede empezar a hacerlo. Las preguntas de examen basadas en el libro pueden seguir mencionando estas funciones, así que conviene conocerlas igual.

**Capítulo 4**

# Selección de modelos (_Model Selection_)

> LOS OBJETIVOS DEL EXAMEN AWS CERTIFIED MACHINE LEARNING (ML) ENGINEER – ASSOCIATE QUE CUBRE ESTE CAPÍTULO PUEDEN INCLUIR, ENTRE OTROS, LOS SIGUIENTES:
>
> ✔ Dominio 2: Desarrollo de modelos de ML
>
> - 2.1 Elegir un enfoque de modelado
> - 2.2 Entrenar y refinar modelos

En el capítulo anterior exploramos los pasos importantes de la transformación de datos y la ingeniería de características, que sientan las bases para aplicaciones eficaces de machine learning (ML). En este capítulo pasamos a la emocionante fase de seleccionar un modelo en función de tu problema de ML.

Con referencia al ciclo de vida de ML (Figura 4.1), en este capítulo nos centramos en elegir un algoritmo de ML adecuado (o un servicio de IA) ajustado a tus requisitos específicos de ML. La distinción entre las dos opciones es la columna vertebral del capítulo. Un **algoritmo** es un método que tú entrenas con tus datos (por ejemplo, XGBoost); el resultado es un modelo propio. Un **servicio de IA** (_AI service_) es un modelo que AWS ya entrenó y que tú solo consultas. Es la diferencia entre ajustar tu propio modelo de regresión y usar una calculadora que ya trae la fórmula.

_Figura 4.1 El ciclo de vida del machine learning._

Aprenderás a seleccionar el algoritmo de ML más adecuado para tu caso de uso específico y a implementarlo de forma eficaz para iniciar el proceso de desarrollo del modelo. Recuerda que el proceso de desarrollo de modelos es iterativo por naturaleza: implica refinar y evaluar continuamente para asegurar el mejor rendimiento y la mayor precisión posibles de tu modelo. Si dominas este enfoque iterativo, estarás bien preparado para abordar desafíos complejos de datos y generar resultados de impacto.

Este capítulo te guiará por el proceso intrincado, pero fascinante, de transformar datos crudos en modelos predictivos potentes que pueden impulsar conocimiento accionable y decisiones.

La selección de un algoritmo de ML apropiado es un paso crucial del proceso de desarrollo del modelo. Esta decisión depende de entender con claridad tu caso de uso, la naturaleza de tus datos y el problema específico que quieres resolver. Tanto si enfrentas una tarea de clasificación, un problema de regresión o una necesidad de agrupamiento (_clustering_), elegir el algoritmo correcto implica equilibrar factores como la interpretabilidad, la precisión, la eficiencia computacional, la escalabilidad y el costo. Nos adentraremos en varios tipos de algoritmos, con pautas sobre cuándo usar cada uno y cómo adecuarlos a tus requisitos particulares. En este contexto, **escalabilidad** es la capacidad de seguir funcionando con un costo y un tiempo razonables cuando crecen los datos o las peticiones. Un algoritmo cuyo costo crece con el cuadrado del número de filas puede ir bien con 10 000 filas y volverse inviable con 10 millones.

Siguiendo el ciclo de vida de ML, una vez seleccionado el algoritmo comienza el recorrido del entrenamiento. Aquí cubriremos los pasos básicos para alimentar el algoritmo con datos. En el próximo capítulo aprenderás a ajustar hiperparámetros y a refinar iterativamente el modelo para mejorar su rendimiento. También hablaremos de técnicas para evitar trampas comunes como el sobreajuste (_overfitting_) y el subajuste (_underfitting_), de modo que tu modelo generalice bien a datos nuevos, no vistos. Un **hiperparámetro** es un ajuste que se fija antes de entrenar (la profundidad máxima de un árbol, el número de vecinos de k-NN), a diferencia de los parámetros, que el algoritmo estima a partir de los datos (los coeficientes de una regresión).

Al terminar este capítulo tendrás una comprensión integral de la fase de selección de modelos, con el conocimiento necesario para elegir el algoritmo o el servicio de IA más apropiado para tu problema de ML y la confianza para implementar tus elecciones de forma eficaz.

## Comprender los servicios de IA de AWS (_Understanding AWS AI Services_)

Los servicios de IA de AWS son modelos de ML potentes, prediseñados y preentrenados (_prebuilt and pretrained_) que ayudan a desarrolladores e ingenieros de ML a integrar IA en sus aplicaciones de forma fluida, sin requerir un conocimiento profundo de ML. «Preentrenado» significa que AWS ya ajustó el modelo con sus propios datos masivos: tú no aportas datos de entrenamiento ni eliges hiperparámetros, solo envías una imagen, un audio o un texto y recibes el resultado. Estos servicios cubren una amplia gama de capacidades de IA, como visión por computadora (_computer vision_: extraer información de imágenes y video), procesamiento de lenguaje natural (NLP: analizar o generar texto humano), reconocimiento de voz (convertir audio hablado en texto), IA generativa (crear contenido nuevo) y analítica predictiva. Por ejemplo, Amazon Rekognition permite a los usuarios analizar imágenes y videos para identificar objetos, personas, texto, escenas y actividades. Por su parte, Amazon Comprehend realiza tareas de NLP como el análisis de sentimiento y el reconocimiento de entidades, lo que permite a las empresas obtener conocimiento a partir de datos de texto no estructurados.

Una de las ventajas clave de los servicios de IA de AWS es su facilidad de uso y su escalabilidad. Vienen con **API totalmente administradas** (_fully managed APIs_) que los desarrolladores pueden llamar desde sus aplicaciones para aprovechar capacidades sofisticadas de ML con un esfuerzo mínimo. Conviene separar los dos términos:

- Una **API** (_application programming interface_) es un contrato para que un programa le pida algo a otro. En AWS, cada llamada es una petición HTTPS (la misma tecnología que usa un navegador) con parámetros en formato JSON, y la respuesta también llega en JSON. En la práctica no escribes esas peticiones a mano: usas un **SDK** (_software development kit_), una biblioteca que las arma por ti. El SDK de AWS para Python se llama **boto3**, y una llamada se ve como `rekognition.detect_labels(Image=...)`.
- **Totalmente administrado** (_fully managed_) significa que AWS opera toda la infraestructura: los servidores, el sistema operativo, las actualizaciones, la réplica ante fallas y el aumento de capacidad. Tú no ves ni administras ninguna máquina; solo consumes el servicio y pagas por uso (por imagen, por minuto de audio, por carácter traducido). Para alguien sin experiencia administrando servidores, esta es la diferencia práctica más importante del capítulo.

Estos servicios también están diseñados para manejar el procesamiento de datos a gran escala, lo que asegura que las aplicaciones puedan cumplir requisitos de alto rendimiento.

Además, los servicios de IA de AWS se integran de forma fluida con otras ofertas de AWS, lo que permite construir flujos de trabajo de ML de extremo a extremo (_end-to-end_), desde la recolección y el procesamiento de datos hasta el entrenamiento y el despliegue de modelos. Por ejemplo, un archivo que llega a un bucket de Amazon S3 puede disparar automáticamente su análisis, y el resultado puede guardarse de vuelta en S3 para consultarlo con SQL. Este enfoque holístico ayuda a agilizar el proceso de desarrollo y acelera la adopción de la IA en diversas industrias.

Para el examen, necesitas saber qué servicios de IA de AWS atienden distintos casos de uso de IA y ML, como el análisis de texto, el reconocimiento de imágenes y video, la síntesis y el reconocimiento de voz, la traducción, las recomendaciones personalizadas, el procesamiento de documentos y la IA generativa. Esto incluye entender servicios como Amazon Comprehend para el análisis de texto, Amazon Rekognition para el reconocimiento de imágenes y video, Amazon Polly para la síntesis de voz, Amazon Transcribe para el reconocimiento de voz, Amazon Translate para la traducción, Amazon Personalize para las recomendaciones, Amazon Textract para el procesamiento de documentos y Amazon Bedrock para la IA generativa. Conocer los casos de uso y las aplicaciones específicas de cada servicio es clave para aprovechar eficazmente las capacidades de IA de AWS en distintos escenarios.

Como mapa mental para el examen, casi todos se reconocen por la pareja «entrada → salida»:

| Servicio | Entrada | Salida |
| --- | --- | --- |
| Rekognition | Imagen o video | Etiquetas, rostros, texto en la escena |
| Textract | Documento escaneado o PDF | Texto, formularios y tablas estructurados |
| Polly | Texto | Audio de voz |
| Transcribe | Audio de voz | Texto |
| Translate | Texto en un idioma | Texto en otro idioma |
| Comprehend | Texto | Sentimiento, entidades, frases clave, idioma |
| Lex | Voz o texto de un usuario | Conversación (intención y respuesta) |
| Personalize | Historial de interacciones usuario-ítem | Recomendaciones |
| Bedrock | Prompt (texto, imagen) | Contenido generado |

### Visión (_Vision_)

AWS ofrece herramientas potentes para el análisis de contenido visual mediante servicios como Amazon Rekognition y Amazon Textract. Amazon Rekognition ayuda a analizar imágenes y videos para identificar objetos, rostros y texto, con capacidades que potencian aplicaciones como la seguridad y el análisis de medios. Amazon Textract destaca en la extracción de texto y datos de documentos escaneados, lo que permite automatizar el procesamiento de documentos. Juntos, estos servicios permiten a las empresas obtener conocimiento valioso de sus datos visuales de forma eficiente y eficaz.

#### Amazon Rekognition

Amazon Rekognition es un servicio versátil de análisis de imágenes y video impulsado por algoritmos de aprendizaje profundo (_deep learning_, redes neuronales con muchas capas), diseñado para identificar una amplia gama de objetos, personas, texto, escenas y actividades en medios visuales. Una de sus funciones más destacadas es el análisis facial, que incluye la detección, la comparación y el reconocimiento de rostros. Las tres operaciones responden preguntas distintas, y el examen suele jugar con esa diferencia:

- **Detección**: «¿hay rostros en esta imagen y dónde están?». Devuelve la ubicación de cada rostro y atributos estimados, como la edad aparente o si tiene los ojos abiertos.
- **Comparación** (1:1): «¿estas dos fotos son de la misma persona?». Devuelve un puntaje de similitud. Es la base de la verificación de identidad.
- **Reconocimiento o búsqueda** (1:N): «¿quién de mi colección de rostros registrados es esta persona?».

Esto permite a los desarrolladores crear aplicaciones capaces de reconocer rostros individuales en tiempo real o en medios almacenados. Además, Amazon Rekognition puede detectar texto dentro de imágenes y videos, lo que permite un análisis integral y la extracción de información textual. Se trata de texto que aparece en una escena, como un letrero o una placa de auto. Para documentos, el servicio indicado es Textract.

El servicio es fácil de usar: ofrece una API sencilla que permite a los desarrolladores añadir funciones de análisis de imágenes y video a sus aplicaciones sin requerir un conocimiento extenso de ML. Su escalabilidad asegura que pueda manejar grandes volúmenes de datos de forma eficiente, lo que lo hace adecuado tanto para proyectos pequeños como para soluciones de nivel empresarial. Al aprovechar Amazon Rekognition, los desarrolladores pueden construir aplicaciones sofisticadas que analicen contenido visual para extraer conocimiento valioso, automatizar procesos y mejorar la experiencia de los usuarios.

##### Casos de uso (_Use Cases_)

Amazon Rekognition se aprovecha para diversas aplicaciones críticas en distintas industrias. Ayuda a detectar contenido inapropiado en imágenes y videos, lo que asegura medios seguros y conformes con las normas de las plataformas; a esto se le llama **moderación de contenido**. Este servicio también se usa para verificar identidades en línea, lo que refuerza la seguridad y los procesos de validación de usuarios en servicios digitales (por ejemplo, comparar la selfie de un cliente con la foto de su identificación al abrir una cuenta bancaria desde el celular). Las empresas de medios pueden usar este servicio para agilizar su análisis, etiquetando y categorizando contenido de forma automática, lo que facilita gestionar y recuperar activos específicos. Además, Amazon Rekognition puede enviar alertas inteligentes a hogares conectados, reconociendo rostros u objetos específicos y notificando a los propietarios sobre actividades inusuales, lo que refuerza la seguridad del hogar y los sistemas de automatización. Estos casos de uso muestran la versatilidad de Amazon Rekognition y su papel significativo en la transformación del uso de los datos visuales en distintos sectores.

> [!warning] Estado del servicio: alertas en video en vivo (verificado el 24-09-2026)
> Las alertas para hogares conectados que describe el libro se construían con **Rekognition Streaming Video Events**, que analiza video en vivo de cámaras. Esa función ya no admite clientes nuevos desde el 30-04-2026. Para una cuenta nueva, AWS sugiere procesar fotogramas sueltos del video con la API de imágenes de Rekognition. La moderación de contenido, la comparación de rostros y el etiquetado de imágenes almacenadas no se ven afectados.

#### Amazon Textract

Amazon Textract es un servicio de IA sofisticado, diseñado para extraer texto, formularios, tablas y otros datos de documentos escaneados. A diferencia de las tecnologías tradicionales de **reconocimiento óptico de caracteres** (OCR, _optical character recognition_), que convierten la imagen de un texto en caracteres pero entregan una «sopa» de líneas sin estructura, Amazon Textract usa ML para entender el diseño (_layout_) y la estructura de los documentos, y capta las relaciones semánticas entre sus elementos. Por ejemplo, sabe que «$1,250.00» es el _valor_ del campo «Total a pagar» y no una línea suelta. Esta capacidad avanzada permite a este servicio de IA interpretar con precisión información compleja, lo que lo hace especialmente útil para procesar grandes volúmenes de documentos de forma rápida y eficaz, transformando datos no estructurados en formatos estructurados que se pueden analizar y utilizar de forma eficiente.

Una de las funciones clave de Amazon Textract es su capacidad de reconocer y extraer datos de tablas y formularios dentro de los documentos, entendiendo el contexto y las relaciones entre las distintas piezas de datos. En un formulario, esas relaciones son pares clave-valor (_key-value pairs_): la clave es la etiqueta impresa («Fecha de nacimiento») y el valor es lo que se llenó a mano o a máquina («12/03/1985»). Esto va más allá de simplemente leer texto: Amazon Textract puede distinguir entre encabezados, filas y columnas de una tabla, de modo que los datos extraídos mantengan su estructura y su significado originales. Este entendimiento semántico es crucial para aplicaciones que requieren una extracción e interpretación precisas de los datos, como los reportes financieros, la documentación legal y los expedientes de salud, donde el contexto de los datos es tan importante como los datos mismos.

Amazon Textract también se integra de forma fluida con otros servicios de AWS, lo que permite construir **pipelines** (cadenas automatizadas de pasos) integrales de procesamiento de datos. Por ejemplo, los datos extraídos pueden almacenarse en Amazon S3 (el almacenamiento de objetos de AWS), analizarse con Amazon Athena (un motor SQL que consulta directamente archivos en S3) o alimentar a Amazon SageMaker para aplicaciones de ML. Esta flexibilidad permite a las organizaciones aprovechar Amazon Textract como parte de una estrategia de datos más amplia, mejorando su capacidad de obtener conocimiento y tomar decisiones a partir de los datos de sus documentos. Al automatizar la extracción y el procesamiento de la información y preservar las relaciones semánticas, Amazon Textract ayuda a reducir el esfuerzo manual, minimizar errores y aumentar la eficiencia operativa.

##### Casos de uso (_Use Cases_)

Amazon Textract se usa ampliamente en diversas industrias para agilizar el procesamiento de documentos y la extracción de datos. En el sector financiero, ayuda a automatizar la extracción de datos de facturas, recibos y documentos fiscales, lo que permite reportes financieros más rápidos y precisos. En salud, Amazon Textract se usa para digitalizar expedientes de pacientes, lo que facilita recuperar y analizar la información médica. Los despachos legales pueden aprovechar Amazon Textract para procesar grandes volúmenes de contratos y documentos legales, asegurando que la información crítica se capture con precisión y sea accesible. Además, en el sector público, Amazon Textract puede ayudar a gestionar y analizar formularios y solicitudes gubernamentales, mejorando la eficiencia de los procesos administrativos. Estos casos de uso resaltan la adaptabilidad de Amazon Textract y su impacto significativo en la automatización de la extracción y el procesamiento de datos en distintas industrias.

Según la documentación vigente, Textract ofrece operaciones especializadas para algunos de estos casos: una para facturas y recibos (_AnalyzeExpense_), otra para documentos de identidad (_AnalyzeID_) y otra para expedientes de préstamos hipotecarios (_AnalyzeLending_), además de la operación general de análisis de documentos, que extrae formularios, tablas, firmas y respuestas a preguntas en lenguaje natural sobre el documento (_Queries_). Verifica en la documentación qué tipos de documento cubre cada una en tu región.

### Voz (_Speech_)

Los servicios de IA de AWS ofrecen capacidades de voz integrales, diseñadas para transformar la forma en que las aplicaciones interactúan con los usuarios mediante la voz. Estos servicios proporcionan herramientas tanto para generar voz a partir de texto como para convertir el lenguaje hablado en texto, lo que permite experiencias de usuario más naturales y accesibles. Para los casos de uso de voz, los dos servicios principales son Amazon Polly, para convertir texto en voz de sonido natural, y Amazon Transcribe, para convertir el lenguaje hablado en texto preciso. Una regla mnemotécnica: Polly _habla_ (texto → audio) y Transcribe _escucha_ (audio → texto).

#### Amazon Polly

Amazon Polly es un servicio de conversión de texto a voz (**TTS**, _text-to-speech_) que usa tecnologías avanzadas de aprendizaje profundo para convertir texto escrito en voz de sonido natural. Ofrece una amplia variedad de voces realistas en múltiples idiomas y admite distintos acentos, lo que da a los desarrolladores flexibilidad para crear experiencias atractivas y localizadas para sus usuarios. Ya sea para sistemas de **respuesta de voz interactiva** (IVR, _interactive voice response_: el menú telefónico automatizado de «para ventas, marque 1»), aplicaciones de lectura de noticias o audiolibros, Amazon Polly mejora la participación de los usuarios al hacer las aplicaciones más accesibles e interactivas.

Amazon Polly destaca por ofrecer síntesis de voz rápida y en tiempo real, con una latencia mínima para aplicaciones que requieren respuestas inmediatas. La **latencia** es el tiempo que pasa entre la petición y la respuesta. En una conversación telefónica, una pausa de más de uno o dos segundos ya se percibe como un fallo, así que un TTS para IVR tiene que empezar a devolver audio en fracciones de segundo. Amazon Polly también admite el **lenguaje de marcado para síntesis de voz** (SSML, _Speech Synthesis Markup Language_), que permite a los desarrolladores controlar aspectos como el tono, la velocidad y la pronunciación de la voz para una experiencia auditiva más personalizada. SSML funciona como el HTML de la voz: se envuelve el texto en etiquetas, por ejemplo `<prosody rate="slow">...</prosody>` para hablar más despacio o `<break time="1s"/>` para insertar una pausa. Con Amazon Polly puedes crear interacciones de voz dinámicas y naturales de forma fácil y eficiente.

##### Casos de uso (_Use Cases_)

Amazon Polly es un servicio de texto a voz versátil con una amplia gama de casos de uso. Es ideal para crear interacciones de voz atractivas en sistemas IVR, asegurando que los clientes reciban respuestas claras y de sonido natural. Polly también es perfecto para generar audiolibros, ofreciendo a los lectores una experiencia inmersiva con una narración realista. Además, mejora la accesibilidad al convertir contenido textual en voz para usuarios con discapacidad visual, lo que hace la información más accesible. En plataformas de aprendizaje en línea (_e-learning_), Polly puede dar vida al contenido educativo con una narración dinámica, mejorando la comprensión y la retención. En general, Amazon Polly es una herramienta potente para cualquier aplicación que se beneficie de una voz de alta calidad y sonido natural.

#### Amazon Transcribe

Amazon Transcribe es un potente servicio de **reconocimiento automático de voz** (ASR, _automatic speech recognition_) que convierte el lenguaje hablado en texto escrito. Está diseñado para ofrecer transcripciones de alta precisión para diversos formatos de audio y video, lo que lo convierte en una herramienta esencial para generar texto consultable a partir de voz grabada. Ya sea para crear subtítulos para contenido de video, generar transcripciones de podcasts y entrevistas o habilitar subtítulos en tiempo real para eventos en vivo, Amazon Transcribe mejora la accesibilidad y la capacidad de búsqueda. También admite múltiples idiomas y dialectos, lo que asegura su aplicabilidad en distintas regiones y contextos. Con funciones como la identificación de hablantes (_speaker identification_), la restauración de la puntuación (_punctuation restoration_) y los vocabularios personalizados (_custom vocabularies_), Amazon Transcribe ayuda a los usuarios a obtener transcripciones precisas y completas para aplicaciones diversas.

Tres de estas funciones merecen una aclaración:

- La **identificación de hablantes**, en la documentación de AWS, es en realidad una _separación_ de hablantes (_speaker diarization_ o _partitioning_): el servicio marca qué fragmentos dijo el hablante 0, el 1, el 2, pero no sabe **quién** es cada uno. Asignarles nombre es trabajo tuyo.
- La **restauración de la puntuación** existe porque el reconocimiento de voz produce, en bruto, palabras sin comas ni puntos; el servicio las añade para que el texto sea legible.
- Un **vocabulario personalizado** es una lista de palabras que el modelo no conoce o suele confundir (nombres de productos, apellidos, siglas del sector) y que le entregas para que las reconozca.

Transcribe trabaja en dos modos. En modo **por lotes** (_batch_), procesa un archivo de audio guardado en S3 y entrega la transcripción al terminar. En modo **streaming**, recibe el audio en vivo por una conexión abierta y va devolviendo texto a medida que se habla. Los subtítulos en vivo que menciona el libro requieren este segundo modo.

##### Casos de uso (_Use Cases_)

Amazon Transcribe atiende una amplia gama de aplicaciones. Es perfecto para crear subtítulos precisos para contenido de video, lo que hace los medios más accesibles y atractivos. Los podcasters y entrevistadores pueden usarlo para generar transcripciones, lo que facilita archivar y buscar el contenido. En entornos en vivo, Transcribe puede ofrecer subtítulos en tiempo real, mejorando la accesibilidad para audiencias con discapacidad auditiva. También es valioso para los **centros de contacto** (_call centers_) de atención al cliente, donde puede transcribir conversaciones para mejorar el análisis y el cumplimiento normativo (_compliance_). El cumplimiento normativo, aquí, es poder demostrar ante un auditor que los agentes dijeron lo que la regulación exige, por ejemplo leer el aviso de privacidad al inicio de la llamada; con la transcripción, eso se puede verificar con una búsqueda de texto en lugar de escuchar horas de grabación. Al convertir audio en texto de forma eficiente, Amazon Transcribe mejora la usabilidad y la accesibilidad del contenido hablado en diversas industrias.

### Lenguaje (_Language_)

Los servicios de IA de lenguaje de AWS están diseñados para mejorar y simplificar una amplia gama de tareas de NLP, con herramientas para entender, generar y traducir el lenguaje humano. Estos servicios permiten a los desarrolladores construir aplicaciones que interactúen con los usuarios de forma más intuitiva y natural, automaticen el análisis de texto y apoyen la comunicación multilingüe. En particular, Amazon Translate ofrece traducción en tiempo real y Amazon Comprehend proporciona conocimiento a través del análisis de texto, incluida la detección de sentimiento y el reconocimiento de entidades.

#### Amazon Translate

Amazon Translate es un servicio de **traducción automática neuronal** (NMT, _neural machine translation_) que ofrece traducción rápida, de alta calidad y asequible. «Neuronal» significa que traduce la oración completa con una red neuronal que tiene en cuenta el contexto, en lugar de traducir frase por frase con reglas o tablas estadísticas, como hacían los sistemas anteriores. Admite docenas de idiomas (del orden de 75 según la documentación; verifica la lista vigente), lo que permite a empresas y desarrolladores traducir grandes volúmenes de texto de forma eficiente e integrar capacidades de traducción directamente en sus aplicaciones. Amazon Translate puede manejar traducción en tiempo real para aplicaciones como sitios web y apps móviles, lo que facilita que los usuarios de todo el mundo interactúen con el contenido en su idioma nativo. El servicio también admite traducción por lotes, útil para traducir rápidamente documentos o datasets grandes. La diferencia es la misma que en Transcribe: en tiempo real envías un texto y recibes la traducción en la misma llamada; por lotes dejas muchos archivos en S3, lanzas un trabajo y recoges los resultados al terminar.

Amazon Translate está diseñado para preservar el contexto y el significado del texto original, de modo que las traducciones no solo sean semánticamente precisas, sino que también suenen naturales. Usa modelos avanzados de aprendizaje profundo entrenados con una enorme variedad de datos multilingües, lo que le permite mejorar continuamente con el tiempo. Amazon Translate es altamente escalable y rentable, lo que lo hace accesible para empresas de todos los tamaños. También se integra de forma fluida con otros servicios de AWS, lo que permite construir soluciones integrales que aprovechen la traducción junto con otras herramientas de IA y de la nube.

> [!warning] Nota de precisión: «mejora continuamente»
> La frase significa que AWS actualiza periódicamente sus modelos, no que el servicio aprenda de tus textos. Si necesitas que respete tu terminología (nombres de marca, términos técnicos que no deben traducirse), Translate ofrece **terminologías personalizadas**: una lista de pares origen-destino que el servicio aplica al traducir. Consulta en la documentación las opciones de personalización vigentes.

##### Casos de uso (_Use Cases_)

Amazon Translate atiende una amplia gama de casos de uso, lo que lo convierte en una herramienta invaluable para empresas y desarrolladores que necesitan comunicación multilingüe. Se usa ampliamente para traducir el contenido de sitios web, lo que permite a los usuarios globales acceder a la información e interactuar con ella en su idioma nativo. Las plataformas de comercio electrónico aprovechan Amazon Translate para ofrecer descripciones de productos en varios idiomas, mejorando la experiencia de los usuarios y ampliando su alcance de mercado. En atención al cliente, ayuda a traducir las comunicaciones por chat y correo electrónico, lo que asegura un servicio fluido en distintas regiones. Amazon Translate también cumple un papel crucial en la traducción de documentos, reportes y manuales de usuario, facilitando la colaboración global y el intercambio de información. Además, admite traducción en tiempo real para aplicaciones como redes sociales y mensajería instantánea, fomentando una comunicación instantánea y eficaz en todo el mundo.

#### Amazon Comprehend

Amazon Comprehend es un servicio de NLP que usa ML para descubrir conocimiento y relaciones dentro del texto. Permite a las empresas entender el sentimiento detrás de las reseñas de clientes, extraer frases clave, identificar **entidades con nombre** (_named entities_: menciones de personas, lugares, organizaciones, fechas o cantidades) e incluso detectar el idioma del texto de entrada. Amazon Comprehend puede analizar grandes volúmenes de texto con rapidez y precisión, lo que lo convierte en una excelente opción para procesar comentarios de clientes, publicaciones en redes sociales y otros datos no estructurados. El servicio también puede organizar documentos identificando temas clave y categorizando el contenido, lo que ayuda a las organizaciones a gestionar y entender mejor sus datos.

> [!warning] Estado del servicio: modelado de temas en Comprehend (verificado el 24-09-2026)
> La identificación de «temas clave» del párrafo anterior corresponde a la función de **modelado de temas** (_topic modeling_) de Comprehend, que ya no admite clientes nuevos desde el 30-04-2026. Las alternativas dentro de AWS son los algoritmos integrados LDA y NTM de SageMaker, que el libro presenta más adelante en este capítulo.

Amazon Comprehend es un servicio altamente configurable que permite crear **modelos personalizados** adaptados a necesidades específicas de negocio. Estos modelos personalizados permiten un reconocimiento de entidades y una clasificación más precisos, basados en la terminología y el contexto particulares de distintas industrias. En la práctica, tú aportas ejemplos etiquetados (por ejemplo, 1 000 correos marcados como «reclamo», «consulta» o «baja») y Comprehend entrena y aloja el modelo por ti: sigues sin elegir algoritmos ni administrar servidores. Igual que otros servicios de IA de AWS, Amazon Comprehend se integra de forma fluida con el ecosistema de AWS, lo que permite construir pipelines integrales de análisis de texto y automatizar flujos de trabajo. Ya sea para mejorar la atención al cliente analizando tickets de soporte o para mejorar sistemas de recomendación de contenido, Amazon Comprehend ofrece herramientas potentes para convertir texto en conocimiento accionable.

Para organizaciones que requieren capacidades más avanzadas, Amazon SageMaker ofrece algoritmos sofisticados de modelado de temas listos para usar (_out of the box_). Estos algoritmos se analizan en detalle más adelante en el capítulo.

##### Casos de uso (_Use Cases_)

Amazon Comprehend puede aplicarse a una amplia gama de casos de uso en distintas industrias. Las empresas suelen usarlo para el análisis de sentimiento, que les ayuda a entender los comentarios de los clientes y el sentimiento en redes sociales para mejorar productos y servicios. En la industria de la salud, Amazon Comprehend puede analizar expedientes médicos para extraer conocimiento valioso e identificar entidades clave como medicamentos, padecimientos y tratamientos. También se usa en sistemas de gestión de contenido para etiquetar y organizar automáticamente grandes volúmenes de documentos, lo que facilita buscarlos y gestionarlos. En atención al cliente, Amazon Comprehend puede analizar tickets de soporte para identificar problemas comunes y mejorar los tiempos de respuesta. Además, ayuda en la detección de fraude analizando patrones de comunicación e identificando actividades sospechosas. En general, Amazon Comprehend permite a las organizaciones obtener conocimiento valioso de los datos de texto, mejorando la toma de decisiones y la eficiencia operativa.

> [!warning] Nota de precisión: Comprehend y Comprehend Medical
> La extracción de medicamentos, padecimientos y tratamientos de expedientes clínicos la hace un servicio aparte, **Amazon Comprehend Medical**, con modelos entrenados específicamente para texto médico. El Comprehend general reconoce entidades genéricas (personas, lugares, fechas). En una pregunta de examen sobre texto clínico, la respuesta suele ser Comprehend Medical.

### Chatbot

AWS ofrece capacidades potentes de **chatbot** (programa que conversa con personas por texto o voz) mediante servicios que permiten a las empresas crear interfaces conversacionales atractivas, interactivas e inteligentes. Estos chatbots pueden usarse en diversas aplicaciones, desde la atención al cliente hasta aplicaciones interactivas y mesas de ayuda internas. AWS ofrece herramientas como Amazon Lex, que usa NLP avanzado para entender lo que escribe o dice el usuario y gestionar conversaciones complejas.

#### Amazon Lex

Amazon Lex es un servicio potente diseñado para integrar interfaces conversacionales en cualquier aplicación, mediante voz y texto. Aprovechando las mismas tecnologías de aprendizaje profundo que impulsan a Amazon Alexa (el asistente de voz de Amazon), Amazon Lex ofrece capacidades avanzadas de **comprensión del lenguaje natural** (NLU, _natural language understanding_). La NLU es la parte del NLP que extrae la intención de una frase: entiende que «quiero cambiar mi vuelo del martes» y «necesito mover mi reservación» piden lo mismo. Esto permite a los desarrolladores crear chatbots sofisticados e interactivos que entienden y responden a lo que dicen los usuarios con un alto grado de precisión. Al integrarse con otros servicios de AWS como **AWS Lambda**, Amazon Lex permite ejecutar sin fricciones la lógica de _backend_, lo que hace posible construir aplicaciones conversacionales complejas de extremo a extremo. Lambda es el servicio _serverless_ de AWS para ejecutar funciones: subes un fragmento de código (por ejemplo, una función de Python que consulta el estado de un pedido en tu base de datos) y AWS lo ejecuta cada vez que ocurre un evento, sin que tengas un servidor encendido; pagas solo por las milésimas de segundo que corre. El _backend_ es la parte del sistema que el usuario no ve: bases de datos, reglas de negocio y sistemas internos.

Amazon Lex tiene la capacidad de manejar conversaciones de varios turnos (_multiturn_), lo que permite a los chatbots mantener el contexto y gestionar diálogos complejos. Un turno es un intercambio pregunta-respuesta. Esto es especialmente útil en aplicaciones de atención al cliente, donde un bot puede necesitar reunir información a lo largo de varias interacciones para resolver un problema. Amazon Lex también admite integraciones nativas con **Amazon Connect**, el servicio de centro de contacto impulsado por IA de AWS, lo que permite a las empresas crear agentes automatizados de centro de llamadas que atiendan a los clientes las 24 horas, los 7 días de la semana, reduciendo la carga de los agentes humanos y mejorando los tiempos de respuesta. Además, Amazon Lex puede usarse para construir chatbots para diversas plataformas, incluidas aplicaciones web, apps móviles y canales de redes sociales, ofreciendo experiencias de usuario consistentes y escalables en los distintos puntos de contacto.

Desde el punto de vista de la ingeniería, Amazon Lex ofrece una consola y un entorno de desarrollo fáciles de usar, donde los desarrolladores pueden definir el modelo de interacción del bot, incluidas las intenciones (_intents_), los espacios (_slots_) y las respuestas, sin necesitar experiencia extensa en ML o NLP. Estos tres conceptos forman el vocabulario básico de Lex, y conviene verlos con un ejemplo:

- Una **intención** es lo que el usuario quiere lograr, por ejemplo `ReservarHotel`. Se define con frases de ejemplo (_utterances_) como «quiero reservar un cuarto».
- Un **slot** es un dato que el bot necesita reunir para cumplir la intención, por ejemplo `ciudad`, `fecha_llegada` y `noches`. Si el usuario no los da, el bot los pide; así surgen los varios turnos.
- La **respuesta** es lo que el bot contesta, a menudo después de que una función Lambda ejecutó la reservación real.

Además, Amazon Lex ofrece herramientas integradas para probar y monitorear el rendimiento del chatbot, lo que ayuda a los desarrolladores a iterar y mejorar sus bots con el tiempo. Con sus capacidades robustas y su integración fluida con otros servicios de AWS, Amazon Lex es una solución integral para construir interfaces conversacionales inteligentes que mejoren la participación de los usuarios y agilicen los procesos de negocio.

> [!warning] Estado del servicio: Lex V1
> Amazon Lex V1 dejó de tener soporte el 15-09-2025. La documentación, la consola y las API vigentes son las de **Lex V2**, que conserva los conceptos de intención, slot y respuesta.

##### Casos de uso (_Use Cases_)

Amazon Lex se usa principalmente para mejorar las experiencias de atención al cliente. Los clientes aprovechan Amazon Lex creando chatbots sofisticados que atienden consultas, dan información y resuelven problemas, minimizando la necesidad de agentes humanos y reduciendo los tiempos de respuesta. En el comercio electrónico, los bots impulsados por Lex pueden ayudar a los clientes con búsquedas de productos, seguimiento de pedidos y recomendaciones personalizadas, mejorando la experiencia de compra. Las organizaciones de salud usan Amazon Lex para automatizar la programación de citas, dar información médica y realizar la preevaluación de pacientes (_prescreening_: un cuestionario inicial de síntomas antes de la consulta). Por último, Amazon Lex se emplea en operaciones internas de negocio para crear asistentes virtuales que ayudan a los empleados con tareas como el soporte de TI, las consultas de recursos humanos y la automatización de flujos de trabajo. En general, Amazon Lex permite a las empresas construir interfaces conversacionales inteligentes que mejoran la participación de los usuarios y agilizan los procesos administrativos.

### Recomendación (_Recommendation_)

AWS ofrece capacidades de recomendación a través de su conjunto de servicios de IA. Estas capacidades buscan mejorar la experiencia de los usuarios entregando contenido personalizado.

En el centro de estas capacidades está Amazon Personalize, que permite a los desarrolladores integrar fácilmente en sus aplicaciones recomendaciones personalizadas en tiempo real. Amazon Personalize utiliza algoritmos sofisticados de ML para analizar datos de los usuarios, como el historial de navegación, el comportamiento de compra y las preferencias, y generar recomendaciones muy relevantes.

#### Amazon Personalize

Amazon Personalize funciona aprovechando algoritmos avanzados de ML para analizar los datos de los usuarios y generar recomendaciones altamente personalizadas. El proceso empieza reuniendo los datos de interacción de los usuarios con los ítems, como clics, visualizaciones y compras, además de información contextual adicional, como los metadatos de los ítems (p. ej., género, precio) y los datos demográficos de los usuarios (p. ej., edad, género). Estos datos alimentan los modelos de ML, que se entrenan para identificar patrones y relaciones entre usuarios e ítems. Los modelos pueden actualizarse continuamente con datos de interacción en tiempo real, lo que asegura que las recomendaciones sigan siendo precisas y relevantes a medida que evolucionan las preferencias de los usuarios. En la jerga de sistemas de recomendación, un **ítem** es cualquier cosa recomendable: un producto, una película, un artículo, una canción. Los datos de interacción tienen forma de tabla larga: `(usuario, ítem, tipo_de_evento, marca_de_tiempo)`, con una fila por clic o compra.

Una característica clave y única de Amazon Personalize es el uso de técnicas de IA generativa para mejorar el proceso de recomendación. Los modelos de IA generativa pueden simular nuevos puntos de datos a partir de los existentes, lo que en la práctica llena huecos del dataset y mejora la precisión de las predicciones. Esto es especialmente útil cuando se trata con datos dispersos (_sparse_) o con usuarios e ítems nuevos que tienen un historial de interacción limitado. Al generar puntos de datos sintéticos, el sistema puede entender mejor las preferencias de los usuarios y dar recomendaciones más precisas, incluso en escenarios donde los modelos tradicionales podrían tener dificultades. Además, Amazon Personalize permite personalizar según tus datos particulares. Puedes elegir los algoritmos de ML más adecuados para tu caso de uso específico y las características de tus datos, y proporcionar metadatos contextuales sobre usuarios e ítems para recomendaciones mejor informadas.

Dos términos de ese párrafo son centrales en recomendación. Los datos son **dispersos** porque la matriz usuarios × ítems está casi vacía: una tienda con un millón de clientes y 100 000 productos tiene 10¹¹ celdas posibles, y cada cliente interactuó con unas decenas; más del 99.99 % de las celdas no tiene dato. El **arranque en frío** (_cold start_) es el problema del usuario o el ítem nuevo, sin historial del cual aprender: el producto que se dio de alta hoy no tiene ni un clic. En Personalize, «elegir los algoritmos» significa elegir una **receta** (_recipe_), el nombre que da el servicio a cada algoritmo preconfigurado para un tipo de recomendación: recomendaciones personalizadas para un usuario, ítems similares a otro, reordenar una lista para un usuario, lo más popular, etc.

> [!warning] Nota de precisión: IA generativa en Personalize
> La documentación vigente de Personalize no describe la generación de puntos de datos sintéticos para rellenar el dataset. Lo que documenta como integración con IA generativa es: (1) **Content Generator**, que usa un modelo de lenguaje para escribir un «tema» que describe un conjunto de ítems recomendados (por ejemplo, el título «Para empezar bien el día» para un carrusel de productos de desayuno), y (2) la integración con **LangChain** para usar las recomendaciones dentro de aplicaciones de IA generativa. El arranque en frío se aborda con recetas que aprovechan los metadatos de los ítems y exploran ítems nuevos, no con datos sintéticos. Consulta la sección _Amazon Personalize and generative AI_ de la documentación para el detalle vigente.

Además, Amazon Personalize emplea algoritmos sofisticados para manejar diversas tareas de recomendación, como el filtrado colaborativo, el filtrado basado en contenido y los enfoques híbridos. El **filtrado colaborativo** (_collaborative filtering_) se apoya en el comportamiento colectivo de los usuarios para identificar usuarios similares y recomendar los ítems que les gustaron («a quienes compraron esto también les gustó…»); no necesita saber nada del ítem, solo quién interactuó con qué. El **filtrado basado en contenido** (_content-based filtering_) se centra en los atributos de los ítems para sugerir productos similares a aquellos en los que un usuario mostró interés («te gustó una novela negra, aquí hay otra novela negra»); por eso funciona con ítems nuevos, porque sus atributos se conocen desde el primer día. Los modelos **híbridos** combinan estos enfoques para aprovechar las fortalezas de ambos y ofrecer recomendaciones robustas y completas. Al usar estas técnicas avanzadas de IA y generativas, Amazon Personalize entrega recomendaciones altamente personalizadas y eficaces que mejoran la participación y la satisfacción de los usuarios. Su flexibilidad y su facilidad de integración aseguran que las empresas puedan adaptar el servicio a sus necesidades específicas y obtener resultados óptimos.

##### Casos de uso (_Use Cases_)

Amazon Personalize se usa ampliamente en diversas industrias para mejorar la experiencia de los usuarios mediante recomendaciones personalizadas. En el comercio electrónico, sugiere productos según el historial de navegación y el comportamiento de compra de los usuarios, lo que aumenta las tasas de conversión (el porcentaje de visitas que terminan en compra) y la satisfacción de los clientes. Los servicios de streaming y las plataformas de noticias lo aprovechan para recomendar películas, series y artículos adaptados a los intereses de cada persona, manteniendo a los usuarios comprometidos. Las empresas también lo usan en campañas de marketing dirigidas, personalizando el contenido de los correos electrónicos para promover productos y ofertas relevantes, lo que mejora la participación. Además, los equipos de atención al cliente usan Amazon Personalize para recomendar artículos de soporte relevantes, ayudando a los usuarios a resolver sus problemas de forma más eficiente.

Al entregar recomendaciones altamente relevantes y oportunas, Amazon Personalize permite a las empresas mejorar la participación de los usuarios y obtener mejores resultados en diversos sectores.

### IA generativa (_Generative AI_)

La IA generativa representa un avance de vanguardia en la inteligencia artificial: permite a los sistemas generar (de ahí viene lo de IA «generativa») contenido nuevo, como texto, imágenes o incluso experiencias multimedia completas, a partir de patrones aprendidos de los datos. La diferencia con el resto del capítulo es el tipo de salida. Un clasificador devuelve una etiqueta de un conjunto cerrado («spam» / «no spam»); un modelo generativo devuelve un objeto nuevo (un párrafo, una imagen) muestreado de la distribución que aprendió. Con Amazon Bedrock, AWS pone capacidades de IA generativa al alcance de los desarrolladores, con una plataforma fluida y escalable para construir, ajustar (_fine-tune_) y desplegar modelos de IA generativa. Amazon Bedrock da acceso a una variedad de potentes **modelos fundacionales** (FM, _foundation models_) de empresas líderes en IA, junto con las herramientas necesarias para personalizar estos modelos e integrarlos sin esfuerzo en las aplicaciones. Un modelo fundacional es un modelo muy grande, preentrenado con cantidades enormes de datos generales, que sirve de base para muchas tareas distintas sin reentrenarlo desde cero; los grandes modelos de lenguaje (LLM) son el ejemplo más conocido. Al usar Amazon Bedrock, las empresas pueden aprovechar el potencial de la IA generativa para innovar y mejorar sus soluciones digitales, impulsando la creatividad y la eficiencia en diversos ámbitos.

#### Amazon Bedrock

Amazon Bedrock es un servicio totalmente administrado que simplifica el proceso de construir y escalar aplicaciones de IA generativa usando modelos fundacionales. Da acceso a una amplia gama de modelos fundacionales de alto rendimiento de empresas líderes en IA como AI21 Labs, Anthropic, Cohere, Meta, Mistral AI y Stability AI, así como a los modelos propios de Amazon, incluidos los recién lanzados modelos fundacionales Nova.

> [!warning] Estado del catálogo de Bedrock (verificado el 24-09-2026)
> El catálogo creció mucho desde la edición del libro. Además de los proveedores citados, hoy incluye modelos de DeepSeek, Google (Gemma), MiniMax, Moonshot AI, NVIDIA, OpenAI, Qwen, TwelveLabs, Writer, xAI y Z.AI, entre otros. A la vez, algunos modelos salen: los de AI21 Labs (Jamba 1.5) están en estado _Legacy_ con fin de vida el 26-11-2026. Cada modelo tiene una **ficha** (_model card_) con su estado (_Active_, _Legacy_ o _EOL_), sus regiones, sus modalidades de entrada y salida y las API que admite. Revisa la página «Models at a glance» de la documentación antes de elegir uno.

Los **modelos fundacionales de propósito general** son modelos versátiles que pueden manejar una variedad de tareas, como la generación de texto, la traducción, el resumen y más. Estos modelos son adecuados para una amplia gama de aplicaciones en distintas industrias. Con una sola API, los desarrolladores pueden experimentar con distintos modelos, personalizarlos con sus propios datos mediante técnicas como el ajuste fino (_fine-tuning_) y la generación aumentada por recuperación (**RAG**, _retrieval augmented generation_), y desplegarlos de forma segura dentro de sus aplicaciones. Las dos técnicas de personalización son muy distintas:

- El **ajuste fino** continúa el entrenamiento del modelo con tus ejemplos (por ejemplo, 5 000 pares pregunta-respuesta de tu empresa) y produce una versión propia del modelo, con pesos modificados. Cambia _cómo_ responde el modelo: estilo, formato, vocabulario.
- **RAG** no toca el modelo. Antes de cada pregunta, un buscador recupera de tu base documental los fragmentos más relevantes y los inserta en el prompt, de modo que el modelo responde apoyándose en ellos. Cambia _qué sabe_ el modelo en esa respuesta, y se actualiza con solo actualizar los documentos.

La arquitectura **serverless** de Amazon Bedrock elimina la necesidad de administrar infraestructura, lo que permite a los desarrolladores centrarse en innovar y aportar valor a sus usuarios. «Serverless» (sin servidor) no significa que no haya servidores, sino que no son tuyos: no eliges tipo de máquina, no la enciendes ni la apagas, y pagas por uso (en Bedrock, por token procesado). Para un modelo de cientos de miles de millones de parámetros, que necesita varias GPU de gama alta solo para cargarse en memoria, esto es lo que hace viable usarlo sin un equipo de infraestructura. La **Converse API**, parte de Amazon Bedrock, ofrece una interfaz consistente y simplificada para que los desarrolladores interactúen con estos modelos fundacionales, lo que facilita integrar capacidades de IA conversacional en las aplicaciones para mejorar la experiencia de los usuarios y lograr interacciones naturales. La ventaja concreta es el formato: con la operación de bajo nivel `InvokeModel`, cada proveedor espera un cuerpo JSON distinto, así que cambiar de modelo obliga a reescribir código; con Converse, los mensajes se escriben siempre igual (`role`, `content`) y cambiar de modelo es cambiar el `modelId`.

> [!note] Recuadro del libro
> No todos los modelos fundacionales admiten la Converse API. Para ver la lista detallada de modelos y funciones compatibles, visita https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference-supported-models-features.html.
>
> _Actualización (24-09-2026):_ la ficha de cada modelo indica ahora qué API admite (`Invoke`, `Converse` y otras). Por ejemplo, Nova Lite admite ambas.

Una de las funciones destacadas de Amazon Bedrock es su **marketplace**, que ofrece una amplia selección de modelos fundacionales especializados para diversos casos de uso. Los modelos fundacionales especializados están adaptados a tareas o dominios específicos, lo que da un rendimiento más preciso y eficiente en aplicaciones particulares. Estos modelos pueden ser muy beneficiosos para industrias como la salud, las finanzas y el entretenimiento, donde el conocimiento especializado y la precisión son críticos. El marketplace permite a los desarrolladores descubrir, probar e integrar estos modelos especializados en sus flujos de trabajo de ML sin fricciones. Ya sea para generar texto, crear imágenes o realizar tareas multimodales complejas, Amazon Bedrock ofrece las herramientas y la flexibilidad necesarias para construir aplicaciones potentes de IA generativa. Una tarea **multimodal** es la que combina tipos de datos distintos, por ejemplo, responder una pregunta en texto sobre una imagen. Al aprovechar Amazon Bedrock, las empresas pueden mantenerse a la vanguardia de la innovación en IA, asegurando que sus aplicaciones sean a la vez de punta y seguras. La Converse API potencia aún más esta capacidad al ofrecer una forma directa de interactuar con estos modelos, lo que hace el proceso de integración fluido y eficiente.

> [!warning] Nota de precisión: los modelos del marketplace no son serverless
> Según la documentación de **Amazon Bedrock Marketplace**, los modelos de ese catálogo no se consumen por token como los del catálogo principal: se despliegan en un _endpoint_ administrado (servidores dedicados) para el que eliges el tipo y el número de instancias, y se cobra mientras el endpoint existe. Esto cambia el modelo de costos: pagas por hora aunque no haya tráfico. Verifica el esquema de precios vigente antes de usarlo.

Al seleccionar un modelo fundacional, es esencial considerar las capacidades específicas que necesitas para tu aplicación. Si tu tarea implica principalmente generación, análisis o traducción de texto, modelos como Amazon Titan Text G1 – Premier, Amazon Titan Text G1 – Express y Amazon Titan Text G1 – Lite son ideales. Estos modelos están integrados en Amazon Bedrock y admiten una amplia gama de tareas relacionadas con texto, como responder preguntas abiertas, generar código, resumir y conversar por chat. Para aplicaciones creativas, modelos como Amazon Nova Canvas destacan en la generación de imágenes detalladas a partir de descripciones de texto, lo que los hace perfectos para tareas de arte y diseño. Para capacidades multimodales, modelos como Amazon Nova Lite y Amazon Nova Pro pueden manejar entradas tanto de texto como de imagen, ofreciendo soluciones versátiles. Para casos de uso sencillos y presupuestos bajos, Amazon Nova Micro es un modelo solo de texto adecuado para escenarios que requieren procesamiento de texto de baja latencia sin necesidad de entrada multimodal. Evaluar las capacidades de estos modelos en el contexto de las necesidades específicas de tu aplicación te ayudará a elegir el modelo fundacional más eficaz.

> [!warning] Estado de los modelos citados (verificado el 24-09-2026)
>
> - **Titan Text G1 (Premier, Express y Lite):** ya no aparecen en el catálogo de modelos de Bedrock.
> - **Nova Canvas:** _Legacy_ desde el 30-03-2026, EOL el 30-09-2026. Los clientes nuevos no pueden usarlo.
> - **Nova Lite, Nova Pro y Nova Micro:** activos. Nova Lite acepta texto, imagen y video, y responde con texto.
> - El criterio del libro (elegir por modalidad, capacidad, latencia y presupuesto) sigue siendo válido; lo que cambia es la lista concreta de modelos. Para el examen, recuerda la lógica: _solo texto y barato_ → un modelo pequeño de texto; _entrada de imagen_ → un modelo multimodal; _generar imágenes_ → un modelo de imagen.

Otro criterio que debes considerar al seleccionar un modelo es su disponibilidad en la región donde operará tu aplicación de IA generativa. Esto asegura un rendimiento óptimo y el cumplimiento de las regulaciones locales, y reduce la latencia, ofreciendo una experiencia de usuario fluida. Recuerda que una **región** de AWS es una zona geográfica con sus propios centros de datos (por ejemplo, `us-east-1` en Virginia del Norte, `eu-west-1` en Irlanda). Las tres razones del libro se entienden así: la **latencia** crece con la distancia física que recorre la petición; las **regulaciones locales** de protección de datos pueden exigir que los datos se procesen dentro de un país o bloque (lo que se llama **residencia de datos**, _data residency_); y no todos los modelos están instalados en todas las regiones.

> [!note] Recuadro del libro
> Los modelos fundacionales están disponibles por región. Para saber qué modelos se admiten en tu región, visita https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html.
>
> _Actualización (24-09-2026):_ la información está ahora en la página «Regional availability by models» y en la ficha de cada modelo.

Para el examen, ten en cuenta que algunos modelos pueden no estar disponibles en tu región, pero aun así pueden usarse con la **inferencia entre regiones** (_cross-region inference_) si esta función está disponible en tu región. La inferencia entre regiones te permite gestionar sin fricciones picos de tráfico no planificados utilizando capacidad de cómputo de distintas regiones de AWS. Con ella, puedes distribuir el tráfico entre múltiples regiones de AWS, lo que permite un mayor **throughput** y una mayor resiliencia en los periodos de demanda máxima. El _throughput_ (rendimiento o caudal) es la cantidad de trabajo que el sistema completa por unidad de tiempo; en Bedrock se mide en peticiones o tokens por minuto, y cada cuenta tiene una cuota por región. La **resiliencia** es la capacidad de seguir funcionando cuando una parte falla o se satura: si la región de origen no tiene capacidad libre, la petición se atiende en otra. Para usar la inferencia entre regiones, necesitas crear un **perfil de inferencia entre regiones** (_cross-region inference profile_). Para saber más, visita https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-support.html.

> [!warning] Nota de precisión: perfiles de inferencia (verificado el 24-09-2026)
>
> - **No hace falta crear el perfil**: AWS ya define perfiles del sistema para los modelos compatibles, y se usan poniendo su identificador en lugar del `modelId`. Por ejemplo, `us.amazon.nova-lite-v1:0` enruta las peticiones de Nova Lite entre regiones de EE. UU. Tú creas perfiles propios (_application inference profiles_) cuando quieres, por ejemplo, separar el uso y el costo por aplicación.
> - Hay dos tipos. El **geográfico** enruta solo dentro de una geografía (EE. UU., UE, Asia-Pacífico) y respeta la residencia de datos. El **global** enruta a cualquier región comercial, sin garantía de residencia, y según la documentación cuesta aproximadamente un 10 % menos.
> - La documentación indica que el enrutamiento no tiene costo adicional (se cobra el precio de la región desde la que llamas), que los datos viajan por la red de AWS cifrados y que CloudTrail registra en tu región de origen en qué región se procesó cada petición.

Por último, igual que en cualquier aplicación bien diseñada (las aplicaciones de IA generativa no son distintas en esto), necesitas considerar los precios. Los precios de Amazon Bedrock se basan en dos modelos principales:

- Bajo demanda y por lotes (_On-Demand and Batch_)
- Throughput aprovisionado (_Provisioned Throughput_)

Con el modo bajo demanda y por lotes, pagas por el número de **tokens** de entrada y de salida procesados, donde un token es una secuencia de caracteres que el modelo interpreta como una sola unidad de significado. Un token suele ser un fragmento de palabra: como regla aproximada, en inglés una palabra equivale a algo más de un token, y en español a algo más que en inglés, porque los tokenizadores se entrenan con más texto en inglés; la cifra exacta depende de cada modelo. Los tokens de salida suelen costar varias veces más que los de entrada, así que una aplicación que genera respuestas largas cuesta más que una que solo clasifica. Con el throughput aprovisionado, te comprometes a un nivel específico de throughput durante un periodo, lo que puede ser más rentable para aplicaciones con uso constante. La lógica es la de un plan de datos móviles frente a pagar por megabyte: si tu consumo es alto y estable, el compromiso sale más barato por unidad; si es bajo o irregular, pagas capacidad ociosa. Para más información, visita https://aws.amazon.com/bedrock/pricing.

> [!warning] Nota de precisión: opciones de precio vigentes (verificado el 24-09-2026)
> La página de precios describe hoy más opciones que las dos del libro:
>
> - **Por lotes** (_Batch_): hasta un 50 % más barato que bajo demanda en los modelos que lo admiten, para trabajos que no necesitan respuesta inmediata.
> - **Throughput aprovisionado**: con compromisos de 1 o de 6 meses.
> - **Niveles de servicio** (_service tiers_) bajo demanda: _Standard_; _Priority_, más rápido y con un recargo; _Flex_, más barato y para cargas sin urgencia; y _Reserved_, con capacidad dedicada y compromiso. No todos los modelos admiten todos los niveles.
> - **Caché de prompts** (_prompt caching_) para modelos compatibles, que abarata la parte del prompt que se repite entre peticiones.
>
> Las cifras cambian; consulta siempre la página de precios vigente.

La siguiente sección ofrece un ejemplo sencillo de cómo usar la IA generativa de forma programática.

#### Uso de Nova Canvas para generar una imagen (_Using Nova Canvas to Generate an Image_)

Los modelos fundacionales Nova se anunciaron recientemente en AWS re:Invent 2024 (la conferencia anual de AWS, donde suelen anunciarse servicios nuevos). En este caso de uso vamos a usar Nova Canvas, un modelo de generación de imágenes de última generación, para generar una imagen a partir de un **prompt** textual (la instrucción en lenguaje natural que se le da al modelo). Este ejemplo te mostrará cómo implementar este caso de uso de forma programática.

> [!warning] Estado del servicio: este ejemplo ya no se puede reproducir tal cual
> Nova Canvas llega a su fin de vida el **30-09-2026** y, como está en estado _Legacy_, las cuentas nuevas ya no pueden invocarlo. El código sigue siendo útil para entender el patrón general de invocación, que es igual para cualquier modelo de imagen de Bedrock: armar un cuerpo JSON, llamar a `invoke_model` y decodificar la imagen en base64. Lo que cambia de un modelo a otro es el esquema del cuerpo JSON. Consulta en «Models at a glance» qué modelos de generación de imágenes están activos en tu región. Titan Image Generator G1 v2, que usaba el mismo esquema de petición, también está en estado _Legacy_.

Como consideración de diseño, la región predeterminada de mi cuenta de AWS es us-east-2 (Ohio). Como Nova Canvas no está disponible de forma nativa en mi región predeterminada (y no quiero implementar la inferencia entre regiones), voy a crear mi cliente de Python en us-east-1 (Virginia del Norte), donde el modelo está disponible de forma nativa en este momento. Desde la consola de AWS, abrimos Amazon Bedrock y seleccionamos us-east-1 (Virginia del Norte) como región de trabajo. Luego solicitamos acceso al modelo, como se muestra en la Figura 4.2.

_Figura 4.2 Solicitud de acceso a Nova Canvas._

> [!warning] Nota de precisión: ya no se solicita acceso a los modelos
> La página «Model access» de la Figura 4.2 ya no forma parte del flujo en las regiones comerciales (sigue existiendo en AWS GovCloud). Todos los modelos de Bedrock están habilitados por defecto: la primera invocación de un modelo de un tercero inicia automáticamente la suscripción en AWS Marketplace, para lo cual tu rol de IAM necesita los permisos `aws-marketplace:Subscribe`, `aws-marketplace:Unsubscribe` y `aws-marketplace:ViewSubscriptions`. Los modelos de Anthropic piden llenar una vez un formulario de caso de uso. Para **impedir** el uso de un modelo, hay que denegarlo explícitamente con políticas de IAM o de la organización.

El siguiente programa de Python usa el módulo `boto3` para crear un cliente de Amazon Bedrock en us-east-1 y, después, interactuar con el modelo fundacional Nova Canvas para solicitarle que genere una imagen. Ejecutamos este programa con la aplicación Code Editor de Amazon SageMaker Studio en us-east-1.

> [!note] Recuadro del libro
> Amazon SageMaker Studio es un **entorno de desarrollo integrado** (IDE, _integrated development environment_) basado en web para tus pipelines de ML de extremo a extremo. Trae de fábrica la mayoría de las bibliotecas de ML que necesitas y ofrece una experiencia de desarrollo fluida similar a Visual Studio (VS) Code.

Un IDE es un editor de código con herramientas integradas: explorador de archivos, terminal, depurador y ejecución de notebooks. Code Editor, la aplicación que usa el libro, se basa en Code-OSS, la versión de código abierto de VS Code, así que se ve y se usa igual, pero corre en una máquina de AWS a la que accedes desde el navegador.

Para empezar con Amazon SageMaker Studio, necesitas crear un **dominio** en la región donde correrá tu programa (en nuestro caso, us-east-1). Un dominio de SageMaker es la unidad de configuración de Studio: agrupa a los usuarios autorizados, sus permisos, la red que usan y el almacenamiento. Lo crea normalmente un administrador una sola vez. Una vez que tu dominio esté disponible, puedes abrir Amazon SageMaker Studio, como se muestra en la Figura 4.3. La Figura 4.4 muestra la página de inicio de Studio, donde puedes crear un **espacio** (_space_). Un espacio es una **instancia administrada** que Amazon SageMaker Studio usa para lanzar la aplicación Code Editor. Una instancia es una máquina virtual en la nube; «administrada» significa que SageMaker la crea, la configura y la apaga por ti, sin que entres a configurar el sistema operativo. En la Figura 4.4, inicié un espacio `dario-ai-space` que usa un tipo de instancia `ml.t3.medium` con 5 GB de almacenamiento.

El nombre del tipo de instancia se lee por partes: el prefijo `ml.` indica que es una instancia de SageMaker; `t3` es la familia y generación (la «t» corresponde a instancias económicas de rendimiento variable, pensadas para uso intermitente como editar código), y `medium` es el tamaño. Una `ml.t3.medium` tiene 2 CPU virtuales y 4 GiB de memoria, menos que una laptop típica de 8 a 16 GB: alcanza para escribir código y llamar a servicios, no para entrenar modelos pesados. Los **5 GB de almacenamiento** son el volumen de disco del espacio. Para dar escala: una laptop suele tener entre 256 GB y 1 TB, y solo instalar PyTorch con sus dependencias ocupa varios GB, así que 5 GB se llenan rápido. El administrador del dominio puede ampliarlo.

Para desarrollar tus aplicaciones de ML, necesitas crear e iniciar tu espacio, como se muestra en la Figura 4.5. Tu espacio tiene conectado un **Elastic File System** (EFS), donde se guardan el código de tus aplicaciones y otros artefactos. Para no incurrir en costos no deseados, asegúrate de detener tu espacio cuando termines de programar. La razón del costo es que una instancia se cobra por hora mientras está encendida, la uses o no; olvidar un espacio encendido todo un fin de semana es la forma más común de recibir una factura inesperada.

Para entender la afirmación sobre EFS, conviene distinguir las tres familias de almacenamiento en la nube:

- **Almacenamiento de bloques.** Es un disco virtual que se conecta a una sola máquina, como el SSD de tu laptop. El sistema operativo lo formatea y lo usa como disco propio. En AWS es **Amazon EBS** (_Elastic Block Store_).
- **Almacenamiento de archivos.** Es una carpeta compartida por red que varias máquinas «montan» a la vez, como la unidad de red de una oficina. **Montar** significa hacer que esa carpeta remota aparezca dentro del sistema de archivos local, de modo que los programas la usan como si fuera un directorio más (`/home/sagemaker-user/...`). En AWS son **Amazon EFS** y la familia **Amazon FSx**. EFS usa **NFS** (_Network File System_), el protocolo estándar en Linux para compartir carpetas por red.
- **Almacenamiento de objetos.** Cada archivo se guarda entero como un objeto, identificado por una clave, dentro de un contenedor llamado _bucket_, y se lee y escribe mediante una API web, no montándolo como disco. En AWS es **Amazon S3**.

> [!warning] Nota de precisión: en el Studio actual, el espacio usa EBS, no EFS (verificado el 24-09-2026)
> Según la documentación vigente, cada espacio de **Code Editor** o **JupyterLab** del Studio actual usa **un volumen de Amazon EBS** (almacenamiento de bloques) para todo su contenido: código, perfil de Git y variables de entorno. El tamaño predeterminado es de 5 GB, justo el que muestra el libro. **Amazon EFS** era el almacenamiento de **Studio Classic**, la versión anterior de la interfaz. La conclusión práctica del libro se mantiene: los archivos persisten entre sesiones, porque el volumen EBS sobrevive cuando detienes el espacio. Lo que cambia es que ese volumen pertenece a un solo espacio y no es una carpeta compartida entre usuarios.
>
> Según la misma documentación, ese volumen ofrece 3 000 **IOPS** y 125 MB/s de **throughput**. Las IOPS (_input/output operations per second_) miden cuántas lecturas o escrituras pequeñas por segundo admite el disco, y el throughput cuántos megabytes por segundo transfiere en lecturas grandes. Para dar escala: un disco duro mecánico ronda los cientos de IOPS, y el SSD NVMe de una laptop moderna llega a decenas o cientos de miles de IOPS y a varios GB/s. Es decir, el disco del espacio es más lento que el de tu laptop: leer un dataset de 10 GB a 125 MB/s toma alrededor de 80 segundos. Para datos grandes, lo habitual es leerlos directamente de S3 en lugar de copiarlos al espacio.

_Figura 4.3 Amazon SageMaker Studio._

_Figura 4.4 Espacio de Amazon SageMaker Studio._

_Figura 4.5 Code Editor de Amazon SageMaker Studio._

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

Este programa utiliza el módulo `boto3` para construir un objeto cliente llamado `bedrock` que interactúa con Amazon Bedrock en us-east-1, donde Nova Canvas está disponible de forma nativa. Como se ve en el método `generate_image`, este cliente usa el método `invoke_model` para enviar un prompt (encapsulado como la propiedad `text` del parámetro `body`) que pide crear una imagen de una playa de arena blanca al atardecer.

Algunos detalles del código que el libro no comenta:

- **`service_name='bedrock-runtime'`.** Bedrock expone dos clientes: `bedrock` sirve para administrar (listar modelos, crear perfiles) y `bedrock-runtime` para invocar modelos. Confundirlos es un error frecuente.
- **La región.** El código no pasa `region_name` a `boto3.client`, así que usa la región configurada en el entorno. En el Studio del libro es us-east-1 porque el dominio está ahí; en otra máquina, conviene escribir `region_name='us-east-1'` de forma explícita.
- **`Config(read_timeout=300)`.** Espera hasta 300 segundos (5 minutos) la respuesta. Generar una imagen tarda mucho más que una llamada típica de API, y el tiempo de espera predeterminado de boto3 podría cortar la conexión antes.
- **`accept` y `contentType`.** Son **tipos MIME**, etiquetas estándar que dicen en qué formato va el contenido. Aquí, JSON en la petición y JSON en la respuesta.
- **`cfgScale`.** Controla cuánto se apega la imagen al prompt: valores altos siguen el texto más literalmente, a costa de variedad. **`seed`** fija la semilla aleatoria, igual que `np.random.seed`: con la misma semilla y el mismo prompt, el modelo tiende a producir la misma imagen.
- **Base64.** La respuesta JSON solo puede llevar texto, así que la imagen (datos binarios) viaja codificada en **base64**, un esquema que representa bytes arbitrarios con 64 caracteres imprimibles; ocupa alrededor de un 33 % más que el binario. `b64decode` recupera los bytes originales del PNG y la biblioteca PIL (Pillow) los abre como imagen.
- **`ClientError`.** Es la excepción que lanza boto3 cuando AWS rechaza la petición: falta de permisos, modelo no disponible en la región, cuota superada. Hoy, con Nova Canvas en estado _Legacy_, una cuenta nueva recibiría aquí un error de acceso.

Al ejecutarse en Amazon SageMaker Studio, este código genera la imagen solicitada, como se muestra en las Figuras 4.6 y 4.7.

_Figura 4.6 Salida del programa._

_Figura 4.7 Una imagen generada por Nova Canvas._

Este programa no se ejecutó para esta versión: requiere una cuenta de AWS, genera costos y el modelo ya no admite clientes nuevos. Según el código, una ejecución exitosa imprime `Image saved to /home/sagemaker-user/ch04/images/white_sand_beach_sunset.png` y luego `Finished generating image with Amazon Nova Canvas model amazon.nova-canvas-v1:0.`, además de los mensajes de registro con prefijo `INFO:`.

Una vez generada la imagen, el programa decodifica los datos de la imagen en base64 y la guarda como `white_sand_beach_sunset.png` en la carpeta `/home/sagemaker-user/ch04/images` del EFS montado en tu instancia de Amazon SageMaker Studio.

> [!note] Recuadro del libro
> Las instancias de Amazon SageMaker Studio vienen con EFS montado por defecto, así que cualquier archivo guardado en directorios locales de la instancia se almacena en el EFS conectado. Esto asegura que nuestra imagen generada esté disponible y persista entre distintas sesiones de Amazon SageMaker Studio.
>
> _Ver la nota de precisión anterior: en los espacios del Studio actual, ese almacenamiento es un volumen EBS por espacio. La persistencia entre sesiones se mantiene._

El programa también incluye un manejo de errores completo para asegurar un proceso fluido, registrando mensajes significativos a lo largo del camino. Esta configuración asegura que la imagen se genere y se almacene de forma eficiente, lista para usarse o mostrarse después.

> [!note] Recuadro del libro
> Mientras se escribía este libro, Amazon SageMaker pasó a llamarse Amazon SageMaker AI. Para el examen, ten en cuenta este cambio de nombre. Siempre que el libro menciona Amazon SageMaker (el servicio), se refiere a Amazon SageMaker AI (https://aws.amazon.com/blogs/aws/introducing-the-next-generation-of-amazon-sagemaker-the-center-for-all-your-data-analytics-and-ai).
>
> _El nombre «Amazon SageMaker» designa ahora una plataforma más amplia de datos, analítica e IA, de la que SageMaker AI es la parte de ML._

##### Casos de uso (_Use Cases_)

Amazon Bedrock es una excelente opción para desarrolladores que buscan construir aplicaciones de IA generativa con modelos fundacionales robustos. Es ideal para tareas como la generación de texto, la creación de imágenes y el procesamiento multimodal, gracias a su integración con modelos de alto rendimiento como Amazon Titan y Amazon Nova. Si necesitas capacidades de IA escalables, confiables y fáciles de usar con infraestructura administrada, Amazon Bedrock ofrece un entorno fluido tanto para la experimentación como para el despliegue. Además, el esquema de pago por uso de Amazon Bedrock lo hace accesible para proyectos de distintos tamaños, desde pequeñas startups hasta grandes empresas.

Por el contrario, Amazon Bedrock podría no ser la mejor opción si tu aplicación requiere modelos altamente especializados o de un dominio específico que no estén disponibles en la plataforma ni en el marketplace. Si necesitas modelos entrenados con datos propietarios, tienes requisitos estrictos de residencia de datos o requisitos de latencia que exigen soluciones muy especializadas, otras plataformas o modelos construidos a la medida podrían ser más apropiados. Además, si el costo es una preocupación principal y tu proyecto tiene necesidades mínimas de IA, soluciones más sencillas y rentables podrían bastar sin las capacidades avanzadas de Amazon Bedrock.

> [!warning] Nota de precisión: los límites de Bedrock son menos estrictos de lo que sugiere el libro
>
> - **Datos propietarios.** Bedrock permite ajustar modelos con tus datos (_fine-tuning_) en los modelos que lo admiten, y RAG no requiere reentrenar nada. También permite importar pesos de modelos propios de ciertas arquitecturas (_Custom Model Import_). Verifica en la documentación qué modelos y arquitecturas admiten cada opción.
> - **Residencia de datos.** La inferencia en una sola región o la inferencia entre regiones _geográfica_ (ver la nota anterior) mantienen el procesamiento dentro de una región o de una geografía.
> - La alternativa dentro de AWS cuando de verdad necesitas control total del modelo (pesos propios, hardware específico, latencia muy ajustada) es desplegarlo tú mismo en **SageMaker AI**, que es el tema de la siguiente sección.

## Desarrollo de modelos con los algoritmos integrados de Amazon SageMaker (_Developing Models with Amazon SageMaker Built-in Algorithms_)

Aunque los servicios de IA de AWS como Amazon Rekognition, Comprehend y Personalize ofrecen soluciones potentes y prediseñadas para tareas comunes como el análisis de imágenes, el procesamiento de texto y los sistemas de recomendación, pueden no dar la flexibilidad necesaria para casos de uso especializados. Estos servicios administrados están diseñados para ser fáciles de usar y requerir poca experiencia en ML, lo que los hace adecuados para despliegues rápidos y aplicaciones estándar. Sin embargo, cuando tu proyecto exige ingeniería de características personalizada, arquitecturas de modelo avanzadas u optimización para métricas de rendimiento específicas, los algoritmos integrados de Amazon SageMaker ofrecen las herramientas y la flexibilidad necesarias para lograr esos objetivos.

Un **algoritmo integrado** (_built-in algorithm_) es una implementación de un algoritmo que AWS empaqueta y mantiene dentro de una **imagen de contenedor**. Un contenedor (en la práctica, una imagen de Docker) es un paquete que incluye el programa y todas sus dependencias, de modo que corre igual en cualquier máquina; es como un entorno de conda congelado que se puede llevar a cualquier servidor. Con un algoritmo integrado, tú no escribes el código de entrenamiento: aportas los datos y los hiperparámetros, y SageMaker hace lo siguiente:

1. Enciende las instancias que pediste para un **trabajo de entrenamiento** (_training job_).
2. Descarga el contenedor del algoritmo y copia (o transmite) tus datos a esas instancias.
3. Ejecuta el entrenamiento y guarda el modelo resultante (un archivo `model.tar.gz`) en S3.
4. Apaga las instancias. Pagas por segundo de uso mientras el trabajo corre.

Después, el modelo guardado se puede **desplegar** (_deploy_) de dos formas. Una es un **endpoint** de inferencia en tiempo real: un servidor que queda encendido y responde peticiones HTTPS en milisegundos, y que se cobra por hora mientras exista, reciba tráfico o no. La otra es una **transformación por lotes** (_batch transform_): un trabajo que se enciende, puntúa un archivo completo de S3 y se apaga. Recuerda que en el vocabulario de ML y de AWS, **inferencia** significa usar un modelo ya entrenado para producir predicciones sobre datos nuevos, no la inferencia estadística.

Elegir construir modelos con los algoritmos integrados de Amazon SageMaker ofrece varias ventajas, en particular cuando necesitas mayor control y personalización sobre tus proyectos de ML. Los algoritmos integrados de Amazon SageMaker están altamente optimizados en velocidad, escala y precisión, lo que te permite entrenar, refinar y desplegar de forma eficiente modelos adaptados a tus necesidades específicas de negocio. Este enfoque es ideal cuando necesitas ajustar hiperparámetros, incorporar conocimiento del dominio o manejar requisitos particulares de preprocesamiento de datos. Al aprovechar los algoritmos integrados de Amazon SageMaker, puedes lograr un mayor grado de precisión y rendimiento, sobre todo en aplicaciones complejas o hechas a la medida donde las soluciones listas para usar (_off-the-shelf_) pueden quedarse cortas.

Además, Amazon SageMaker ofrece una integración fluida con el ecosistema más amplio de AWS, lo que asegura un flujo de trabajo coherente desde la preparación de los datos hasta el despliegue del modelo. Puedes importar fácilmente datos desde Amazon S3, Amazon EFS o Amazon FSx for Lustre; utilizar funciones de AWS Lambda para el preprocesamiento, y desplegar modelos mediante endpoints de Amazon SageMaker o AWS Lambda para la inferencia en tiempo real. Esta integración estrecha simplifica el ciclo de vida de ML, mejora la eficiencia operativa y reduce el tiempo y el esfuerzo necesarios para gestionar los distintos componentes del pipeline de ML. También te permite aprovechar las capacidades de **entrenamiento distribuido** de Amazon SageMaker, que pueden acelerar significativamente el entrenamiento del modelo con datasets grandes. El entrenamiento distribuido reparte el trabajo entre varias instancias (o varias GPU): por ejemplo, cada una procesa una parte de los datos y comparten los resultados parciales, de modo que un entrenamiento que en una máquina tardaría diez horas puede terminar en poco más de una hora con diez máquinas, si el algoritmo escala bien.

Las tres fuentes de datos de entrenamiento que menciona el libro (y que se repiten en cada algoritmo de este capítulo) corresponden a las tres familias de almacenamiento descritas en la sección de Nova Canvas:

- **Amazon S3** (almacenamiento de objetos) es la opción predeterminada y la más barata por GB. SageMaker puede copiar los datos de S3 al disco de la instancia antes de empezar (**modo File**) o transmitirlos directamente desde S3 mientras entrena (**modo Pipe**), lo que arranca más rápido y requiere menos disco local. Los algoritmos integrados de este capítulo documentan qué modos admiten.
- **Amazon EFS** (almacenamiento de archivos por NFS) conviene cuando los datos ya viven en una carpeta compartida que usan otras máquinas, porque evita copiarlos a S3.
- **Amazon FSx for Lustre** es un **sistema de archivos paralelo** administrado. Lustre es un sistema de archivos de supercómputo que reparte cada archivo entre muchos servidores de almacenamiento, de modo que cientos de procesos pueden leer a la vez sin formar cola. Puede vincularse a un bucket de S3 y presentar sus objetos como archivos. Importa cuando el cuello de botella del entrenamiento es leer datos (por ejemplo, millones de imágenes pequeñas que alimentan varias GPU): las GPU son caras y no deben quedarse esperando al disco.

En cuanto a **AWS Lambda** para inferencia: funciona para modelos pequeños y tráfico irregular, porque solo se paga cuando llega una petición. Tiene límites de memoria, de tamaño del paquete y de duración por ejecución, y no ofrece GPU, así que no sirve para modelos grandes. Verifica los límites vigentes en la documentación de Lambda.

En esta sección exploraremos los algoritmos integrados que ofrece Amazon SageMaker, sus casos de uso apropiados y cómo utilizarlos eficazmente para construir modelos de ML robustos. La Figura 4.8 puede ayudarte a relacionar los algoritmos integrados con tipos específicos de problemas de ML, formatos de entrada y casos de uso.

_Figura 4.8 Algoritmos integrados de Amazon SageMaker._

Como la figura no está en el `.txt`, esta es la tabla equivalente de la documentación vigente de SageMaker AI (verificada el 24-09-2026):

| Tipo de problema | Algoritmos integrados |
| --- | --- |
| Clasificación binaria o multiclase, y regresión (datos tabulares) | AutoGluon-Tabular, CatBoost, Factorization Machines, k-NN, LightGBM, Linear Learner, TabTransformer, XGBoost |
| Pronóstico de series de tiempo | DeepAR |
| Embeddings (objetos de alta dimensión a baja dimensión) | Object2Vec |
| Reducción de dimensionalidad | PCA |
| Detección de anomalías | Random Cut Forest (RCF) |
| Anomalías en direcciones IP | IP Insights |
| Clustering | K-Means |
| Modelado de temas | LDA, NTM |
| Clasificación de texto | BlazingText, Text Classification - TensorFlow |
| Traducción, resumen, voz a texto | Sequence-to-Sequence |
| Clasificación de imágenes | Image Classification - MXNet, Image Classification - TensorFlow |
| Detección de objetos | Object Detection - MXNet, Object Detection - TensorFlow |
| Segmentación semántica | Semantic Segmentation |

> [!warning] Nota de precisión: la lista de algoritmos integrados
>
> - La documentación incluye algoritmos tabulares que el libro no menciona: **AutoGluon-Tabular**, **CatBoost**, **LightGBM** y **TabTransformer**. LightGBM y CatBoost son implementaciones de _gradient boosting_ de la misma familia que XGBoost.
> - **Random Forest no es un algoritmo integrado de SageMaker**, aunque el libro lo presente junto a XGBoost y lo repita en «Puntos esenciales» y en la pregunta 6. Para entrenar un random forest en SageMaker se usa scikit-learn dentro de un contenedor de _framework_ (tu propio script de `sklearn` ejecutado como trabajo de entrenamiento). No lo confundas con **Random Cut Forest**, que sí es integrado, pero sirve para detectar anomalías.
> - Además de los algoritmos integrados, **SageMaker JumpStart** ofrece modelos preentrenados listos para ajustar o desplegar (por ejemplo, YOLO, Faster R-CNN o BERT).

### Algoritmos de ML supervisado (_Supervised ML Algorithms_)

Como aprendiste en el capítulo 1, el ML supervisado es un tipo de algoritmo que aprende de datos de entrenamiento etiquetados para hacer predicciones o tomar decisiones. En el aprendizaje supervisado, cada ejemplo de entrenamiento incluye los datos de entrada, en forma de un conjunto de características, junto con la salida o **etiqueta** (_label_) correspondiente. El algoritmo usa estos ejemplos etiquetados para aprender la correspondencia entre entradas y salidas, y luego puede aplicar esa correspondencia aprendida a datos nuevos, no vistos, para predecir resultados.

Este enfoque es especialmente útil cuando existe una relación clara y predefinida entre las características de entrada y la etiqueta correspondiente, también llamada **variable objetivo** (_target variable_). En vocabulario estadístico, las características son las covariables o predictores y la variable objetivo es la respuesta. Entre las aplicaciones comunes del aprendizaje supervisado están tareas como la **regresión**, donde el objetivo es predecir un valor continuo (p. ej., predecir el precio de una casa a partir de características como el tamaño y la ubicación), y la **clasificación**, donde el objetivo es categorizar los datos en clases discretas (p. ej., identificar si un correo es spam o no).

El aprendizaje supervisado debe usarse cuando hay datos etiquetados disponibles y el objetivo es hacer predicciones precisas a partir de ellos. Dentro del aprendizaje supervisado, hay varios caminos según la naturaleza de tu problema de ML.

Para problemas de **regresión**, pueden emplearse algoritmos como la regresión lineal, los árboles de decisión y XGBoost para predecir valores continuos.

En la **clasificación binaria**, algoritmos como la regresión logística, las máquinas de vectores de soporte (SVM) y los k vecinos más cercanos (k-NN) ayudan a distinguir entre dos clases.

Para la **clasificación multiclase**, donde los datos deben categorizarse en más de dos clases, suelen usarse enfoques como los árboles de decisión, los random forests y las redes neuronales.

Además, el aprendizaje supervisado abarca algoritmos de pronóstico de series de tiempo como DeepAR, que predice valores futuros a partir de observaciones pasadas. También hay algoritmos como Object2Vec, que se usan para crear **embeddings** que capturan información semántica de objetos o entidades, lo que mejora el rendimiento de tareas posteriores (_downstream_) como los sistemas de recomendación o la búsqueda semántica. Un embedding es un vector numérico denso (por ejemplo, de 64 números) que representa un objeto de modo que objetos parecidos quedan cerca en ese espacio; se desarrolla más adelante, en la sección de BlazingText. Una tarea _downstream_ es cualquier modelo posterior que usa esos vectores como características de entrada.

Estas asignaciones son orientativas: la regresión logística admite varias clases (con la función _softmax_) y los árboles sirven tanto para regresión como para clasificación. Lo que el libro enumera son los usos típicos.

#### Algoritmos generales de regresión y clasificación (_General Regression and Classification Algorithms_)

Esta sección presenta los algoritmos supervisados de regresión y clasificación más comunes que ofrece Amazon SageMaker.

##### Linear Learner

Empezaremos con el algoritmo **Linear Learner** de Amazon SageMaker, que ofrece una solución tanto para problemas de clasificación como de regresión.

En los casos de uso de regresión lineal, le das al algoritmo muestras etiquetadas $(\mathbf{x}, y)$. $\mathbf{x}$ es un vector de alta dimensión de valores de características, $\mathbf{x} = (x_1, x_2, \dots, x_d) \in \mathbb{R}^d$, e $y$ es un número real que denota la etiqueta asociada al punto de datos de la muestra. (Los símbolos se perdieron en el `.txt`; esta es la notación estándar de la documentación de SageMaker.)

- Para problemas de clasificación binaria, la etiqueta debe ser 0 o 1.
- Para problemas de clasificación multiclase, las etiquetas deben ir de 0 a `num_classes – 1`.

El algoritmo aprende una función lineal o, en problemas de clasificación, una función lineal con umbral, y relaciona un vector $\mathbf{x}$ con una aproximación de la etiqueta $y$. «Función lineal con umbral» significa que calcula una combinación lineal $\mathbf{w}^\top\mathbf{x} + b$ y asigna la clase según si el resultado supera un umbral.

El algoritmo Linear Learner requiere una matriz de datos, con filas que representan las observaciones (o tus puntos de datos) y columnas que representan las dimensiones de las características. También requiere una columna adicional que contenga las etiquetas correspondientes a los puntos de datos. Como mínimo, Linear Learner de Amazon SageMaker requiere que especifiques como argumentos las ubicaciones de los datos de entrada y de salida y el tipo de objetivo (clasificación o regresión). La dimensión de las características también es obligatoria. Para más información, consulta https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateTrainingJob.html. `CreateTrainingJob` es la operación de la API de SageMaker que lanza un trabajo de entrenamiento; el SDK de Python la llama por ti cuando ejecutas `estimator.fit(...)`.

> [!warning] Nota de precisión: `feature_dim` ya no es obligatorio (verificado el 24-09-2026)
> En la documentación vigente, los únicos hiperparámetros obligatorios de Linear Learner son `predictor_type` (`binary_classifier`, `multiclass_classifier` o `regressor`) y, solo para multiclase, `num_classes`. La dimensión de las características, `feature_dim`, es opcional y su valor predeterminado es `auto`, que la infiere de los datos.

El algoritmo Linear Learner de Amazon SageMaker requiere que los datos de entrada estén en formato de archivo CSV o **RecordIO-protobuf**.

RecordIO-protobuf es un formato binario: cada observación se convierte en una secuencia de números de punto flotante de 4 bytes empaquetada con _Protocol Buffers_, un formato de serialización de Google. Se lee más rápido que el CSV (no hay que interpretar texto) y admite datos dispersos guardando solo los valores distintos de cero. El CSV, por su parte, tiene reglas estrictas en SageMaker: **sin fila de encabezado**, con la **variable objetivo en la primera columna**, y declarando el tipo de contenido `text/csv` en el canal de entrenamiento. Olvidar cualquiera de las tres es una causa común de trabajos fallidos.

Puedes especificar parámetros adicionales en el mapa de cadenas `HyperParameters` del cuerpo de la petición. Estos parámetros controlan el procedimiento de optimización o aspectos específicos de la función objetivo con la que entrenas, como el número de **épocas** (_epochs_: pasadas completas sobre los datos de entrenamiento), la regularización y el tipo de pérdida. «Mapa de cadenas» significa que todos los valores se envían como texto (`{"epochs": "15"}`), aunque sean números. Según la documentación, la regularización se controla con `l1` (penalización L1, como en lasso) y `wd` (_weight decay_, penalización L2, como en ridge). Dos comportamientos predeterminados sorprenden a quien viene de la estadística: Linear Learner **estandariza las características** por defecto (`normalize_data=true`) y **entrena varios modelos en paralelo** con configuraciones cercanas (`num_models=auto`), quedándose con el mejor en los datos de validación.

Profundicemos en la regresión lineal y la regresión logística, que son los casos de uso más comunes del modelado lineal para problemas de ML de regresión y de clasificación, respectivamente. Las máquinas de vectores de soporte (SVM) también se incluyen como otra opción del algoritmo Linear Learner.

> [!warning] Nota de precisión: qué SVM ofrece Linear Learner
> Linear Learner solo produce **SVM lineales**: se obtienen eligiendo la pérdida `hinge_loss` para clasificación binaria, o las pérdidas `eps_insensitive_squared_loss` y `eps_insensitive_absolute_loss` para una regresión tipo SVR. **No admite kernels** (polinomial, RBF), que el libro describe más abajo. Para un SVM con kernel en SageMaker, hay que usar scikit-learn (`sklearn.svm.SVC`/`SVR`) en un contenedor de framework.

###### Regresión lineal (_Linear Regression_)

La regresión lineal es uno de los algoritmos más simples y más usados del ML supervisado. Su objetivo principal es modelar la relación entre una variable dependiente (también llamada variable objetivo o de salida) y una o más variables independientes (llamadas características o predictores). Lo hace ajustando una ecuación lineal a los datos observados. La ecuación lineal puede escribirse de la siguiente forma:

$$y = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \dots + \beta_n x_n + \varepsilon$$

donde $y$ es la salida real observada, $\beta_0$ es el intercepto, $\beta_1, \beta_2, \dots, \beta_n$ son los coeficientes (o pesos) de las características $x_1, x_2, \dots, x_n$, y $\varepsilon$ representa el error, también llamado residuo (la diferencia entre los valores reales y los predichos).

Tu objetivo es predecir los coeficientes $\beta_0, \dots, \beta_n$ que el modelo aprenderá para minimizar el error $\varepsilon$ en todo el dataset.

> [!warning] Nota de precisión: error frente a residuo, y «predecir» coeficientes
> En estadística, $\varepsilon$ es el **error** (no observable) del modelo poblacional, y el **residuo** es su contraparte estimada, $e_i = y_i - \hat{y}_i$. El libro los trata como sinónimos. Del mismo modo, los coeficientes no se «predicen»: se **estiman**, y lo que se minimiza es una función de los residuos (aquí, su media cuadrática), no el error en sí.

Al evaluar el modelo, observamos los residuos para entender qué tan bien está funcionando. En la regresión lineal, el modelo busca la función lineal que minimiza el **error cuadrático medio** (MSE, _mean squared error_). La fórmula del MSE es la siguiente:

$$\text{MSE} = \frac{1}{n} \sum_{i=1}^{n} \left(y_i - \hat{y}_i\right)^2$$

donde $n$ denota el número de observaciones de tu dataset, $y_i$ denota el valor real de la observación $i$, y $\hat{y}_i$ denota el valor predicho para la observación $i$.

Minimizar el MSE ayuda a asegurar que las predicciones estén lo más cerca posible de los valores reales, lo que reduce efectivamente el error global.

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

Veamos cómo funciona este programa:

- **Generación de datos.** Generamos una matriz $X$ con 100 filas (muestras) y 1 columna (característica). Cada fila denota el valor (observación o muestra) de una sola característica, que es un número aleatorio distribuido uniformemente entre 0 y 2. Luego definimos una relación lineal $y = 4 + 3X$ con algo de residuo añadido. En el programa, $y$ representa el vector de variables objetivo correspondientes a cada muestra. El «residuo añadido» es ruido normal estándar, $\varepsilon \sim N(0, 1)$, generado con `np.random.randn`.
- **Ajuste del modelo.** Creamos y ajustamos un modelo de regresión lineal con `LinearRegression` de scikit-learn.
- **Visualización.** Como se ilustra en la Figura 4.9, graficamos los puntos de datos y la recta de regresión. Por último, dibujamos líneas punteadas para mostrar los residuos (las diferencias entre los valores reales y los predichos) de cada punto de datos (100 en total).

_Figura 4.9 Regresión lineal._

- **MSE.** Calculamos e imprimimos el MSE para cuantificar el rendimiento del modelo.

Salida real (scikit-learn 1.5.2, numpy 2.0.2):

```text
Mean Squared Error: 0.9924386487246479
```

Los coeficientes estimados, que el programa no imprime, son $\hat{\beta}_0 = 4.222$ y $\hat{\beta}_1 = 2.968$, cerca de los valores reales 4 y 3. El MSE de 0.99 tampoco es casual: el ruido simulado tiene varianza 1, y el MSE en entrenamiento de un modelo bien especificado se acerca a esa varianza. Es decir, el modelo ya explica todo lo explicable y el resto es ruido irreducible. Observa también que el MSE se calcula con los mismos datos de ajuste, así que mide el ajuste, no la capacidad de generalizar; en el capítulo 5 se usa un conjunto de prueba aparte.

###### Regresión logística (_Logistic Regression_)

A pesar de su nombre, la regresión logística es un algoritmo de clasificación, que permite modelar la probabilidad de una predicción de categoría correcta mediante una combinación lineal de las características de entrada. Este algoritmo es especialmente útil cuando la variable dependiente es binaria, es decir, cuando solo puede tomar dos resultados posibles: predicción de categoría correcta o predicción de categoría incorrecta.

> [!warning] Nota de precisión: qué probabilidad modela la regresión logística
> La regresión logística modela $P(Y = 1 \mid \mathbf{x})$, la probabilidad de que la observación pertenezca a la **clase positiva** (por ejemplo, «es spam»), no la probabilidad de que la predicción sea «correcta». Los dos resultados posibles de la variable dependiente son las dos clases (spam / no spam), no «acierto / error». El nombre «regresión» viene de que es un modelo lineal generalizado: una regresión sobre el logit de la probabilidad.

La regresión logística funciona aplicando una **función logística** a la combinación lineal de las características de entrada, lo que transforma la salida para que caiga en el rango entre 0 y 1. Esta salida puede interpretarse entonces como la probabilidad de que ocurra el evento, y se usa un umbral (comúnmente 0.5) para clasificar el evento en una de las dos categorías posibles.

La función logística, también conocida como **función sigmoide**, es clave en la regresión logística y se define así:

$$\sigma(z) = \frac{1}{1 + e^{-z}}$$

En la fórmula, $z$ es la combinación lineal de las características de entrada y sus coeficientes correspondientes:

$$z = \beta_0 + \beta_1 x_1 + \beta_2 x_2 + \dots + \beta_n x_n$$

Como se ilustra en la Figura 4.10, la función logística transforma cualquier número real en una medida de probabilidad, es decir, en el intervalo unitario (0, 1), lo que la hace adecuada para estimar probabilidades. La curva en forma de S de la función sigmoide asegura que, cuando la entrada $z$ se vuelve muy grande o muy pequeña, la salida se acerca a 1 o a 0, respectivamente, pero nunca alcanza esos valores extremos. Esta característica es crucial para modelar probabilidades, ya que asegura que las probabilidades predichas siempre estén dentro de un rango válido.

_Figura 4.10 Función logística._

La mejor forma de entender cómo funciona un algoritmo es con un ejemplo sencillo. El siguiente programa en Python usa el dataset Iris para predecir si una flor es de la especie Iris-virginica a partir del largo y el ancho del sépalo. El programa usa un algoritmo de regresión logística y grafica la frontera de decisión, así como las estimaciones de probabilidad que da la función sigmoide.

> [!note] Recuadro del libro
> El dataset Iris es un dataset popular que contiene mediciones de distintas características de flores Iris (largo del sépalo, ancho del sépalo, largo del pétalo, ancho del pétalo) de tres especies (Iris-setosa, Iris-versicolor, Iris-virginica). Este dataset está disponible en la función `load_iris` del módulo `sklearn.datasets`. Para más información, visita https://scikit-learn.org/1.5/auto_examples/datasets/plot_iris_dataset.html.
>
> _Tiene 150 flores, 50 de cada especie, y las medidas están en centímetros._

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

- **Cargar y preprocesar los datos.** Se carga el dataset Iris y se seleccionan solo las características largo y ancho del sépalo. La variable objetivo se convierte a formato binario, que indica si la flor es Iris-virginica.
- **Dividir los datos.** Los datos se dividen en conjuntos de entrenamiento y de prueba con `train_test_split` (80 % y 20 %: 120 y 30 flores). Esto ayuda a evaluar el rendimiento del modelo con datos no vistos.
- **Entrenar el modelo de regresión logística.** Se instancia un modelo de regresión logística y se entrena con los datos de entrenamiento mediante el método `fit`. Por defecto, `LogisticRegression` de scikit-learn aplica una penalización L2 (`C=1.0`), así que los coeficientes quedan algo encogidos respecto de la estimación de máxima verosimilitud sin penalizar que daría un paquete estadístico clásico.
- **Generar una cuadrícula de puntos.** Se crea con `np.meshgrid` una cuadrícula de puntos que cubre el espacio de características (largo y ancho del sépalo). Esta cuadrícula servirá para visualizar la frontera de decisión y las probabilidades.
- **Calcular las probabilidades.** Se usa el modelo entrenado para predecir, con el método `predict_proba`, la probabilidad de que cada punto de la cuadrícula pertenezca a la clase Iris-virginica. Luego estas probabilidades se reorganizan para que coincidan con la forma de la cuadrícula.
- **Graficar los resultados.** La frontera de decisión y las probabilidades se grafican con `contourf`. Encima se grafican los puntos de datos con `scatter`. Se añade una barra de color para indicar la escala de probabilidad y una leyenda que indica qué representan los puntos circulares y cuadrados (_Not Iris-Virginica_ e _Iris-Virginica_).

Este programa demuestra la capacidad de la regresión logística de crear una frontera de decisión lineal y visualiza cómo varían las probabilidades del modelo en el espacio de características. La Figura 4.11 muestra la gráfica que genera este programa.

_Figura 4.11 Rangos de probabilidad calculados con regresión logística._

Al ejecutarlo (scikit-learn 1.5.2), el modelo ajustado es $z = -10.845 + 1.995\,x_{\text{largo}} - 0.605\,x_{\text{ancho}}$. Clasifica bien 27 de las 30 flores de prueba (exactitud de 0.90) y el 77.5 % de las de entrenamiento. La frontera de decisión ($p = 0.5$, es decir, $z = 0$) es una recta que deja a la derecha las flores de sépalo largo.

En la Figura 4.11, los puntos cuadrados indican muestras reales de Iris-virginica, mientras que los puntos circulares indican otras especies de Iris (Iris-setosa o Iris-versicolor). Observa cómo los rangos de probabilidad están separados entre sí por líneas. Esto se debe a que el algoritmo de regresión logística intenta relacionar las características (en este caso, el largo y el ancho del sépalo) mediante una ecuación lineal.

Durante el entrenamiento, un algoritmo de regresión logística trata de encontrar una frontera de decisión lineal que separe las dos clases. Profundicemos un poco más.

Matemáticamente, el algoritmo de regresión logística ajusta una combinación lineal de las características de entrada a las **log-odds** (el logaritmo de las momios) del resultado binario.

En nuestro ejemplo, el resultado binario es el evento de que una muestra de datos sea realmente una Iris-virginica, y $p$ denota la probabilidad de que ocurra ese evento:

$$\ln\!\left(\frac{p}{1-p}\right) = \beta_0 + \beta_1 x_1 + \beta_2 x_2 = z$$

La última parte de la ecuación se conoce como la **función logit** y devuelve las log-odds de $p$, que luego pasan por la función logística (sigmoide) para producir una probabilidad entre 0 y 1:

$$p = \sigma(z) = \frac{1}{1 + e^{-z}}$$

Como el logit $z$ es una combinación lineal de las características, la frontera de decisión que crea la regresión logística también será lineal. Esta frontera representa los puntos donde el modelo predice una probabilidad del 50 % (es decir, $p = 0.5$, o $z = 0$). Los puntos de datos a un lado de la línea tendrán probabilidades mayores al 50 %, y los del otro lado, probabilidades menores al 50 %.

En la Figura 4.11, las líneas de contorno que ves representan los umbrales de probabilidad generados por el modelo de regresión logística. Cada línea corresponde a un valor de probabilidad específico. Como la ecuación subyacente usada para calcular estas probabilidades es lineal, las líneas de contorno son paralelas y están espaciadas uniformemente. Estas líneas indican regiones con probabilidades similares y ayudan a visualizar cómo el modelo separa las distintas clases.

> [!warning] Nota de precisión: las curvas de nivel son paralelas, pero no equidistantes
> Las curvas son paralelas porque cada una es el conjunto de puntos con un mismo valor de $z$, y $z$ es lineal. Pero el programa dibuja niveles equiespaciados de **probabilidad** (0.1, 0.2, …, 0.9), y el logit no es lineal en $p$: esos niveles corresponden a $z = -2.197, -1.386, -0.847, -0.405, 0, 0.405, \dots$. La distancia en el plano entre dos curvas consecutivas es $\Delta z / \lVert \boldsymbol{\beta} \rVert$. Con el modelo ajustado, las curvas de 0.4-0.5 y 0.5-0.6 están separadas 0.19 cm, y las de 0.1-0.2 y 0.8-0.9, 0.39 cm: el doble. Las bandas se estrechan cerca de la frontera, donde la sigmoide es más empinada. Solo serían equidistantes si se dibujaran niveles equiespaciados de $z$ (de log-odds).

###### Máquinas de vectores de soporte (_Support Vector Machines_)

Las SVM son algoritmos versátiles de aprendizaje supervisado que pueden usarse tanto para tareas de clasificación como de regresión, y ofrecen soluciones potentes para una variedad de problemas de ML.

En clasificación, las SVM funcionan encontrando el **hiperplano** óptimo que separa las clases en el espacio de características con el **margen** máximo. Un hiperplano es la generalización de una recta a más dimensiones: una recta en 2D, un plano en 3D. Este margen es la distancia entre el hiperplano y los puntos de datos más cercanos de cada clase, conocidos como **vectores de soporte** (_support vectors_). Solo esos puntos determinan la solución: se pueden mover o eliminar los demás sin que el hiperplano cambie. Al maximizar este margen, las SVM buscan lograr una mejor generalización con datos no vistos. Las SVM pueden manejar problemas de clasificación tanto lineales como no lineales usando el **truco del kernel** (_kernel trick_) para transformar el espacio de características a dimensiones más altas, lo que permite trazar una frontera lineal en ese espacio transformado. El «truco» es que nunca se calcula la transformación explícitamente: el algoritmo solo necesita productos internos entre pares de puntos, y el kernel $K(\mathbf{x}_i, \mathbf{x}_j)$ los calcula directamente en el espacio original. Los kernels más usados son el lineal, el polinomial y el de **función de base radial** (RBF, _radial basis function_, $K(\mathbf{x}_i,\mathbf{x}_j)=\exp(-\gamma\lVert\mathbf{x}_i-\mathbf{x}_j\rVert^2)$, que mide similitud por cercanía), cada uno adecuado para distintos tipos de distribuciones de datos.

Para tareas de regresión, las SVM se adaptan en la **regresión de vectores de soporte** (SVR, _support vector regression_). El objetivo principal de la SVR es encontrar una función que se desvíe de los valores objetivo reales en no más de un margen especificado y que, al mismo tiempo, sea lo más plana posible. En otras palabras, la SVR trata de ajustar la mejor recta posible dentro de un margen de tolerancia ($\varepsilon$) para los valores objetivo. Este método usa un concepto similar al de los vectores de soporte de la clasificación con SVM: identifica los puntos de datos clave que definen la recta o curva de regresión óptima e ignora los puntos que caen dentro del margen. El resultado es un modelo robusto y flexible que puede manejar valores atípicos y ruido de forma eficaz. En términos estadísticos, la SVR usa la **pérdida ε-insensible**, $\max(0, |y - \hat{y}| - \varepsilon)$: los errores menores que $\varepsilon$ no cuestan nada y los mayores crecen linealmente (no cuadráticamente, como en mínimos cuadrados), por eso los atípicos pesan menos. Igual que la SVM para clasificación, la SVR también se beneficia de las funciones kernel para capturar relaciones complejas entre las características y la variable objetivo, lo que la convierte en una herramienta potente para el análisis de regresión.

###### Casos de uso (_Use Cases_)

La regresión lineal se usa mejor cuando existe una relación lineal clara entre la variable dependiente y una o más variables independientes. Es ideal para problemas de simples a moderadamente complejos donde el objetivo es predecir un resultado continuo, como predecir el precio de una casa a partir de los metros cuadrados y el número de recámaras. La regresión lineal es fácil de implementar e interpretar, lo que la hace adecuada para escenarios donde la transparencia del modelo es importante. Sin embargo, puede no ser la mejor opción para relaciones más complejas que impliquen no linealidad o interacciones entre variables. En esos casos, usar regresión lineal puede llevar a un rendimiento pobre del modelo y a predicciones inexactas. Además, la regresión lineal es sensible a los valores atípicos y a la **multicolinealidad**, que pueden distorsionar los resultados. Por lo tanto, es menos apropiada para datasets con atípicos significativos, datos de alta dimensión o casos donde no se cumple el supuesto de una relación lineal. Para estos escenarios más complejos, otros algoritmos como la regresión polinomial, los árboles de decisión o el gradient boosting podrían ser más eficaces.

La regresión logística es un método de referencia para problemas de clasificación binaria, donde la variable de resultado es categórica con dos resultados posibles, como la detección de spam, el diagnóstico de enfermedades (positivo/negativo) o la predicción de la **pérdida de clientes** (_churn_: si un cliente cancelará el servicio, sí/no). Es adecuada cuando la relación entre las variables independientes y las log-odds de la variable dependiente es lineal. La regresión logística es fácil de implementar e interpretar, y ofrece no solo la clasificación, sino también la probabilidad de pertenencia a la clase. Sin embargo, es menos eficaz con datasets complejos con relaciones no lineales o interacciones entre variables. En esos casos, su rendimiento podría quedar por detrás de técnicas más avanzadas como los árboles de decisión, las SVM o las redes neuronales, que pueden modelar patrones más intrincados. Además, la regresión logística supone la ausencia de multicolinealidad entre las variables independientes y requiere un dataset suficientemente grande para producir resultados confiables, lo que la hace menos apropiada para datos pequeños, muy correlacionados o de dimensión muy alta.

> [!note] Recuadro del libro
> La **multicolinealidad** se refiere a una situación en el modelado estadístico, específicamente en el análisis de regresión múltiple, en la que dos o más variables predictoras están altamente correlacionadas. Esta alta correlación significa que una variable predictora puede predecirse casi linealmente a partir de las demás. Esto crea redundancia y puede causar problemas para estimar de forma confiable los coeficientes del modelo de regresión. Una forma de abordar la multicolinealidad es con técnicas de **regularización**, que añaden una penalización al modelo de regresión lineal por haber aprendido coeficientes grandes que no generalizan bien a datos no vistos. La regularización se cubrirá en el capítulo 5.

> [!warning] Nota de precisión: multicolinealidad y predicción
> La multicolinealidad no es un supuesto del modelo en el sentido estricto ni sesga las predicciones: infla la **varianza de los coeficientes estimados** (se diagnostica, por ejemplo, con el factor de inflación de la varianza, VIF), lo que vuelve inestable su interpretación. Si el objetivo es solo predecir y la correlación entre predictores se mantiene en los datos nuevos, el efecto sobre la precisión suele ser pequeño. Importa sobre todo cuando se quiere interpretar cada coeficiente, que es justo el argumento de «transparencia» con el que el libro recomienda estos modelos.

Las SVM son modelos potentes de aprendizaje supervisado que se usan principalmente para tareas de clasificación, aunque también pueden aplicarse a problemas de regresión. Son especialmente eficaces con datasets de alta dimensión y en casos donde las clases están bien separadas por un margen claro. Las SVM son ventajosas cuando el número de características supera el número de muestras, porque en esos escenarios son menos propensas al sobreajuste.

> [!note] Recuadro del libro
> En ML, el **sobreajuste** (_overfitting_) ocurre cuando un modelo aprende demasiado bien los datos de entrenamiento, capturando el ruido y los valores atípicos además de los patrones subyacentes. Esto da como resultado un modelo que rinde excepcionalmente bien con los datos de entrenamiento, pero mal con datos no vistos, porque no logra generalizar a ejemplos nuevos.
>
> El sobreajuste puede mitigarse con técnicas como la validación cruzada, la poda de árboles de decisión, los métodos de regularización o la simplificación del modelo, para que capture solo los patrones relevantes sin el ruido. Veremos el sobreajuste y las técnicas para abordarlo en el capítulo 5.

Sin embargo, las SVM no son ideales para datasets grandes debido a su intensidad computacional, que puede llevar a tiempos de entrenamiento más largos y a un mayor uso de memoria. Además, las SVM pueden tener dificultades con datasets ruidosos y clases que se superponen, donde podrían rendir peor que algoritmos más flexibles como los random forests o el gradient boosting. También requieren un ajuste cuidadoso de hiperparámetros como el tipo de kernel y los parámetros de regularización para lograr un rendimiento óptimo. En resumen, las SVM se usan mejor con datasets de tamaño pequeño a mediano (hasta aproximadamente 10 000 muestras), de alta dimensión y con clases claramente separadas, y son menos adecuadas para datasets grandes y ruidosos o para problemas que requieren entrenamiento y predicciones rápidos.

El umbral de unas 10 000 muestras es una regla práctica, no un límite del algoritmo, y tiene una explicación de costo: un SVM con kernel trabaja con la **matriz de kernel**, que tiene un valor por cada par de muestras. Con $n = 10\,000$ son $10^8$ valores, unos 800 MB en doble precisión, y el tiempo de entrenamiento crece aproximadamente entre $n^2$ y $n^3$. Con $n = 100\,000$ la matriz completa ocuparía unos 80 GB, más que la memoria de la mayoría de las máquinas. Los SVM lineales (como los de Linear Learner) no tienen ese problema.

##### K vecinos más cercanos (_K-Nearest Neighbors_)

El algoritmo k-NN es un método de ML supervisado simple pero eficaz, que se usa para problemas de clasificación y regresión. La idea central de k-NN es que es probable que los puntos de datos similares estén cerca unos de otros en el espacio de características. Al hacer una predicción, k-NN identifica los $k$ vecinos más cercanos de un punto de datos dado y usa sus etiquetas para determinar la etiqueta del punto nuevo. En clasificación, el algoritmo asigna la etiqueta más común entre los vecinos más cercanos (votación por mayoría); en regresión, promedia las etiquetas de los vecinos. Es un método **no paramétrico** y **perezoso** (_lazy_): no estima parámetros durante el entrenamiento, sino que guarda los datos y hace todo el trabajo al momento de predecir.

El valor de $k$ se fija antes de que empiece el proceso de aprendizaje y representa el número de puntos de datos más cercanos (los vecinos) que se consideran al hacer una predicción para un punto nuevo. Es un hiperparámetro crucial del algoritmo k-NN. Un valor pequeño de $k$ (p. ej., $k = 1$) hace que el modelo sea muy sensible al ruido de los datos, porque solo considera al vecino más cercano. Por el contrario, un valor grande de $k$ (el valor de ejemplo se perdió en la extracción; piensa en algo como $k = 20$) suaviza las predicciones al promediar sobre más vecinos, lo que puede ser beneficioso con datos ruidosos, pero puede pasar por alto patrones locales. Es el dilema sesgo-varianza en su forma más pura: $k$ pequeño da poco sesgo y mucha varianza; $k$ grande, lo contrario. Elegir el valor correcto de $k$ implica equilibrar estos factores y suele hacerse con técnicas como la **validación cruzada**, que se analizará en el capítulo 5. En esencia, $k$ determina el «vecindario» alrededor de un punto de datos que influye en su predicción.

A pesar de su simplicidad, k-NN puede ser costoso computacionalmente, sobre todo con datasets grandes, porque requiere calcular la distancia entre el punto de consulta y todos los demás puntos del dataset. Con un millón de puntos de entrenamiento y 100 características, cada predicción individual implica del orden de $10^8$ operaciones si se hace por fuerza bruta. Pueden usarse varias métricas de distancia para medir las distancias, como la euclidiana, la de Manhattan y la de Minkowski:

$$d_{\text{euclidiana}} = \sqrt{\sum_{j}(x_j - x'_j)^2} \qquad d_{\text{Manhattan}} = \sum_j |x_j - x'_j| \qquad d_{\text{Minkowski}} = \Big(\sum_j |x_j - x'_j|^p\Big)^{1/p}$$

La de Minkowski generaliza a las otras dos: con $p = 1$ es la de Manhattan y con $p = 2$, la euclidiana. La elección de la métrica de distancia puede afectar significativamente el rendimiento del algoritmo y debe ajustarse a la naturaleza de los datos y del problema.

> [!warning] Nota de precisión: métricas del k-NN de SageMaker (verificado el 24-09-2026)
> El algoritmo integrado k-NN de SageMaker no ofrece Manhattan ni Minkowski. Su hiperparámetro `index_metric` admite `L2` (euclidiana, el valor predeterminado), `INNER_PRODUCT` (producto interno) y `COSINE` (similitud coseno). Internamente construye un índice con la biblioteca **FAISS** (de Meta), que permite búsquedas exactas (`faiss.Flat`) o aproximadas y mucho más rápidas (`faiss.IVFFlat`, `faiss.IVFPQ`). Sus hiperparámetros obligatorios son `feature_dim`, `k`, `predictor_type` (`classifier` o `regressor`) y `sample_size` (cuántos puntos de entrenamiento muestrear para el índice). También puede reducir la dimensión antes de indexar (`dimension_reduction_type`), lo que ataca el costo mencionado arriba.

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

- **Carga de datos.** En este paso cargamos el dataset Iris y seleccionamos las dos primeras características para facilitar la visualización: el largo y el ancho del sépalo.
- **División de datos.** Dividimos el dataset en conjuntos de entrenamiento y de prueba.
- **Entrenamiento del modelo.** Creamos un clasificador k-NN con $k = 3$ y lo ajustamos a los datos de entrenamiento.
- **Graficar la frontera de decisión.** La frontera de decisión de la gráfica es el resultado de la predicción de k-NN con $k = 3$ y se muestra visualmente en la Figura 4.12 mediante las tres regiones del espacio de características bidimensional de nuestro dataset Iris. La región del lado izquierdo de la Figura 4.12 indica predicciones de Iris-setosa, la región de abajo indica predicciones de Iris-versicolor y la región del lado derecho indica predicciones de Iris-virginica.

_Figura 4.12 Ajuste de k-NN al dataset Iris en 2D._

- **Graficar los puntos de entrenamiento y de prueba.** De forma similar, la forma de cada punto de datos corresponde directamente a su etiqueta de clase real (es decir, al valor verdadero de la clase: Iris-setosa, Iris-versicolor o Iris-virginica), lo que permite distinguir visualmente entre las predicciones y las etiquetas de los distintos puntos de la gráfica. Los puntos de prueba se grafican con relleno blanco y contorno negro usando `facecolors='none'` y `edgecolors='k'`.

Observa en la Figura 4.12 cómo algunos puntos de entrenamiento de Iris-virginica están ubicados en la zona inferior del espacio de características (es decir, la zona de Iris-versicolor), lo que indica que nuestro algoritmo k-NN (con $k = 3$) clasificó por error estos puntos de Iris-virginica como Iris-versicolor. Como resultado, este modelo no rindió bien. En el capítulo 5 se dará más información, cuando aprendas a evaluar el rendimiento de un modelo ajustando el valor de sus hiperparámetros, como $k$.

Al ejecutar el programa (scikit-learn 1.5.2), la exactitud es de 0.875 en entrenamiento y de 0.80 en prueba. Las matrices de confusión (filas: especie real; columnas: especie predicha; orden setosa, versicolor, virginica) muestran dónde están los errores:

```text
Entrenamiento (120 flores)      Prueba (30 flores)
[[40  0  0]                     [[10  0  0]
 [ 0 32  9]                      [ 0  7  2]
 [ 0  6 33]]                     [ 0  4  7]]
```

Las 6 virginicas de entrenamiento clasificadas como versicolor son las que menciona el libro. Setosa se separa perfectamente; el problema está entre versicolor y virginica, que con solo las dos medidas del sépalo se superponen mucho. Con las medidas del pétalo el problema casi desaparece, como muestra el ejemplo de XGBoost más adelante.

Para usar el algoritmo k-NN en Amazon SageMaker, primero debes preparar tu dataset y subirlo a una ubicación de almacenamiento adecuada (p. ej., Amazon S3, Amazon EFS o Amazon FSx for Lustre). Luego puedes configurar un **estimador** (_estimator_) con el algoritmo k-NN, especificando parámetros como el número de vecinos $k$ y la métrica de distancia. Un estimador es el objeto del SDK de SageMaker para Python que describe un trabajo de entrenamiento: qué contenedor usar, con qué rol de IAM (la identidad con permisos para leer tus datos en S3), en cuántas instancias y de qué tipo, con qué hiperparámetros y dónde guardar el modelo. Después de configurar el estimador con los hiperparámetros necesarios y las rutas de los datos de entrada, inicias el trabajo de entrenamiento, que Amazon SageMaker gestionará, incluido el aprovisionamiento y el escalado de la infraestructura requerida. **Aprovisionar** significa reservar y poner en marcha los recursos (instancias, discos, red) que el trabajo necesita. Una vez terminado el entrenamiento, el modelo puede desplegarse como un endpoint en Amazon SageMaker para hacer predicciones en tiempo real. Este enfoque integrado aprovecha la infraestructura robusta de Amazon SageMaker para entrenar y desplegar modelos k-NN de forma eficiente.

###### Casos de uso (_Use Cases_)

k-NN es más adecuado para problemas de ML de clasificación y regresión cuando el dataset es relativamente pequeño (menos de 1 000 muestras) a moderado (de 1 000 a aproximadamente 10 000 muestras) y las fronteras de decisión no son demasiado complejas.

Es especialmente eficaz cuando los datos tienen una proximidad clara e intuitiva en el espacio de características que da sentido al concepto de «cercanía». k-NN también debería considerarse cuando el costo de la interpretabilidad del modelo es alto. Esto se debe a que k-NN se considera intrínsecamente un algoritmo muy interpretable, lo que significa que puedes entender fácilmente cómo hace predicciones a partir de la proximidad de los puntos de datos en el espacio de características, lo que lo convierte en una buena opción cuando la explicabilidad es una preocupación principal.

> [!warning] Nota de precisión: k-NN, fronteras complejas e interpretabilidad
>
> - El libro se contradice: aquí dice que k-NN conviene cuando las fronteras «no son demasiado complejas», y en «Puntos esenciales para el examen» dice que es «particularmente útil cuando la frontera de decisión es compleja y no lineal». Lo segundo es lo correcto desde el punto de vista estadístico: al ser no paramétrico, k-NN se adapta a fronteras de cualquier forma, como se ve en las regiones irregulares de la Figura 4.12. Para el examen, quédate con la versión de «Puntos esenciales».
> - Su interpretabilidad es **local**: puedes justificar una predicción mostrando los vecinos («se parece a estos tres casos»), pero no hay coeficientes que resuman el efecto global de cada variable, como en una regresión.
> - Los rangos de «pequeño» y «moderado» son orientativos. Con índices aproximados como FAISS, k-NN escala a millones de puntos; el límite práctico de la versión por fuerza bruta es el costo por predicción descrito arriba.

Sin embargo, k-NN es menos adecuado para datasets grandes por su alto costo computacional y su uso de memoria. Rinde mal con datos de alta dimensión (la «maldición» de la dimensionalidad), porque la métrica de distancia pierde significado, y puede tener dificultades con datos ruidosos a menos que se ajuste con cuidado. La **maldición de la dimensionalidad** tiene una explicación concreta: al crecer el número de dimensiones, las distancias entre puntos al azar se concentran alrededor de un mismo valor, de modo que el vecino «más cercano» está casi tan lejos como el más lejano y la noción de vecindario se diluye. Además, k-NN requiere un preprocesamiento significativo, como el escalado de características, para funcionar eficazmente; sin él, una variable medida en pesos (del orden de miles) dominaría la distancia sobre otra medida en años (del orden de decenas). Si el dataset contiene muchas características irrelevantes o muy correlacionadas, el rendimiento de k-NN puede degradarse, y será necesario considerar algoritmos alternativos como las SVM o los random forests en esos casos.

##### Árboles de decisión (Random Forest y XGBoost) (_Decision Trees (Random Forest and XGBoost)_)

Los algoritmos de árboles de decisión están entre los métodos más populares y más usados en ML, tanto para tareas de clasificación como de regresión. Su naturaleza intuitiva y su capacidad de manejar varios tipos de datos los convierten en una opción de referencia para muchos científicos de datos e ingenieros de ML.

Un **árbol de decisión** es un modelo de decisiones con forma de árbol, formado por nodos, ramas y hojas. El **nodo raíz** es el punto de partida y representa todo el dataset; los **nodos internos** son los puntos donde los datos se dividen según ciertas condiciones, y las **hojas** son los puntos terminales que representan los resultados o predicciones finales. Las **ramas** son los caminos que representan los resultados de las decisiones.

Los árboles de decisión operan particionando recursivamente el dataset en subconjuntos según los valores de las características, lo que da como resultado una estructura de árbol. El objetivo es crear ramas tales que los datos de cada subconjunto (nodo) sean lo más homogéneos posible. La **homogeneidad** significa que cada subconjunto de datos (o nodo) resultante de una división debe contener puntos de datos lo más parecidos posible entre sí. En última instancia, quieres lograr una alta **pureza** en cada nodo, lo que indica que los puntos de datos de cada nodo pertenecen a la misma clase (en tareas de clasificación) o tienen valores similares (en tareas de regresión).

Este proceso implica varios pasos clave:

- **Selección de características.** El algoritmo selecciona la mejor característica para dividir los datos en cada nodo. La selección se basa en métricas como la **impureza de Gini**, la **ganancia de información** (entropía) y la **reducción de varianza**, según el tipo de tarea (clasificación o regresión). La fórmula para calcular la varianza es la siguiente:

  $$\text{Var}(S) = \frac{1}{|S|}\sum_{i \in S} (y_i - \bar{y}_S)^2$$

  donde $S$ es el conjunto de observaciones del nodo y $\bar{y}_S$ su media. En regresión se elige la división que más reduce la varianza ponderada de los nodos hijos, $\text{Var}(S) - \frac{|S_L|}{|S|}\text{Var}(S_L) - \frac{|S_R|}{|S|}\text{Var}(S_R)$. Las fórmulas de clasificación, que el libro menciona sin mostrar, son (con $p_c$ la proporción de la clase $c$ en el nodo):

  $$\text{Gini}(S) = 1 - \sum_{c} p_c^2 \qquad\qquad H(S) = -\sum_c p_c \log_2 p_c$$

  Ambas valen 0 en un nodo puro. La ganancia de información de una división es la entropía del padre menos la entropía ponderada de los hijos. (Las fórmulas se perdieron en el `.txt`; estas son las definiciones estándar.)
- **División de los datos.** El dataset se divide en subconjuntos según la característica seleccionada. Cada subconjunto se convierte entonces en un nodo nuevo del árbol, y el proceso se repite recursivamente.
- **Criterios de parada.** La división recursiva se detiene cuando se cumple una condición predefinida. Puede ser una profundidad máxima del árbol, un número mínimo de muestras por nodo o que ya no sea posible mejorar la pureza.
- **Poda.** Para prevenir el sobreajuste, los árboles de decisión pueden podarse. La **poda** (_pruning_) consiste en eliminar ramas poco significativas que no contribuyen al rendimiento del modelo. Puede hacerse con técnicas como la **poda por costo-complejidad** (_cost-complexity pruning_), que elige el subárbol que minimiza $R(T) + \alpha|T|$: el error del árbol más una penalización $\alpha$ por cada hoja, en la misma lógica que un criterio de información penaliza parámetros.

Aunque los árboles de decisión son interpretables y versátiles en su capacidad de captar relaciones complejas y no lineales en los datos, también tienen la desventaja de ser muy sensibles al dataset de entrenamiento. Como resultado, el algoritmo podría generar predicciones completamente distintas ante el cambio de un solo punto de datos. En términos estadísticos, son estimadores de **alta varianza**: un cambio pequeño en los datos puede alterar la primera división, y con ella todo el árbol que cuelga debajo. Para abordar esta limitación se usan **métodos de ensamble** (_ensemble methods_), que combinan muchos modelos, como el random forest y el gradient boosting. Los random forests combinan múltiples árboles de decisión, cada uno entrenado en paralelo con distintos subconjuntos de los datos, para mejorar la precisión y la robustez, mientras que el gradient boosting construye los árboles de forma secuencial, y cada árbol corrige los errores de los anteriores.

###### Random Forest

Con los random forests, se eligen al azar varios puntos de datos del dataset de entrenamiento para crear un dataset nuevo. Para cada punto de datos, se seleccionan unas cuantas características para construir el árbol de decisión del dataset nuevo. A este proceso se le llama **bootstrapping**. El algoritmo random forest implica usar varios árboles de decisión (de ahí el nombre, pues un bosque está formado por muchos árboles), y cada árbol se entrena con un subconjunto aleatorio de los datos, contribuyendo a una predicción agregada, más precisa y robusta, votando por la clase más popular en tareas de clasificación o promediando las predicciones en tareas de regresión.

> [!warning] Nota de precisión: qué se muestrea en un random forest
> Hay dos fuentes de azar, y el libro las mezcla:
>
> 1. **Bootstrap de filas.** Cada árbol se entrena con una muestra de $n$ observaciones tomada **con reemplazo** del dataset original (el mismo bootstrap de la estadística). Solo esto se llama _bootstrapping_; combinado con el promedio de los árboles se llama _bagging_ (_bootstrap aggregating_).
> 2. **Submuestreo de características.** En **cada división** de cada árbol (no «para cada punto de datos»), solo se considera un subconjunto aleatorio de las características, típicamente $\sqrt{d}$ en clasificación. Esto descorrelaciona los árboles, y es lo que distingue al random forest del bagging simple.
>
> Como los árboles son estimadores con poco sesgo y mucha varianza, promediarlos reduce la varianza; y cuanto menos correlacionados estén, más la reduce.

Los algoritmos de random forest tienen la ventaja de entrenar múltiples árboles de decisión en paralelo, lo que da tiempos de entrenamiento significativamente más rápidos. Sin embargo, como cada árbol de decisión se genera de forma independiente, el modelo puede no optimizar completamente los errores de los árboles individuales, y perderse los beneficios del aprendizaje secuencial y de los ajustes basados en gradientes que ofrecen algoritmos más avanzados como XGBoost. «En paralelo» significa que, como los árboles no dependen unos de otros, se pueden repartir entre los núcleos de la CPU o entre máquinas; con 8 núcleos, 500 árboles se entrenan en unos 1/8 del tiempo que tomaría uno tras otro. En boosting eso no es posible entre árboles, porque cada uno necesita los errores del anterior.

###### XGBoost

El algoritmo **XGBoost** (_extreme gradient boosting_), conocido por su eficiencia y su rendimiento, es un algoritmo de ML potente basado en árboles. El objetivo principal de entrenar un modelo XGBoost es construir un ensamble de árboles de decisión que mejoren secuencialmente la precisión de las predicciones corrigiendo los errores de los árboles anteriores. En concreto, cada árbol nuevo se ajusta al **gradiente de la función de pérdida** respecto de las predicciones actuales; con pérdida cuadrática, ese gradiente es simplemente el residuo, así que cada árbol aprende a predecir lo que los anteriores dejaron sin explicar, y su aporte se suma multiplicado por una tasa de aprendizaje pequeña. Un resultado clave de este proceso de entrenamiento es la determinación de las características más importantes del dataset. La **importancia de las características** (_feature importance_) es crucial porque ayuda a entender qué atributos de los datos aportan más al poder predictivo del modelo. Este entendimiento puede orientar la selección de características y el ajuste del modelo, además de dar conocimiento sobre los patrones subyacentes de los datos.

Al comparar XGBoost con random forest, ambos algoritmos son métodos de ensamble que utilizan árboles de decisión, pero difieren significativamente en su enfoque y en su rendimiento. Como se ilustra en la Figura 4.13, random forest construye múltiples árboles de decisión independientes en paralelo y agrega sus predicciones, lo que ayuda a reducir la varianza y a mejorar la robustez. Es especialmente eficaz para manejar datasets grandes con alta varianza, pero no siempre capta los patrones complejos de los datos tan eficazmente como los métodos de gradient boosting.

_Figura 4.13 Comparación entre random forest y XGBoost._

Por otro lado, XGBoost construye árboles de decisión de forma secuencial, y cada árbol busca corregir los errores de los anteriores. Este proceso secuencial de boosting ayuda a XGBoost a reducir tanto el sesgo como la varianza, lo que da modelos más precisos, sobre todo con datasets complejos. Además, las técnicas avanzadas de optimización y de regularización de XGBoost lo hacen más eficiente y menos propenso al sobreajuste en comparación con random forest. La regularización de XGBoost está incorporada en su función objetivo: penaliza el número de hojas de cada árbol ($\gamma$) y la magnitud de los valores de las hojas ($\lambda$, una penalización L2 como la de ridge).

> [!warning] Nota de precisión: ¿XGBoost sobreajusta menos que random forest?
> No en general. El boosting reduce sobre todo el **sesgo** y, si se añaden demasiados árboles o se usa una tasa de aprendizaje alta, **sí** sobreajusta; por eso se controla con la tasa de aprendizaje (`eta`), la profundidad, el submuestreo y la parada temprana. Random forest reduce sobre todo la **varianza**, y añadirle más árboles no lo hace sobreajustar más. Lo que suele observarse en la práctica es que un XGBoost **bien ajustado** alcanza mejor precisión que un random forest en datos tabulares, a cambio de requerir más ajuste de hiperparámetros, que es lo que el propio libro dice en los casos de uso.

El código de Python que se presenta a continuación demuestra cómo entrenar un modelo XGBoost con el dataset Iris. Muestra los pasos para preparar los datos, entrenar el modelo y evaluar la importancia de las características. Al graficar la importancia de las características, el código revela cuáles son las más influyentes en las predicciones.

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

Algunos elementos del código merecen explicación. `DMatrix` es la estructura de datos interna de XGBoost, optimizada en memoria. `max_depth=3` limita cada árbol a tres niveles de divisiones, `eta=0.1` es la tasa de aprendizaje (cada árbol aporta solo el 10 % de su corrección) y `num_round=50` es el número de rondas de boosting. Con `multi:softprob`, el modelo devuelve para cada punto un vector con la probabilidad de cada una de las 3 clases, y `np.argmax` elige la mayor. Como el dataset se pasa como arreglo de NumPy sin nombres de columna, XGBoost llama a las características `f0`, `f1`, `f2` y `f3`; por eso el código añade una leyenda que las traduce.

En la Figura 4.14, el largo del pétalo (`f2`) aparece como una característica significativa, lo que indica su fuerte correlación con las clases objetivo del dataset.

_Figura 4.14 Importancia de las características con XGBoost._

Salida real de la gráfica de importancia (xgboost 3.0.5). `plot_importance` usa por defecto el tipo de importancia `weight`, que cuenta **cuántas veces** se usa cada característica para dividir en todos los árboles:

```text
f2 (petal length):  241
f3 (petal width):   110
f1 (sepal width):    46
f0 (sepal length):   42
```

Con la importancia por **ganancia** (`importance_type='gain'`, la mejora promedio de la pérdida que aporta cada división), el orden es el mismo: `f2` 4.21, `f3` 2.38, `f1` 0.36, `f0` 0.21. Las dos medidas del pétalo dominan, lo que explica por qué el k-NN anterior, que solo usaba las del sépalo, tenía dificultades. Advertencia estadística: la importancia `weight` favorece a las variables continuas con muchos valores posibles de corte, y ninguna de estas medidas indica la dirección del efecto ni equivale a significancia; para eso existen métodos como la importancia por permutación o los valores SHAP.

Después de identificar las características más importantes, el modelo XGBoost puede usarse para hacer predicciones con datos nuevos, no vistos. La Figura 4.15 muestra las predicciones para dos puntos de datos, cada uno formado por cuatro valores numéricos expresados en «cm» (centímetros) correspondientes a las características `f0`, `f1`, `f2` y `f3`.

_Figura 4.15 Predicciones de XGBoost para dos puntos de datos._

Salida real (xgboost 3.0.5):

```text
Prediction for datapoint 1 ([5.1 3.5 1.4 0.2]): Iris-setosa
Prediction for datapoint 2 ([6.2 3.4 5.4 2.3]): Iris-virginica
```

Las probabilidades detrás de esas predicciones son 0.985 para setosa en el primer punto y 0.974 para virginica en el segundo. El modelo también acierta las 30 flores del conjunto de prueba (exactitud de 1.0), aunque con 30 observaciones esa cifra tiene mucha incertidumbre. Un detalle: el primer punto de «datos nuevos», `[5.1, 3.5, 1.4, 0.2]`, es en realidad la primera flor del dataset Iris.

El código también ilustra cómo preparar puntos de datos nuevos, ejecutar predicciones con el modelo entrenado e interpretar los resultados. Este proceso asegura que el modelo no solo sea robusto en términos de su rendimiento en entrenamiento, sino también eficaz en aplicaciones del mundo real, donde continuamente llegan datos nuevos. Al aprovechar el conocimiento obtenido de la importancia de las características y las capacidades predictivas de XGBoost, los ingenieros de ML pueden desarrollar modelos de ML muy precisos y confiables para diversos problemas de datos estructurados.

Para usar el algoritmo XGBoost en Amazon SageMaker, primero debes preparar tu dataset y subirlo a una ubicación de almacenamiento adecuada (p. ej., Amazon S3, Amazon EFS o Amazon FSx for Lustre). Luego configuras un estimador de XGBoost especificando hiperparámetros esenciales como la función objetivo (p. ej., `binary:logistic` para clasificación binaria o `reg:squarederror` para regresión), el número de rondas de boosting y la tasa de aprendizaje. Otros hiperparámetros importantes son `max_depth`, para controlar la profundidad de cada árbol; `subsample`, para especificar la fracción de muestras usada para entrenar cada árbol, y `colsample_bytree`, para determinar la fracción de características usadas al construir cada árbol. Estos dos últimos introducen en el boosting el mismo tipo de azar que usa el random forest, y ayudan a reducir el sobreajuste. En el XGBoost de SageMaker, el número de rondas se llama `num_round` y es obligatorio. Hay una lista detallada de hiperparámetros en https://docs.aws.amazon.com/sagemaker/latest/dg/xgboost_hyperparameters.html.

Después de configurar el estimador con estos hiperparámetros, inicias el trabajo de entrenamiento, y Amazon SageMaker gestiona la infraestructura necesaria para el proceso de entrenamiento. Una vez terminado el entrenamiento, el modelo XGBoost entrenado puede desplegarse como un endpoint en Amazon SageMaker, lo que te permite hacer predicciones en tiempo real con datos nuevos, no vistos.

###### Casos de uso (_Use Cases_)

Los árboles de decisión son una opción adecuada cuando la interpretabilidad es un aspecto crítico de tu problema de ML. Son fáciles de entender y de visualizar, lo que los hace ideales para escenarios donde es esencial explicar las decisiones del modelo a las partes interesadas (_stakeholders_: las personas afectadas por el modelo o que deciden sobre él, como la dirección, un regulador o el área de negocio). Los árboles de decisión funcionan bien tanto con datos numéricos como categóricos y no requieren un preprocesamiento extenso de los datos; por ejemplo, no necesitan escalar las variables, porque una división del tipo «ingreso > 50 000» no cambia si el ingreso se mide en pesos o en miles de pesos. Sin embargo, son propensos al sobreajuste, sobre todo con datasets complejos, lo que lleva a una generalización pobre con datos nuevos. Por lo tanto, son menos adecuados para tareas que involucran datasets grandes y ruidosos o que requieren una alta precisión predictiva.

Los random forests deberían considerarse cuando el objetivo es mejorar el rendimiento predictivo mitigando el problema de sobreajuste asociado a los árboles de decisión individuales. Al promediar los resultados de múltiples árboles, los random forests dan predicciones más robustas y precisas. Son especialmente eficaces con datasets grandes de alta varianza y cuando se necesita conocer la importancia de las características. Sin embargo, los random forests pueden volverse computacionalmente intensivos con un gran número de árboles y modelos profundos, lo que los hace menos ideales para aplicaciones que requieren predicciones en tiempo real o para entornos con recursos computacionales limitados.

XGBoost es el algoritmo preferido cuando la alta precisión predictiva y la eficiencia son primordiales. Destaca en el manejo de datasets grandes e interacciones complejas entre características, gracias a su proceso secuencial de construcción de árboles y a sus técnicas avanzadas de regularización (la regularización es una técnica para abordar el sobreajuste y se cubrirá en detalle en el capítulo 5). XGBoost es especialmente útil en competencias de ML y en escenarios que requieren el mejor rendimiento posible del modelo. Sin embargo, requiere un ajuste cuidadoso de hiperparámetros y puede ser más complejo de implementar que los árboles de decisión y los random forests. Además, la complejidad y las exigencias computacionales de XGBoost podrían ser excesivas para tareas más sencillas donde la interpretabilidad y el despliegue rápido son más críticos que lograr la máxima precisión.

#### Recomendación (_Recommendation_)

Los sistemas de recomendación son fundamentales para ofrecer experiencias personalizadas en diversas industrias. Un enfoque eficaz dentro de Amazon SageMaker es usar el algoritmo de **máquinas de factorización** (_factorization machines_), que destaca en el manejo de los datos dispersos de alta dimensión comunes en las tareas de recomendación. Al aprender **factores latentes** que representan las interacciones usuario-ítem, las máquinas de factorización pueden predecir con precisión las preferencias de los usuarios. Un factor latente es una variable no observada que el modelo infiere, parecida a los factores de un análisis factorial: un usuario y una película pueden quedar descritos por, digamos, 64 números que nadie etiquetó, pero que acaban capturando cosas como «gusto por la acción» o «preferencia por películas recientes». Un aspecto clave de estos modelos son los embeddings, que son representaciones continuas de las características que capturan las relaciones semánticas dentro de los datos, lo que permite un procesamiento eficiente y un mejor rendimiento. El algoritmo **Object2Vec** de Amazon SageMaker potencia aún más esta capacidad al aprender embeddings que pueden representar tipos diversos de objetos, lo que facilita sistemas de recomendación sofisticados, capaces de entender y predecir con eficacia comportamientos y preferencias complejos de los usuarios.

##### Máquinas de factorización (_Factorization Machines_)

El algoritmo **Factorization Machines** de Amazon SageMaker es una herramienta potente diseñada para manejar datasets dispersos de alta dimensión, que son comunes en los sistemas de recomendación y en las tareas de predicción de clics. Este algoritmo es especialmente eficaz para captar interacciones entre características que los modelos lineales tradicionales podrían pasar por alto. Las máquinas de factorización extienden las capacidades de la **factorización de matrices**, permitiendo una gama más amplia de interacciones entre variables. Al descomponer las relaciones complejas de los datos en componentes más simples y manejables, el algoritmo puede aprender eficientemente de grandes cantidades de datos y dar predicciones precisas. La implementación de Amazon SageMaker admite tareas tanto de clasificación como de regresión, lo que la hace versátil para diversas aplicaciones.

Para un estadístico, la manera más clara de verla es como una regresión con **todos los términos de interacción de segundo orden**, pero con los coeficientes de interacción restringidos a un producto interno de vectores latentes:

$$\hat{y}(\mathbf{x}) = w_0 + \sum_{i=1}^{d} w_i x_i + \sum_{i=1}^{d}\sum_{j=i+1}^{d} \langle \mathbf{v}_i, \mathbf{v}_j \rangle\, x_i x_j$$

Con $d = 1\,000\,000$ características, una regresión con todas las interacciones tendría del orden de $5 \times 10^{11}$ coeficientes, y la mayoría de los pares de características nunca aparecen juntos en los datos, así que no se podrían estimar. Con vectores latentes $\mathbf{v}_i$ de dimensión $k$ (el hiperparámetro `num_factors`), solo hay $d \times k$ parámetros, y la interacción entre dos características que nunca coincidieron se estima igual, a través de lo que cada una comparte con las demás. La factorización de matrices clásica de los sistemas de recomendación es el caso particular en el que las únicas características son el identificador del usuario y el del ítem.

Una de las ventajas clave del algoritmo Factorization Machines de Amazon SageMaker es su capacidad de manejar la dispersión de forma eficiente. Los datasets dispersos, donde muchas características valen cero o faltan, pueden plantear desafíos significativos para muchos algoritmos de ML. Sin embargo, las máquinas de factorización aprovechan los factores latentes para modelar estas interacciones de forma compacta, reduciendo la dimensionalidad del problema y mejorando el rendimiento. Esto hace al algoritmo especialmente adecuado para tareas como las recomendaciones usuario-ítem, donde la matriz de datos puede ser muy grande pero también muy dispersa. Por ejemplo, si se codifican con one-hot un millón de usuarios y 100 000 productos, cada fila tiene 1 100 000 columnas y solo dos unos.

Para usar el algoritmo Factorization Machines en Amazon SageMaker, primero debes preparar tu dataset y subirlo a una ubicación de almacenamiento adecuada (p. ej., Amazon S3, Amazon EFS o Amazon FSx for Lustre). Luego configuras un estimador para el algoritmo de máquinas de factorización especificando los hiperparámetros esenciales. Estos incluyen `predictor_type` (`binary_classifier` o `regressor`), el número de factores para captar las interacciones entre características, `feature_dim` (número total de características), `mini_batch_size` y `epochs` para la duración del entrenamiento. Un **mini-batch** (minilote) es el número de observaciones que el algoritmo procesa antes de actualizar los parámetros; en lugar de calcular el gradiente con todo el dataset, lo estima con muestras pequeñas, lo que permite entrenar con datos que no caben en memoria. Además, puedes fijar parámetros de regularización como `bias_lr` (tasa de aprendizaje del término de sesgo), `linear_lr` (tasa de aprendizaje del término lineal) y `factor_lr` (tasa de aprendizaje del término de factores) para controlar el sobreajuste. Para saber más sobre los hiperparámetros de las máquinas de factorización, visita https://docs.aws.amazon.com/sagemaker/latest/dg/fact-machines-hyperparameters.html.

> [!warning] Nota de precisión: hiperparámetros de Factorization Machines (verificado el 24-09-2026)
>
> - Los obligatorios son `feature_dim`, `num_factors` y `predictor_type`. La documentación sugiere `num_factors = 64` como punto de partida.
> - `bias_lr`, `linear_lr` y `factors_lr` (con «s»; no `factor_lr`) son **tasas de aprendizaje**, no parámetros de regularización. La regularización se controla con `bias_wd`, `linear_wd` y `factors_wd` (_weight decay_, una penalización L2).
> - El «sesgo» (_bias_) de `bias_lr` es el intercepto $w_0$ de la fórmula de arriba, no el sesgo estadístico de un estimador.
> - Solo admite clasificación **binaria** y regresión; no multiclase.

Una vez configurado el estimador con estos hiperparámetros, inicia el trabajo de entrenamiento, y Amazon SageMaker gestionará la infraestructura necesaria. Después del entrenamiento, despliega el modelo entrenado como un endpoint en Amazon SageMaker para hacer predicciones en tiempo real con datos nuevos.

###### Casos de uso (_Use Cases_)

Los casos de uso principales de las máquinas de factorización de Amazon SageMaker incluyen los sistemas de recomendación y el modelado predictivo en el comercio electrónico. Por ejemplo, pueden usarse para predecir las preferencias de los usuarios en tiendas en línea, analizando el comportamiento de compra pasado y las interacciones con los productos. Además, las máquinas de factorización son muy eficaces para predecir la **tasa de clics** (CTR, _click-through rate_: la proporción de veces que un anuncio mostrado recibe un clic) en la publicidad en línea, donde entender las interacciones de los usuarios con los anuncios puede influir significativamente en la segmentación y la personalización de los anuncios. Es un problema de clasificación binaria («¿hará clic?») con características muy dispersas (usuario, anuncio, sitio, hora, dispositivo) cuyas interacciones importan: un anuncio de juguetes funciona en un sitio de crianza, no en uno de finanzas. En general, la versatilidad y la eficiencia del algoritmo de máquinas de factorización lo convierten en una herramienta valiosa para cualquier escenario que involucre datasets dispersos a gran escala.

##### Object2Vec

**Object2Vec** de Amazon SageMaker es un algoritmo versátil de embeddings neuronales diseñado para generar embeddings densos de baja dimensión a partir de datos de alta dimensión. «Denso» es lo opuesto a «disperso»: un vector de 100 números casi todos distintos de cero, en lugar de un one-hot de un millón de posiciones con un solo uno. Estos embeddings capturan las relaciones semánticas entre objetos, lo que los hace útiles para tareas como la búsqueda de vecinos más cercanos, el clustering y la representación de características en tareas posteriores de aprendizaje supervisado. Object2Vec generaliza la conocida técnica **Word2Vec**, optimizada para varios tipos de datos estructurados y no estructurados. Word2Vec aprende un vector por palabra a partir de las palabras que la rodean en un corpus; Object2Vec aplica la misma idea a pares de objetos cualesquiera (usuario-película, pregunta-respuesta, oración-oración), aprendiendo de ejemplos de pares relacionados. Según la documentación, por eso es formalmente un algoritmo supervisado: necesita pares etiquetados con su relación, aunque esas etiquetas muchas veces salen de los propios datos (por ejemplo, «este usuario vio esta película») sin anotación humana.

Para configurar los hiperparámetros de Object2Vec en Amazon SageMaker, debes especificar varios parámetros clave. Entre ellos están `enc0_max_seq_len` (la longitud máxima de secuencia del codificador enc0), `enc0_vocab_size` (el tamaño del vocabulario de tokens de enc0), `dropout` (la probabilidad de _dropout_ de las capas de la red), `early_stopping_patience` (el número de épocas consecutivas sin mejora antes de la parada temprana) y `enc_dim` (la dimensión de la capa de embedding de salida). Además, puedes personalizar la lista de comparadores (`comparator_list`) para definir cómo se comparan los embeddings, y fijar la tasa de aprendizaje (`learning_rate`), el tamaño del mini-batch (`mini_batch_size`) y el tipo de optimizador (`optimizer`). Estos hiperparámetros te permiten ajustar el algoritmo a tu caso de uso específico, asegurando un rendimiento y una precisión óptimos. Para saber más sobre los hiperparámetros de Object2Vec, visita https://docs.aws.amazon.com/sagemaker/latest/dg/object2vec-hyperparameters.html.

Algunos de esos términos, en breve:

- Object2Vec tiene dos **codificadores** (_encoders_), enc0 y enc1, uno para cada objeto del par. Un codificador es la parte de la red que convierte una entrada (una secuencia de tokens) en un vector.
- El **dropout** apaga al azar una fracción de las neuronas en cada paso de entrenamiento, lo que obliga a la red a no depender de ninguna en particular; es una forma de regularización.
- La **parada temprana** (_early stopping_) detiene el entrenamiento cuando la métrica de validación deja de mejorar durante cierto número de épocas (la «paciencia»), para no sobreajustar.
- El **comparador** combina los dos embeddings del par (por ejemplo, con su diferencia o su producto elemento a elemento) antes de predecir la relación.
- El **optimizador** es el método de descenso de gradiente que actualiza los pesos: SGD, Adam, etc.

###### Casos de uso (_Use Cases_)

Object2Vec en Amazon SageMaker es ideal para escenarios donde necesitas generar embeddings de alta calidad a partir de datos complejos de alta dimensión para capturar relaciones semánticas. Es especialmente útil en sistemas de recomendación, búsquedas de similitud y clustering, y como características de entrada para tareas posteriores de aprendizaje supervisado. Sin embargo, Object2Vec puede no ser la mejor opción si tu objetivo principal es realizar transformaciones lineales simples o si tu dataset no es disperso y no se beneficia de captar interacciones intrincadas. Además, para aplicaciones que requieren entrenamiento en tiempo real o datasets extremadamente grandes con restricciones estrictas de latencia, otros algoritmos optimizados para esas tareas podrían ser más adecuados. En general, Object2Vec destaca en situaciones que requieren embeddings robustos e interpretables para datos complejos, estructurados o no estructurados.

> [!warning] Nota de precisión: los embeddings no son interpretables en el sentido habitual
> Las coordenadas de un embedding no tienen significado propio: nadie puede decir qué representa la dimensión 17. Lo interpretable es la **geometría**: qué objetos quedan cerca de cuáles. Si la interpretabilidad de cada variable es un requisito, un embedding no la resuelve.

#### Pronóstico (_Forecasting_)

El pronóstico con Amazon SageMaker ofrece soluciones robustas para predecir valores futuros a partir de datos históricos, lo que permite a las empresas tomar decisiones basadas en datos. Amazon SageMaker ofrece DeepAR como algoritmo integrado para el pronóstico de series de tiempo. Veamos cómo funciona.

##### DeepAR

El algoritmo **DeepAR** de Amazon SageMaker es una herramienta potente diseñada para el pronóstico de series de tiempo. Aprovecha las **redes neuronales recurrentes** (RNN, _recurrent neural networks_) para captar patrones temporales complejos y dependencias dentro de los datos. Una RNN procesa una secuencia paso a paso y mantiene un **estado oculto**, un vector que resume lo visto hasta ese momento y que se actualiza con cada observación nueva; es, en espíritu, una generalización no lineal de un modelo de espacio de estados. A diferencia de los métodos estadísticos tradicionales, DeepAR puede manejar datasets a gran escala con **múltiples series de tiempo**, lo que lo hace especialmente eficaz para tareas de pronóstico que involucran productos o ubicaciones diversos. Al entrenar con series de tiempo relacionadas, el algoritmo puede aprender patrones compartidos y mejorar la precisión de los pronósticos individuales, lo que es especialmente beneficioso para aplicaciones como el pronóstico de demanda, la gestión de inventarios y las predicciones financieras.

La diferencia de fondo con ARIMA o el suavizamiento exponencial es que estos ajustan **un modelo por serie**, mientras que DeepAR ajusta **un solo modelo global** con todas las series a la vez. Con 50 000 productos, una serie con dos meses de historia «toma prestada» la estructura estacional que el modelo aprendió de las demás, igual que un modelo jerárquico o de efectos mixtos comparte información entre grupos. Por eso DeepAR también puede pronosticar productos nuevos sin historia propia (arranque en frío), si se le dan características categóricas que los relacionen con otros (categoría, tienda).

DeepAR está diseñado para dar **pronósticos probabilísticos**. Esto significa que no solo predice una estimación puntual, sino que también cuantifica la incertidumbre generando un rango de valores futuros posibles. Matemáticamente, DeepAR produce una distribución de probabilidad alrededor de un valor futuro predicho, en lugar de solo una estimación puntual, lo que permite a los usuarios entender la incertidumbre asociada a sus predicciones. La capacidad de DeepAR de producir pronósticos probabilísticos es crítica para los procesos de decisión que necesitan considerar distintos resultados y sus probabilidades asociadas. Esto permite a las empresas gestionar mejor los riesgos y optimizar recursos, entendiendo todo el espectro de escenarios futuros posibles. En la práctica, se piden **cuantiles**: el P50 (mediana) como pronóstico central y, por ejemplo, el P90 para dimensionar el inventario de seguridad, de modo que solo en un 10 % de los días la demanda supere lo almacenado.

Al configurar DeepAR desde Amazon SageMaker, hay que considerar varios hiperparámetros clave. El hiperparámetro `epochs` determina el número de veces que el modelo recorrerá los datos de entrenamiento, lo que influye en la duración del entrenamiento y en el rendimiento. El hiperparámetro `context_length` especifica el número de pasos de tiempo del pasado que el modelo usa para hacer los pronósticos. El hiperparámetro `prediction_length` define el número de pasos de tiempo futuros que el modelo predecirá. Además, `num_layers` y `num_cells` controlan, respectivamente, la profundidad y el tamaño de la RNN, lo que afecta la capacidad del modelo de aprender patrones complejos. Otros hiperparámetros importantes son `mini_batch_size`, para la eficiencia del entrenamiento, y `learning_rate`, para ajustar los pesos del modelo durante el entrenamiento. Ajustar adecuadamente estos hiperparámetros es esencial para optimizar el rendimiento y la precisión del modelo en tareas de pronóstico específicas. Para ver la lista completa de hiperparámetros de DeepAR, visita https://docs.aws.amazon.com/sagemaker/latest/dg/deepar_hyperparameters.html.

> [!warning] Nota de precisión: hiperparámetros de DeepAR (verificado el 24-09-2026)
>
> - Los obligatorios son cuatro: `context_length`, `epochs`, `prediction_length` y **`time_freq`**, que el libro no menciona. `time_freq` es la frecuencia de la serie (`M`, `W`, `D`, `H`, `min` o múltiplos como `5min`) y el modelo la usa para elegir las características de calendario y los rezagos.
> - `prediction_length` queda fijo al entrenar: el modelo no puede pronosticar más pasos que ese horizonte.
> - La distribución del pronóstico se elige con `likelihood`: `gaussian`, `beta`, `negative-binomial` (para conteos), `student-T` (el valor predeterminado) o `deterministic-L1`, que da solo un pronóstico puntual.
> - La documentación aclara que `context_length` puede ser mucho menor que la estacionalidad: el modelo añade automáticamente rezagos estacionales (por ejemplo, el de un año en datos diarios).

###### Casos de uso (_Use Cases_)

Usa DeepAR para el pronóstico de series de tiempo cuando trabajes con datasets a gran escala formados por múltiples series de tiempo relacionadas, sobre todo cuando necesites pronósticos probabilísticos que cuantifiquen la incertidumbre de las predicciones. Es especialmente eficaz para aplicaciones como el pronóstico de demanda, la gestión de inventarios y las predicciones financieras, donde es crítico captar patrones y dependencias complejos. Sin embargo, evita usar DeepAR si tu dataset es pequeño o consiste en series de tiempo simples de una sola variable, donde los métodos estadísticos tradicionales (como ARIMA o el suavizamiento exponencial) pueden bastar. Además, si se requiere entrenamiento en tiempo real o predicciones de muy baja latencia, otros algoritmos optimizados para esas tareas podrían ser más apropiados.

> [!warning] Estado del servicio: Amazon Forecast
> Amazon Forecast, el servicio de IA administrado de pronóstico que usaba una variante de este algoritmo (DeepAR+), ya no admite clientes nuevos desde el 29-07-2024. Para un proyecto nuevo en AWS, las opciones son DeepAR en SageMaker, el pronóstico de SageMaker Canvas/AutoML o un modelo propio en un contenedor de framework.

### Algoritmos de ML no supervisado (_Unsupervised ML Algorithms_)

Si alguna vez resolviste crucigramas, conoces la satisfacción de llenar los espacios en blanco y ver cómo las palabras encajan. Ahora imagina una versión más desafiante llamada «crucigrama sin diagrama» (_diagramless crossword_), en la que no solo tienes que resolver las pistas, sino también averiguar la estructura de la cuadrícula. Este intrigante pasatiempo es una metáfora perfecta para entender el ML no supervisado. Igual que en el crucigrama sin diagrama, donde quien lo resuelve debe descubrir la cuadrícula oculta y las palabras, los algoritmos de ML no supervisado trabajan sin etiquetas predefinidas, buscando patrones y estructuras directamente en los datos.

En el ámbito del ML no supervisado hay varios algoritmos clave, cada uno con un propósito distinto. Una técnica popular es el **clustering** (agrupamiento), y K-means es un ejemplo clásico. K-means agrupa los puntos de datos en clusters según sus similitudes, de forma muy parecida a separar las piezas de un rompecabezas en secciones coherentes. Otro método esencial es la **reducción de dimensionalidad**, ejemplificada por el **análisis de componentes principales** (PCA, _principal component analysis_). PCA simplifica datasets complejos reduciendo el número de dimensiones, algo parecido a doblar un mapa grande para enfocarse en un área específica. El **modelado de temas** (_topic modeling_) es otra excelente aplicación del ML no supervisado, con algoritmos como la **asignación latente de Dirichlet** (LDA, _latent Dirichlet allocation_) y el **modelo neuronal de temas** (NTM, _neural topic model_). Estos algoritmos descubren temas ocultos dentro de grandes datasets de texto, parecido a revelar la historia subyacente en un revoltijo de palabras. Además, la **detección de anomalías** es una aplicación crítica del ML no supervisado, usada para identificar valores atípicos o patrones inusuales en los datos. Algoritmos como **Random Cut Forest** (RCF) e **IP Insights** destacan en este ámbito, detectando anomalías que podrían indicar fraude u otras actividades irregulares. En conjunto, estos algoritmos muestran la versatilidad del ML no supervisado, que ofrece herramientas potentes para explorar y entender las estructuras ocultas de tus datos sin necesitar etiquetas ni guía predefinidas.

#### Clustering

En el ML no supervisado, se explora un dataset para descubrir patrones ocultos, agrupar en clusters los puntos de datos similares e identificar estructuras intrínsecas sin etiquetas predefinidas, lo que da conocimiento valioso sobre las características y relaciones subyacentes de los datos. El clustering, en particular, facilita la transición de los datos crudos a la información valiosa al organizar los datos en grupos significativos. Esta organización transforma puntos de datos aislados en información reveladora, que luego puede interpretarse y analizarse para generar conocimiento. Al entender estos clusters, se puede derivar conocimiento accionable que impulse decisiones informadas y la planeación estratégica.

##### Clustering con K-means (_K-Means Clustering_)

El algoritmo K-means es un enfoque popular de ML no supervisado que se usa para agrupar datos en un número predefinido de clusters, $k$.

El algoritmo trabaja de forma iterativa, asignando cada punto de datos a uno de los $k$ clusters según la similitud de los puntos de datos.

La similitud, en este contexto, suele definirse mediante una medida de distancia. La métrica de distancia más común es la **distancia euclidiana**, que calcula la distancia en línea recta entre dos puntos en un espacio multidimensional (nuestro espacio de características). El algoritmo minimiza la suma de las distancias al cuadrado de cada punto al **centroide** del cluster al que está asignado. Un centroide es el punto central de un cluster, calculado como la media de todos los puntos del cluster. Representa el centro de masa del cluster y se usa para asignar puntos de datos a los clusters.

Un desafío común es determinar el número apropiado de clusters para un dataset dado. El **método del codo** (_elbow method_) es una heurística que sirve para estimar el número de clusters graficando la **suma de cuadrados dentro de los clusters** (WCSS, _within-cluster sum of squares_) contra el número de clusters. A medida que aumenta el número de clusters, la WCSS disminuye, pero hay un punto donde la tasa de disminución se frena bruscamente, formando un «codo» en la gráfica. Este punto del codo sugiere un número óptimo de clusters, que equilibra el subajuste y el sobreajuste. Matemáticamente, la WCSS de un conjunto de clusters se calcula como la suma de las distancias al cuadrado entre cada punto de datos y el centroide de su cluster.

Dado un conjunto de clusters $C_1, C_2, \dots, C_k$, con centroides $\boldsymbol{\mu}_1, \boldsymbol{\mu}_2, \dots, \boldsymbol{\mu}_k$, la WCSS se calcula así:

$$\text{WCSS} = \sum_{j=1}^{k} \sum_{\mathbf{x} \in C_j} \lVert \mathbf{x} - \boldsymbol{\mu}_j \rVert^2$$

donde $k$ es el número de clusters, $C_j$ es el conjunto de puntos de datos del $j$-ésimo cluster, $\mathbf{x}$ es un punto de datos del $j$-ésimo cluster, $\boldsymbol{\mu}_j$ es el centroide del $j$-ésimo cluster y $\lVert \mathbf{x} - \boldsymbol{\mu}_j \rVert^2$ es la distancia euclidiana al cuadrado entre el punto $\mathbf{x}$ y el centroide $\boldsymbol{\mu}_j$. (La fórmula se perdió en el `.txt`; esta es la definición estándar.) La WCSS siempre baja al aumentar $k$ (con $k = n$ vale cero), así que no puede minimizarse directamente para elegir $k$; por eso se busca el codo.

###### Visualización del método del codo (_Elbow Method Visualization_)

Para ayudarte a entender el método del codo, el siguiente programa de Python grafica la función del codo para el dataset Iris usando la clase `KMeans` del módulo `sklearn.cluster`:

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

- **Inicialización del modelo.** `KMeans(n_clusters=k, random_state=0)` inicializa el modelo K-means con `k` clusters y un estado aleatorio especificado para la reproducibilidad.
- **Entrenamiento del modelo.** `kmeans.fit(scaled_data)` entrena el modelo K-means con `scaled_data`. Durante este proceso se realizan los siguientes pasos:
  - _Inicialización de los centroides:_ el algoritmo asigna posiciones aleatorias iniciales a los centroides de los clusters.
  - _Asignación de los puntos:_ cada punto de datos se asigna al centroide más cercano según la distancia euclidiana.
  - _Actualización de los centroides:_ los centroides se recalculan como la media de todos los puntos asignados a ese cluster.
  - _Reiteración:_ el proceso de asignación de puntos y actualización de centroides se repite hasta la convergencia (es decir, hasta que los centroides ya no cambian significativamente).
- **Cálculo de la WCSS.** Después de entrenar el modelo, `kmeans.inertia_` se añade a la lista `wcss`. `kmeans.inertia_` es un atributo de la clase `KMeans` (del módulo `sklearn.cluster`) que representa la WCSS. Este atributo mide la compacidad de los clusters: cuanto menor es la WCSS, más cerca están los puntos de un cluster de su centroide.

> [!warning] Nota de precisión: cómo inicializa scikit-learn los centroides
> Desde hace varias versiones, `KMeans` de scikit-learn **no** inicializa al azar por defecto: usa **k-means++** (`init='k-means++'`), que elige el primer centroide al azar y los siguientes con probabilidad proporcional al cuadrado de su distancia a los ya elegidos, de modo que quedan separados. Además, desde la versión 1.4 el número de reinicios `n_init` vale `'auto'`, que con k-means++ significa **un solo intento**; antes eran 10 intentos, de los que se conservaba el de menor WCSS. Como K-means solo garantiza un **óptimo local**, esto cambia resultados, como se verá en el ejemplo de PCA. Para resultados estables, escribe `n_init=10` explícitamente. (El K-means integrado de SageMaker, en cambio, sí usa `init_method='random'` por defecto.)

Al ejecutarse en Amazon SageMaker Studio, este código genera la gráfica del codo y la guarda como `elbow_method_plot.png` en la carpeta `./images` del EFS montado en tu instancia de Amazon SageMaker Studio.

> [!note] Recuadro del libro
> Las instancias de Amazon SageMaker Studio vienen con EFS montado por defecto, así que cualquier archivo guardado en directorios locales de la instancia se almacena en el EFS conectado. Esto asegura que la gráfica generada esté disponible y persista entre distintas sesiones de Amazon SageMaker Studio.
>
> _En los espacios del Studio actual es un volumen EBS por espacio; ver la nota de la sección de Nova Canvas._

La gráfica del codo se muestra en la Figura 4.16, e ilustra eficazmente el número óptimo de clusters al resaltar el punto donde la WCSS empieza a disminuir más lentamente, formando la figura de un «codo». Esta señal visual ayuda a determinar el número apropiado de clusters para el dataset.

_Figura 4.16 Función del codo._

Salida real (scikit-learn 1.5.2). El programa solo imprime la ruta; los valores de la WCSS, que son los que dibuja la gráfica, se muestran debajo:

```text
Elbow method plot has been saved to './images/elbow_method_plot.png'

k     1       2       3       4      5      6      7      8      9      10
WCSS  600.00  222.36  139.82  114.09  90.81  81.50  72.82  65.24  57.31  47.26
```

La WCSS con $k = 1$ vale exactamente 600 por una razón: con datos estandarizados, cada una de las 4 variables tiene varianza 1, y $150 \times 4 = 600$. (En Windows, la ruta se imprime con `\` en lugar de `/`.)

Según los datos de la Figura 4.16, $k = 3$ es el número óptimo de clusters. Esto significa que añadir más clusters más allá de $k = 3$ no da una mejora sustancial en la compacidad de los clusters. En el caso del dataset Iris, este clustering óptimo capta eficazmente la agrupación natural de las distintas especies, equilibrando simplicidad y precisión sin sobreajustar.

> [!warning] Nota de precisión: el codo es ambiguo
> Las caídas de la WCSS son 377.6 (de 1 a 2 clusters), 82.5 (de 2 a 3) y luego alrededor de 25 (de 3 a 4 y de 4 a 5). El quiebre más marcado está en $k = 2$ y hay un segundo quiebre, menor, en $k = 3$. Elegir 3 es razonable, pero en buena parte porque sabemos de antemano que hay tres especies. Con datos reales, el codo suele ser así de difuso, y conviene complementarlo con criterios como el coeficiente de silueta o el estadístico gap, además del conocimiento del dominio. Tampoco hay que leer «equilibra subajuste y sobreajuste» en el sentido supervisado: sin etiquetas, no hay un error de prueba que minimizar.

Pongamos ahora a trabajar el algoritmo K-means y veamos qué tan eficazmente se agrupan nuestros puntos de datos en tres clusters. Ten en cuenta que el dataset Iris es un espacio de características de cuatro dimensiones con 150 puntos de datos. Como resultado, una gráfica que ilustre el clustering no puede representarse visualmente a menos que la dimensionalidad se reduzca de cuatro a tres (o a dos). En la siguiente sección aprenderemos otra técnica de ML no supervisado pensada para reducir la dimensionalidad de un dataset. Para ofrecer una visión completa manteniendo una visualización intuitiva y accesible, vamos a aplicar el algoritmo K-means a una versión simplificada y tridimensional del dataset Iris.

> [!note] Recuadro del libro
> En la próxima sección, cuando veamos el algoritmo PCA, se cubrirá una forma más precisa de agrupar el dataset Iris. PCA está diseñado para reducir la dimensionalidad de tu dataset de características conservando la máxima varianza posible de los datos.

El siguiente programa de Python aplica el clustering K-means a tres características del dataset Iris (largo del sépalo, largo del pétalo, ancho del pétalo) y guarda la gráfica con los clusters y los centroides como `kmeans_iris_3d_plot.png` en la carpeta `./ch04/images` del EFS montado en tu instancia de Amazon SageMaker Studio:

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

En el código, la instrucción `clusters = kmeans.labels_` asigna las etiquetas calculadas a la variable `clusters`, que luego se usa para colorear y agrupar los puntos de datos en la gráfica según su pertenencia a un cluster.

El atributo `kmeans.labels_` devuelve un arreglo de etiquetas enteras que indican a qué cluster pertenece cada punto de datos. Esto ocurre después de ajustar el algoritmo K-means con el dataset de entrenamiento mediante la instrucción `kmeans.fit(scaled_data)`. En esencia, la instrucción `clusters = kmeans.labels_` asigna una etiqueta de cluster a cada punto de datos del dataset, que muestra en qué cluster se agrupó el punto según los resultados del algoritmo de clustering.

> [!note] Recuadro del libro
> No confundas etiquetas con predicciones. Aunque la instrucción `clusters = kmeans.labels_` asigna las etiquetas de cluster a cada punto de datos después de entrenar el algoritmo K-means con la instrucción `kmeans.fit(scaled_data)`, no es en sí un predictor. Es una salida del algoritmo K-means que te dice a qué cluster pertenece cada punto. No predice la pertenencia a un cluster de puntos de datos nuevos, no vistos. Si quieres predecir el cluster de puntos nuevos con un modelo K-means entrenado, debes usar el método `kmeans.predict()` de la clase `KMeans`. Se darán más detalles en el próximo capítulo.

En resumen, el método `kmeans.fit()` entrena el modelo K-means con los datos de entrenamiento, mientras que el atributo `kmeans.labels_` devuelve las etiquetas de cluster de los datos de entrenamiento, y el método `kmeans.predict()` predice las etiquetas de cluster de puntos de datos nuevos, no vistos. Un detalle: los números de cluster son arbitrarios. El «cluster 0» no tiene por qué corresponder a la especie 0; si vuelves a entrenar con otra semilla, los números pueden permutarse.

Ahora que entiendes cómo funciona el código, puedes ver los tres clusters en la Figura 4.17. La gráfica tridimensional del clustering K-means del dataset Iris ilustra eficazmente la agrupación de los puntos de datos según tres características estandarizadas: el largo del sépalo, el largo del pétalo y el ancho del pétalo. La gráfica usa formas distintas para diferenciar los tres clusters, lo que resalta su separación espacial en el espacio de características. Los centroides de cada cluster se muestran de forma prominente con marcadores de estrella amarillos más grandes, lo que los hace fáciles de identificar. Esta visualización ofrece una representación clara e intuitiva de cómo el algoritmo K-means segmentó el dataset Iris en tres grupos distintos, reflejando la agrupación natural de las distintas especies de Iris.

_Figura 4.17 Clustering K-means en 3D del dataset Iris._

Salida real (scikit-learn 1.5.2). Como el dataset trae las especies verdaderas, se puede medir qué tan «natural» es la agrupación con una tabla cruzada de cluster contra especie:

```text
Plot has been saved as './ch04/images/kmeans_iris_3d_plot.png'

            setosa  versicolor  virginica
cluster 0        0           5         38
cluster 1       50           0          0
cluster 2        0          45         12
```

Setosa queda aislada en un cluster propio y sin errores. Versicolor y virginica se reparten bien en su mayoría, pero 17 de las 100 flores de esas dos especies caen en el cluster «equivocado». El **índice de Rand ajustado** (ARI), que vale 1 con una coincidencia perfecta y alrededor de 0 con una asignación al azar, es 0.715. «Refleja la agrupación natural de las especies» es cierto para setosa y aproximado para las otras dos.

Para usar el algoritmo K-means integrado en Amazon SageMaker, puedes seguir estos pasos. Primero, configura una sesión de Amazon SageMaker y especifica el contenedor del algoritmo K-means. Luego, crea un estimador de K-means, especifica el número de clusters y ajusta el modelo con tus datos. Por último, despliega el estimador entrenado en un endpoint para obtener inferencias. El siguiente fragmento muestra un ejemplo breve:

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

Las piezas del fragmento, en términos de infraestructura:

- **`sagemaker.Session()`** guarda la configuración de conexión (región, credenciales, bucket predeterminado).
- **`get_execution_role()`** obtiene el **rol de IAM** con el que corre tu notebook. Un rol de IAM es una identidad con permisos que asumen temporalmente personas o servicios; aquí, SageMaker lo usa para leer tus datos de S3 y escribir el modelo. Esta función solo funciona dentro de un entorno de SageMaker; fuera de él hay que pasar el ARN (el identificador) del rol a mano.
- **`image_uris.retrieve('kmeans', región)`** devuelve la dirección de la imagen de contenedor del algoritmo integrado en esa región.
- **`ml.m4.xlarge`** es una instancia de propósito general de 4 vCPU y 16 GiB de memoria. La familia m4 es de generaciones anteriores; en regiones recientes puede no estar disponible, y la documentación de SageMaker usa hoy tipos como `ml.m5.xlarge`.
- **`deploy()`** crea un endpoint: un servidor encendido las 24 horas que se cobra por hora hasta que lo borras con `predictor.delete_endpoint()`. Es el error de costo más frecuente de quien empieza.

> [!warning] Nota de precisión: el fragmento de K-means no funciona tal cual
> Revisado contra la documentación vigente (24-09-2026); no se ejecutó contra AWS.
>
> 1. **Falta `feature_dim`**, que es obligatorio en el K-means integrado junto con `k`. El trabajo de entrenamiento fallaría. Corrección: `kmeans.set_hyperparameters(k=3, feature_dim=4)`.
> 2. **El formato de los datos.** Si el archivo de S3 es CSV, hay que declararlo (`text/csv`); para datos sin variable objetivo, la documentación indica `text/csv;label_size=0`. Sin tipo declarado, el algoritmo espera RecordIO-protobuf. Corrección: `from sagemaker.inputs import TrainingInput` y `kmeans.fit({'train': TrainingInput('s3://...', content_type='text/csv;label_size=0')})`.
> 3. **El escalador no está ajustado.** `StandardScaler().transform(...)` sobre un escalador nuevo lanza un error. Se comprobó localmente: `NotFittedError: This StandardScaler instance is not fitted yet.` Hay que reutilizar el escalador ajustado con los datos de entrenamiento (`scaler.fit(datos_entrenamiento)` antes de subirlos a S3), no crear uno nuevo.
> 4. **La serialización.** El `Predictor` genérico no sabe convertir una lista de Python en una petición HTTP. Hay que indicarle un formato, por ejemplo con `from sagemaker.serializers import CSVSerializer` y `from sagemaker.deserializers import JSONDeserializer`, y pasar `serializer=CSVSerializer(), deserializer=JSONDeserializer()` a `deploy()`.
> 5. Todo el fragmento requiere la versión 2 del SDK: `pip install "sagemaker<3"`.

###### Casos de uso (_Use Cases_)

K-means es más adecuado para escenarios en los que tienes una idea clara del número de clusters que esperas encontrar en tu dataset. Este algoritmo es muy eficaz para particionar datasets en grupos distintos y sin solapamiento según la similitud de las características. Destaca en escenarios con clusters bien separados y esféricos en el espacio de características. Aplicaciones como la segmentación de clientes, el **análisis de la canasta de compra** (_market basket analysis_), la compresión de imágenes y la detección de anomalías son ideales para K-means, porque el algoritmo maneja eficientemente datasets grandes y escala bien con un número creciente de muestras. Cuando los datos de entrada son principalmente numéricos y están relativamente bien estructurados, K-means puede dar agrupaciones reveladoras y accionables.

> [!warning] Nota de precisión: canasta de compra y compresión de imágenes
> El análisis de la canasta de compra («quienes compran pan también compran leche») se resuelve típicamente con **reglas de asociación** (por ejemplo, el algoritmo Apriori), no con K-means. La compresión de imágenes con K-means se refiere a la **cuantización de color**: se agrupan los colores de los píxeles en $k$ clusters y cada píxel se reemplaza por su centroide, de modo que la imagen se guarda con solo $k$ colores.

Evita usar K-means si tus datos no se ajustan a los supuestos de clusters esféricos y de tamaño similar, o si los clusters que quieres detectar tienen formas y densidades variables. K-means es sensible a los valores atípicos, así que si tus datos contienen anomalías significativas, estas pueden sesgar los resultados y producir un clustering pobre (un atípico lejano arrastra al centroide hacia sí, porque la media no es robusta). Además, K-means requiere un número predefinido de clusters ($k$), que puede ser difícil de determinar sin conocimiento previo de la estructura de los datos. Si tu dataset incluye características no numéricas o si necesitas un método de clustering más flexible que pueda manejar formas y densidades arbitrarias de los clusters, como **DBSCAN** (_density-based spatial clustering of applications with noise_, que agrupa por densidad y marca como ruido los puntos aislados) o el **clustering jerárquico**, K-means puede no ser la mejor opción.

#### Reducción de dimensionalidad (_Dimensionality Reduction_)

Los algoritmos de reducción de dimensionalidad son fundamentales en el ML no supervisado: ayudan a simplificar datasets complejos reduciendo el número de características sin perder la información esencial. Estas técnicas transforman datos de alta dimensión en una forma de menor dimensión, lo que facilita visualizar y analizar tus datos. Entre los algoritmos comunes están PCA, que identifica las direcciones (componentes principales) que maximizan la varianza de los datos, y **t-SNE** (_t-distributed stochastic neighbor embedding_), que se centra en preservar la estructura local y las relaciones entre los puntos de datos. La diferencia práctica: PCA es una proyección lineal que conserva las distancias globales y puede aplicarse a datos nuevos; t-SNE es no lineal, conserva sobre todo quién es vecino de quién y se usa casi solo para visualizar, porque las distancias entre grupos lejanos en su gráfica no significan mucho. Al reducir la dimensionalidad, estos algoritmos ayudan a descubrir patrones subyacentes, facilitan la compresión de datos y mejoran la eficiencia de las tareas de ML posteriores.

Amazon SageMaker ofrece PCA como algoritmo integrado para la reducción de dimensionalidad, lo que permite a ingenieros de ML y científicos de datos reducir eficazmente el número de características de un dataset conservando la mayor variabilidad posible, con lo que mejoran la eficiencia y el rendimiento de los modelos de ML posteriores.

##### Análisis de componentes principales (_Principal Component Analysis_)

PCA es una técnica potente de reducción de dimensionalidad, muy usada en ML y en análisis de datos. Funciona identificando los componentes principales, que son las direcciones de los datos que explican la mayor varianza. Al transformar los datos originales de alta dimensión a un nuevo sistema de coordenadas definido por estos componentes principales, PCA reduce eficazmente el número de características conservando la mayor parte posible de la variabilidad de los datos. Formalmente, los componentes principales son los **vectores propios** de la matriz de covarianza (o de correlación, si los datos se estandarizaron), ordenados por su valor propio, que es la varianza que explica cada uno. Esto facilita visualizar y analizar los datos, y también mejora el rendimiento de los algoritmos de ML posteriores al mitigar problemas relacionados con la multicolinealidad y el sobreajuste. Lo primero ocurre porque los componentes son **ortogonales** (no correlacionados entre sí) por construcción.

Con referencia al ejemplo anterior, apliquemos el algoritmo PCA al dataset Iris para reducir la dimensionalidad de cuatro a tres.

El siguiente programa de Python usa el algoritmo PCA para reducir la dimensionalidad de nuestro dataset y luego aplica el clustering K-means para agrupar los puntos de datos en tres clusters:

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

Ejecuté este programa en Amazon SageMaker Studio; la Figura 4.18 muestra la imagen resultante, que ilustra el clustering K-means en 3D del dataset Iris con dimensiones reducidas por PCA y los centroides resaltados.

_Figura 4.18 Clustering K-means en 3D del dataset Iris usando PCA._

La gráfica muestra tres clusters distintos, cada uno representado con una forma distinta:

- Cluster 1: círculos
- Cluster 2: cuadrados
- Cluster 3: rombos

Los ejes de la gráfica 3D representan los tres componentes principales (PC1, PC2 y PC3) derivados del algoritmo PCA. Estos componentes capturan la máxima varianza del dataset, reduciendo eficazmente su dimensionalidad y preservando su estructura esencial.

Los centroides de cada cluster se resaltan con estrellas y se enumeran como C1, C2 y C3. Estos centroides representan los puntos centrales de cada cluster y dan una referencia de las características típicas de los puntos de datos de cada grupo.

La separación espacial de los clusters en la gráfica 3D demuestra la eficacia de PCA para reducir la dimensionalidad y de K-means para identificar grupos distintos en los datos. La clara distinción entre clusters sugiere que las características de las flores Iris están bien representadas por los componentes principales elegidos.

Salida real (scikit-learn 1.5.2), con dos datos que el programa no imprime: la proporción de varianza explicada por cada componente y la tabla cruzada de cluster contra especie.

```text
3D plot has been saved in './ch04/images' as 'pca_kmeans_iris_plot_3d.png'

Varianza explicada:   PC1 0.7296   PC2 0.2285   PC3 0.0367   (acumulada 0.9948)

            setosa  versicolor  virginica
cluster 0        0          46         50
cluster 1       33           0          0
cluster 2       17           4          0
```

> [!warning] Nota de precisión: con scikit-learn actual, este ejemplo da un mal clustering
>
> - **Qué pasa.** Con scikit-learn 1.5.2, K-means parte en dos a setosa (33 + 17) y junta en un solo cluster a versicolor y virginica. El ARI es 0.43, peor que el 0.715 del ejemplo anterior con tres variables. La WCSS de esta solución es 187.95.
> - **Por qué.** Desde scikit-learn 1.4, `KMeans` hace un solo intento de inicialización (`n_init='auto'`; ver la nota del método del codo) y, con `random_state=42`, cae en un **óptimo local**. Con `KMeans(n_clusters=3, random_state=42, n_init=10)`, el algoritmo encuentra una solución con WCSS de 136.82, setosa queda completa en un cluster (50) y el ARI sube a 0.62. Es muy probable que el libro usara una versión anterior, en la que `n_init=10` era el valor predeterminado.
> - **PCA no hace el clustering «más preciso».** El recuadro anterior promete una forma «más precisa» de agrupar con PCA, pero aquí los tres componentes conservan el 99.5 % de la varianza, así que las distancias entre puntos casi no cambian respecto de los datos originales estandarizados en 4D: K-means sobre las 4 variables estandarizadas da exactamente el mismo resultado (ARI 0.43 con un solo intento). PCA sirve en este ejemplo para **visualizar** los clusters en 3D, no para mejorarlos. Reducir dimensión ayuda al clustering cuando las dimensiones descartadas son sobre todo ruido, algo que no ocurre con Iris.
> - **Los clusters no son las especies.** Ni siquiera la mejor solución (ARI 0.62) separa bien versicolor de virginica. Una buena separación visual en la gráfica no garantiza que los grupos correspondan a categorías reales.

Amazon SageMaker ofrece una implementación de PCA altamente escalable y eficiente, que puede manejar datasets grandes con facilidad. Para usar PCA en Amazon SageMaker, empiezas creando un objeto estimador de PCA, especificando el número de componentes que quieres conservar. Luego ajustas el estimador a tu dataset, lo que implica calcular los componentes principales y transformar los datos originales al nuevo espacio de menor dimensión. El algoritmo PCA de Amazon SageMaker admite formatos de datos tanto dispersos como densos, lo que lo hace versátil para diversos tipos de tareas de preprocesamiento de datos. Según la documentación, ofrece dos modos: `regular`, que calcula los componentes de forma exacta, y `randomized`, que los aproxima con un algoritmo aleatorizado mucho más rápido cuando hay muchas observaciones y muchas características.

Una vez entrenado el modelo PCA, puedes desplegarlo como un endpoint en Amazon SageMaker para transformar datos nuevos al vuelo, o puedes usarlo en transformaciones por lotes para datasets más grandes. Esto permite una integración fluida en tus flujos de trabajo de ML, ya que puedes preprocesar datos en tiempo real o a escala. Además, la integración de Amazon SageMaker con otros servicios de AWS, como Amazon S3 para el almacenamiento de datos y Amazon SageMaker Studio para el desarrollo, ofrece un entorno integral para desarrollar, probar y desplegar modelos PCA. Al aprovechar el algoritmo PCA integrado de Amazon SageMaker, los científicos de datos y los ingenieros de ML pueden reducir eficientemente la dimensionalidad, mejorar el rendimiento de los modelos y agilizar sus pipelines de procesamiento de datos.

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

Qué hace cada bloque, en términos de infraestructura: el programa guarda Iris como CSV en el disco local, lo sube a S3 con boto3 (`default_bucket()` crea o reutiliza un bucket con un nombre del tipo `sagemaker-<región>-<cuenta>`), lanza un trabajo de entrenamiento de PCA y, en lugar de crear un endpoint, crea un **transformador** (_transformer_), que ejecuta una transformación por lotes: enciende instancias, proyecta cada fila del archivo de S3 sobre los componentes y deja el resultado en otro archivo de S3. `split_type='Line'` le indica que cada línea del archivo es un registro, y `strategy='SingleRecord'` que envíe los registros al modelo de uno en uno. `subtract_mean=True` centra los datos, que es lo que hace PCA estándar.

> [!warning] Nota de precisión: el fragmento de PCA no funciona tal cual
> Revisado contra la documentación vigente (24-09-2026); no se ejecutó contra AWS.
>
> 1. **Falta `mini_batch_size`**, que en el PCA integrado es obligatorio junto con `feature_dim` y `num_components`. Corrección: añadir, por ejemplo, `mini_batch_size=150` a `set_hyperparameters`.
> 2. **El canal de entrenamiento no declara CSV.** Corrección: `pca.fit({'train': TrainingInput(s3_data_path, content_type='text/csv;label_size=0')})`, con `from sagemaker.inputs import TrainingInput`.
> 3. **`transform()` no devuelve nada.** En el SDK v2, `Transformer.transform()` devuelve `None` y, por defecto, espera a que termine el trabajo. Por eso `transformed_output.wait()` y `transformed_output.output_path` fallarían con `AttributeError`. Corrección: `pca_transformer.transform(...)` sin asignar, y luego `pca_transformer.output_path`.
> 4. **El nombre del archivo de salida.** Una transformación por lotes nombra la salida como la entrada más `.out`: aquí sería `iris.csv.out`, no `train.csv.out`.
> 5. **El formato de la salida no es CSV.** Según la documentación, PCA responde en JSON (`{"projections": [{"projection": [...]}, ...]}`), JSON Lines o RecordIO-protobuf, así que `pd.read_csv` no produciría una tabla limpia. Además, leer directamente una ruta `s3://` con pandas requiere tener instalado el paquete `s3fs`.
> 6. **El comentario `# Deploy the PCA model` es engañoso:** `transformer()` no despliega un endpoint; prepara una transformación por lotes, que no queda cobrando al terminar.
> 7. Requiere la versión 2 del SDK (`pip install "sagemaker<3"`), y `ml.m4.xlarge` es de una generación anterior (ver la sección de K-means).

Aprenderás más sobre el ajuste de hiperparámetros, el despliegue de modelos y el monitoreo de modelos en los capítulos 5, 6 y 7, respectivamente.

###### Casos de uso (_Use Cases_)

PCA es muy útil en escenarios donde hay que reducir la dimensionalidad de los datos para mejorar la eficiencia del análisis y del modelado. Es especialmente eficaz con datasets de alta dimensión donde la multicolinealidad (alta correlación) entre las características es una preocupación. Al transformar los datos en un nuevo conjunto de componentes ortogonales, PCA ayuda a conservar la varianza más significativa mientras reduce el número de variables. Entre los casos de uso comunes están la compresión de imágenes, donde PCA reduce el número de píxeles necesarios para representar una imagen sin una pérdida significativa de calidad, y el análisis exploratorio de datos, donde ayuda a visualizar datasets complejos proyectándolos en espacios de menor dimensión. Además, PCA se usa para preprocesar datos para modelos de ML, mejorando el rendimiento al reducir el ruido y los riesgos de sobreajuste.

> [!warning] Nota de precisión: PCA y la compresión de imágenes
> PCA no reduce el número de **píxeles**: la imagen reconstruida tiene la misma resolución. Lo que reduce es el número de **coeficientes** necesarios para representarla. Por ejemplo, con un conjunto de fotos de rostros de $100 \times 100$ píxeles (10 000 valores cada una), cada foto puede aproximarse con sus coordenadas en los primeros 100 componentes principales, es decir, 100 números en lugar de 10 000.

Sin embargo, PCA no debe usarse de forma indiscriminada. Una limitación es que supone relaciones lineales entre las variables, lo que podría no captar la complejidad de datos con interacciones no lineales. Tampoco es adecuado para datasets donde la varianza no es una medida apropiada de importancia, porque PCA prioriza las características con mayor varianza y podría descartar varianzas menores pero cruciales. Por ejemplo, en un problema de clasificación, la dirección que separa las clases puede tener poca varianza total y quedar descartada, porque PCA no mira la variable objetivo. Por la misma razón, las variables deben estandarizarse antes: sin escalar, una variable medida en metros dominaría a una medida en kilómetros solo por sus unidades. Además, la interpretabilidad de los componentes principales puede ser difícil, porque son combinaciones de las características originales que pueden no tener un significado claro e intuitivo. Por lo tanto, aunque PCA es una herramienta potente de reducción de dimensionalidad, es esencial considerar la naturaleza de tus datos y los requisitos específicos de tu análisis antes de aplicarlo.

#### Modelado de temas (_Topic Modeling_)

El modelado de temas se considera una técnica de ML no supervisado porque no requiere datos etiquetados para identificar patrones o temas dentro de una colección de documentos. A diferencia del aprendizaje supervisado, donde los modelos se entrenan con pares de entrada-salida para predecir resultados específicos, los algoritmos de modelado de temas como LDA y NTM exploran la estructura inherente de los datos. Buscan descubrir los temas subyacentes que mejor representan la semántica de los documentos, basándose únicamente en los patrones de **coocurrencia de palabras** (qué palabras tienden a aparecer juntas en los mismos documentos), sin conocimiento previo ni anotaciones. Esto hace al modelado de temas muy valioso para explorar y entender grandes datasets de texto no estructurado, lo que permite descubrir temas ocultos y conocimiento sin necesidad de datos de entrenamiento etiquetados de antemano. Ambos algoritmos trabajan con la representación de **bolsa de palabras** (_bag of words_): cada documento se reduce a un vector de conteos de cada palabra del vocabulario, sin importar el orden.

Como analogía, piensa en el modelado de temas como algo parecido al clustering. Igual que los algoritmos de clustering agrupan puntos de datos según su similitud, los algoritmos de modelado de temas agrupan documentos según sus temas subyacentes. Al reconocer patrones y relaciones en el texto, estos algoritmos pueden categorizar documentos en temas distintos, lo que facilita gestionar y entender grandes volúmenes de datos textuales. La analogía tiene un límite importante: el clustering asigna cada punto a **un** grupo, mientras que el modelado de temas asigna a cada documento una **mezcla** de temas (por ejemplo, 70 % economía y 30 % política).

Amazon SageMaker ofrece LDA y NTM listos para usar, lo que permite a los usuarios implementar y escalar sin esfuerzo sus soluciones de modelado de temas sin requerir experiencia previa en aprendizaje profundo.

##### Asignación latente de Dirichlet (_Latent Dirichlet Allocation_)

LDA es un **modelo probabilístico generativo** que se usa para identificar temas en un gran **corpus** (colección de documentos) de texto. «Generativo» significa que postula un proceso aleatorio que habría producido los documentos, y ajustar el modelo es invertir ese proceso. Supone que cada documento es una mezcla de temas y que cada tema es una mezcla de palabras. El nombre «Dirichlet» viene de la **distribución de Dirichlet**, un tipo de distribución de probabilidad esencial en LDA. Se usa una distribución de Dirichlet para modelar la incertidumbre sobre las probabilidades de los distintos temas dentro de un documento. En esencia, ofrece una forma de asignar probabilidades a los distintos temas de un documento, asegurando que la suma de esas probabilidades sea uno. Esta distribución ayuda a determinar la proporción de los distintos temas en cada documento, lo que permite a LDA descubrir la estructura temática latente del texto. La forma de la distribución de Dirichlet se controla con un conjunto de parámetros (llamados **parámetros de concentración**) que pueden ajustarse para influir en cómo se reparte la masa de probabilidad entre los distintos temas. Al actualizar iterativamente las distribuciones de temas sobre los documentos y de palabras sobre los temas, LDA puede identificar temas significativos que representen mejor la semántica del texto.

Para un estadístico, el proceso generativo se escribe así, con $K$ temas: para cada tema $k$, una distribución sobre el vocabulario $\boldsymbol{\phi}_k \sim \text{Dir}(\boldsymbol{\beta})$; para cada documento $d$, unas proporciones de temas $\boldsymbol{\theta}_d \sim \text{Dir}(\boldsymbol{\alpha})$; y para cada palabra del documento, un tema $z \sim \text{Categórica}(\boldsymbol{\theta}_d)$ y luego la palabra $w \sim \text{Categórica}(\boldsymbol{\phi}_z)$. La Dirichlet es la distribución conjugada de la categórica (multinomial), lo que simplifica la inferencia. Con parámetros de concentración pequeños (menores que 1), cada documento tiende a concentrarse en pocos temas; con valores grandes, a mezclar muchos por igual.

En AWS, LDA puede utilizarse de forma eficiente a través de Amazon SageMaker, que ofrece un algoritmo LDA integrado para el modelado de temas. Para usar LDA en Amazon SageMaker, empiezas preparando tus datos de texto y subiéndolos a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). A continuación, creas un estimador de LDA especificando el número de temas y otros hiperparámetros. Amazon SageMaker se encarga del proceso de entrenamiento, aprovechando su infraestructura escalable para manejar datasets grandes y tareas computacionalmente intensivas. Una vez entrenado el modelo, puedes usarlo para transformar documentos nuevos, obteniendo información sobre sus distribuciones de temas. «Preparar los datos de texto» significa, en el caso del LDA integrado, entregarle ya la bolsa de palabras (los conteos por palabra de cada documento), en CSV o RecordIO-protobuf, no el texto crudo. Esta capacidad es especialmente útil para aplicaciones como el clustering de documentos, la recomendación de contenido y la **recuperación de información** (_information retrieval_: encontrar los documentos relevantes para una consulta, como hace un buscador), donde es crucial entender la estructura temática del texto. Al integrar LDA en tu flujo de trabajo de AWS, puedes aprovechar el poder del modelado de temas para mejorar tu análisis de datos y tus soluciones de ML.

###### Casos de uso (_Use Cases_)

LDA es muy eficaz en escenarios donde es crítico entender la estructura temática de un gran corpus de texto. Destaca en aplicaciones como el clustering de documentos, la recomendación de contenido y la recuperación de información. Por ejemplo, en un servicio agregador de noticias, LDA puede ayudar a agrupar artículos por temas como política, deportes o tecnología, mejorando la experiencia de los usuarios al permitir la navegación por temas. Del mismo modo, en la investigación académica, LDA puede usarse para analizar una gran cantidad de artículos de investigación y descubrir los temas predominantes y sus tendencias a lo largo del tiempo. Al identificar la distribución de temas dentro de cada documento, LDA permite a empresas e investigadores extraer conocimiento de datos de texto no estructurados de forma eficiente. Un detalle práctico: LDA no pone nombre a los temas. Devuelve, para cada tema, las palabras más probables («balón, gol, liga, partido…»), y una persona decide llamarlo «deportes».

Sin embargo, LDA tiene algunas limitaciones debido a su supuesto de temas distribuidos según una Dirichlet, que podría no captar dependencias más complejas entre palabras y temas. En esos casos puede entrar en juego NTM. NTM aprovecha las redes neuronales para aprender las representaciones de los temas, lo que da mayor flexibilidad y la capacidad de captar patrones intrincados en los datos. Esto hace a NTM especialmente eficaz para datasets con fuertes dependencias temporales o secuenciales, o donde las relaciones entre palabras y temas no están bien modeladas por una distribución de Dirichlet. Al incorporar NTM en tu flujo de trabajo de AWS, puedes abordar algunas de las deficiencias de LDA y mejorar tu capacidad de analizar y entender datos de texto complejos, asegurando representaciones de temas más precisas y significativas.

> [!warning] Nota de precisión: NTM y las dependencias secuenciales
> El NTM integrado de SageMaker también trabaja con bolsas de palabras, igual que LDA, así que **no ve el orden de las palabras** y no modela dependencias secuenciales dentro del texto. La afirmación del libro sobre «dependencias temporales o secuenciales» no tiene respaldo en la documentación del algoritmo. Si el orden importa, hacen falta modelos de secuencias (por ejemplo, los basados en Transformers). Lo que sí aporta NTM es flexibilidad: un codificador neuronal en lugar de los supuestos de conjugación de LDA.

##### Modelo neuronal de temas (_Neural Topic Model_)

NTM es un enfoque avanzado de modelado de temas que utiliza redes neuronales para aprender representaciones de temas más flexibles y matizadas que los métodos tradicionales como LDA. Al aprovechar el poder del aprendizaje profundo, NTM puede captar patrones y dependencias complejos en los datos textuales, lo que lo hace muy eficaz para extraer temas significativos de grandes volúmenes de texto. A diferencia de LDA, que se apoya en distribuciones de Dirichlet para modelar la probabilidad de los temas dentro de los documentos, NTM usa redes neuronales para aprender automáticamente esas distribuciones, lo que da una mayor adaptabilidad a distintos tipos de datos de texto. Esto da como resultado temas más precisos e interpretables, que pueden ser increíblemente valiosos para aplicaciones como el clustering de documentos, la recomendación de contenido y el análisis de sentimiento. Técnicamente, el NTM de SageMaker es un modelo de **inferencia variacional neuronal**: una red (el codificador) convierte la bolsa de palabras de un documento en una representación latente de temas, y otra (el decodificador) intenta reconstruir las palabras a partir de ella; se entrena como un autoencoder variacional.

NTM puede implementarse sin fricciones con Amazon SageMaker, que ofrece un algoritmo integrado para NTM. Para utilizar NTM en Amazon SageMaker, empiezas preparando tus datos de texto y subiéndolos a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). A continuación, creas un estimador de NTM en Amazon SageMaker especificando los hiperparámetros y las configuraciones de entrenamiento necesarios. La infraestructura escalable de Amazon SageMaker maneja las exigencias computacionales del entrenamiento del algoritmo NTM, lo que lo hace adecuado para datasets grandes. Una vez entrenado el modelo, puede usarse para transformar documentos nuevos y generar distribuciones de temas, lo que da conocimiento valioso sobre la estructura temática de tus datos de texto. Esta capacidad permite a las empresas analizar y entender eficientemente sus datos textuales, lo que les permite tomar decisiones basadas en datos y mejorar sus flujos de trabajo de ML. Al integrar NTM en tu entorno de AWS, puedes aprovechar las fortalezas de las redes neuronales para descubrir conocimiento más profundo de datos textuales complejos.

###### Casos de uso (_Use Cases_)

NTM es especialmente útil en escenarios donde los métodos tradicionales de modelado de temas como LDA se quedan cortos. Por ejemplo, NTM destaca en el manejo de datasets grandes y complejos donde las relaciones entre palabras y temas son intrincadas y no están bien representadas por distribuciones de Dirichlet. Su capacidad de aprovechar el aprendizaje profundo le permite captar representaciones de temas más matizadas y flexibles, lo que lo hace ideal para aplicaciones como el análisis de sentimiento, el análisis de comentarios de clientes y la exploración temática de corpus de texto extensos. Además, NTM es adecuado para tareas que requieren entender dependencias temporales o secuenciales en el texto, dando conocimiento más preciso y significativo sobre la estructura temática subyacente (ver la nota de precisión anterior sobre esta última afirmación).

No obstante, NTM puede no ser la mejor opción para todos los casos de uso. Con datasets más pequeños o en situaciones con recursos computacionales limitados, la complejidad y las exigencias de recursos del algoritmo NTM podrían superar sus beneficios. En esos casos, modelos más simples como LDA pueden ser más eficientes, rentables y suficientes. Además, si los datos de texto están muy estructurados o si los temas ya están bien definidos, la flexibilidad adicional de NTM podría no ofrecer ventajas significativas sobre los métodos tradicionales de modelado de temas. Es esencial considerar la naturaleza de los datos y los requisitos específicos del análisis para determinar el algoritmo más apropiado para la tarea. Si los temas ya están bien definidos y tienes ejemplos etiquetados, el problema deja de ser no supervisado: es una clasificación de texto, y conviene un clasificador (como BlazingText, más adelante).

#### Detección de anomalías (_Anomaly Detection_)

La detección de anomalías es un enfoque de ML no supervisado porque identifica valores atípicos o patrones inusuales en los datos sin requerir ejemplos etiquetados de anomalías. Esto es especialmente útil cuando los datos etiquetados escasean o cuando las anomalías son raras e impredecibles. Piensa en el fraude: los casos confirmados llegan tarde (semanas después, cuando el cliente reclama), son pocos y los defraudadores cambian de táctica, así que un clasificador entrenado con los fraudes del año pasado puede no reconocer los de este.

Amazon SageMaker ofrece Random Cut Forest (RCF) e IP Insights como algoritmos integrados para la detección de anomalías.

- RCF es un potente algoritmo no supervisado de detección de anomalías que funciona creando un bosque de árboles de decisión aleatorios para identificar los puntos de datos que se desvían de la norma.
- IP Insights aprende los patrones de uso de las direcciones IPv4 para detectar actividades sospechosas, como intentos de inicio de sesión inusuales o la creación de recursos desde direcciones IP anómalas.

Ambos algoritmos están diseñados para operar sin datos etiquetados, lo que los hace ideales para la detección de anomalías en tiempo real en diversas aplicaciones.

##### Random Cut Forest

El algoritmo RCF funciona construyendo un bosque de árboles de decisión aleatorios. Cada árbol del bosque se construye con una muestra aleatoria de los datos de entrenamiento, y las anomalías se detectan según la profundidad a la que caen los puntos de datos dentro de estos árboles. Los puntos de datos que alteran significativamente la estructura de los árboles, lo que da como resultado ubicaciones inusualmente profundas, se marcan como anomalías. Este método es muy eficaz para identificar valores atípicos en datasets grandes, como la detección de transacciones fraudulentas o de intrusiones en la red.

> [!warning] Nota de precisión: las anomalías quedan a poca profundidad, no a mucha
> Según la documentación de SageMaker, el puntaje de anomalía de RCF es el cambio esperado en la complejidad del árbol al insertar el punto, y es **aproximadamente inversamente proporcional a la profundidad** a la que queda el punto. Un punto alejado del resto se aísla con pocos cortes aleatorios, así que queda cerca de la raíz (**poco profundo**) y recibe un puntaje alto. Los puntos normales, rodeados de otros, necesitan muchos cortes para quedar aislados y terminan en hojas profundas. El libro lo dice al revés. La idea es la misma que la de _isolation forest_: lo raro se aísla rápido.

Como otros algoritmos de ML, para implementar RCF en Amazon SageMaker empiezas preparando tu dataset y subiéndolo a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). A continuación, creas un estimador de RCF en Amazon SageMaker especificando hiperparámetros como el número de árboles y el número de muestras por árbol. Amazon SageMaker se encarga del proceso de entrenamiento, aprovechando su infraestructura escalable para construir el RCF de forma eficiente. Una vez entrenado el modelo, puedes desplegarlo en un endpoint de Amazon SageMaker para la detección de anomalías en tiempo real, lo que te permite monitorear las anomalías y responder a ellas a medida que ocurren.

Según la documentación, esos dos hiperparámetros tienen una interpretación útil. `num_trees` reduce el ruido del puntaje (es un promedio sobre los árboles) y se recomienda empezar con 100. `num_samples_per_tree` debe elegirse de modo que su inverso aproxime la proporción esperada de anomalías: con 256 muestras por árbol, se espera que alrededor de 1/256, un 0.4 %, de los datos sean anomalías. El algoritmo toma su muestra con **muestreo de reservorio** (_reservoir sampling_), una técnica que obtiene una muestra aleatoria uniforme de un flujo de datos sin conocer de antemano su tamaño, y por eso sirve para datos que llegan continuamente.

###### Casos de uso (_Use Cases_)

RCF es especialmente útil para detectar anomalías en datasets grandes y de alta dimensión donde los métodos tradicionales podrían tener dificultades. Destaca en aplicaciones como la detección de fraude, la seguridad de redes y el mantenimiento predictivo. Por ejemplo, en servicios financieros, RCF puede identificar patrones de transacción inusuales que podrían indicar actividad fraudulenta. En seguridad de redes, puede detectar anomalías en el tráfico que sugieran posibles intrusiones. Del mismo modo, en entornos industriales, RCF puede monitorear los datos de sensores de la maquinaria para detectar señales tempranas de falla (p. ej., MTBF, tiempo medio entre fallas), lo que ayuda a prevenir averías costosas y tiempos de inactividad. Su capacidad de manejar grandes volúmenes de datos y de descubrir patrones sutiles e inesperados lo convierte en una herramienta potente para la detección de anomalías en tiempo real.

> [!warning] Nota de precisión: el MTBF no es una señal de falla
> El **tiempo medio entre fallas** (MTBF, _mean time between failures_) es una métrica de confiabilidad: el promedio de horas de operación entre una falla y la siguiente, calculado con el historial de un equipo o de una flota. No es una señal que RCF detecte en los sensores. Lo que RCF detecta son desviaciones en las lecturas (vibración, temperatura, presión) que anticipan una falla; el **mantenimiento predictivo** usa esas alertas para intervenir antes de que ocurra, y su éxito se refleja en un MTBF más alto.

RCF es menos eficaz cuando las anomalías ya están bien definidas y etiquetadas, porque en esos casos los algoritmos de aprendizaje supervisado pueden dar resultados más precisos. Además, RCF podría no rendir bien con datasets muy pequeños, donde el enfoque de muestreo aleatorio podría pasar por alto patrones importantes. Cuando el dataset es pequeño o hay ejemplos etiquetados de anomalías disponibles, otros métodos como los algoritmos de clasificación supervisada o técnicas estadísticas más simples pueden ser más apropiados. Es esencial considerar la naturaleza de los datos y los requisitos específicos de la aplicación para determinar si RCF es la opción correcta.

##### IP Insights

El algoritmo **IP Insights** es un método de aprendizaje no supervisado que aprende los patrones de uso de las direcciones IPv4 asociándolas con entidades como identificadores de usuario o números de cuenta. Una **dirección IP** es el número que identifica a un dispositivo en internet, como el número de teléfono de una conexión. Una dirección **IPv4** se escribe como cuatro números de 0 a 255 separados por puntos (por ejemplo, `203.0.113.45`), lo que da unos 4 300 millones de direcciones posibles. Las direcciones cercanas numéricamente suelen pertenecer al mismo proveedor de internet o a la misma zona, y el algoritmo aprovecha esa estructura de prefijos. IP Insights captura las asociaciones entre direcciones IP y entidades, determinando qué tan probable es que una entidad use una dirección IP en particular. IP Insights usa redes neuronales para aprender representaciones vectoriales latentes tanto de las entidades como de las direcciones IP, y puede generar embeddings que se pueden usar en tareas de ML posteriores. Cuando se le consulta con un par (entidad, dirección IPv4), IP Insights devuelve un puntaje que indica qué tan anómalo es el patrón, lo que lo hace útil para detectar actividades sospechosas como intentos de inicio de sesión inusuales o la creación de recursos desde direcciones IP anómalas.

Según la documentación, el entrenamiento funciona así: el algoritmo solo ve pares reales observados, así que fabrica **muestras negativas** emparejando al azar entidades con direcciones IP (generadas al azar o tomadas de otros usuarios) y aprende a distinguir los pares reales de los inventados, minimizando la entropía cruzada, como una regresión logística. Los identificadores de entidad se pasan tal cual aparecen en los registros (un nombre de usuario, un ID de cuenta), porque el algoritmo los convierte internamente con una función hash. El puntaje que devuelve (`dot_product`) es el producto interno de los dos embeddings y se interpreta como el **log-odds** de que el par provenga de los datos reales frente al azar: **un puntaje alto significa un par habitual y uno bajo, un par anómalo**. Solo admite direcciones **IPv4**; el formato más nuevo, IPv6, no está soportado.

Para implementar IP Insights en Amazon SageMaker, empiezas preparando tus datos en forma de pares (entidad, dirección IPv4) y subiéndolos a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). En concreto, un CSV de dos columnas: identificador de la entidad y dirección IPv4, por ejemplo `usuario_8812,203.0.113.45`. A continuación, creas un estimador de IP Insights en Amazon SageMaker, especificando hiperparámetros como el número de vectores de entidad y el tamaño de los vectores de embedding (en la documentación, `num_entity_vectors` y `vector_dim`). Amazon SageMaker se encarga del proceso de entrenamiento y, una vez entrenado el modelo, puedes desplegarlo en un endpoint de Amazon SageMaker para predicciones en tiempo real o para procesamiento por lotes. Esto te permite monitorear las anomalías y responder a ellas a medida que ocurren, reforzando tus medidas de seguridad y tu eficiencia operativa. La documentación recomienda instancias con GPU para entrenar e instancias con CPU para la inferencia, y fija un tamaño máximo de modelo de 8 GB.

###### Casos de uso (_Use Cases_)

IP Insights es más adecuado para detectar comportamientos anómalos en escenarios donde es crítico rastrear los patrones de uso de las direcciones IP. Destaca en aplicaciones como la detección de intentos fraudulentos de inicio de sesión, los patrones de acceso inusuales y la identificación de cuentas comprometidas. Por ejemplo, las plataformas de comercio electrónico y los servicios en línea pueden usar IP Insights para monitorear y marcar actividades sospechosas de inicio de sesión que se desvíen del comportamiento normal de los usuarios, reforzando la seguridad y previniendo el acceso no autorizado. La documentación de AWS describe un uso típico: si el puntaje de un inicio de sesión es anómalo, el servidor pide un segundo factor de autenticación (**MFA**, _multi-factor authentication_, como un código enviado al celular) en lugar de bloquear al usuario. Además, puede utilizarse para proteger recursos identificando y mitigando patrones inusuales de creación o uso de recursos desde direcciones IP anómalas, lo que ayuda a mantener la integridad y la seguridad de los sistemas.

Sin embargo, IP Insights puede no ser adecuado para todos los escenarios. Es menos eficaz en entornos donde las direcciones IP cambian con frecuencia o donde las entidades no tienen patrones consistentes de uso de IP, porque el modelo se basa en aprender asociaciones estables entre entidades y direcciones IP. Un ejemplo: los usuarios que se conectan casi siempre por datos móviles, porque las operadoras celulares reasignan direcciones con frecuencia y comparten una misma dirección entre muchos clientes. En esos casos, los sistemas tradicionales basados en reglas u otros métodos de detección de anomalías podrían ser más apropiados. Además, IP Insights podría no rendir bien en contextos donde hay datos etiquetados disponibles y los enfoques de aprendizaje supervisado podrían dar una detección de anomalías más precisa.

### Algoritmos de análisis textual (_Textual Analysis Algorithms_)

El análisis textual se refiere al proceso de usar algoritmos y técnicas de ML para entender, interpretar y obtener información significativa de datos de texto. Esto puede involucrar diversas tareas, como la clasificación de textos, el análisis de sentimiento, el reconocimiento de entidades, el modelado de temas y el resumen. El objetivo del análisis textual es transformar texto no estructurado en conocimiento estructurado que pueda usarse para tomar decisiones, mejorar la experiencia de los usuarios y, en última instancia, obtener inferencias.

Al principio del capítulo ya aprendiste cómo Amazon Comprehend aborda el análisis textual como un servicio de IA administrado. También aprendiste antes cómo los algoritmos LDA y NTM, que son técnicas de ML no supervisado, cumplen un papel significativo en el análisis textual, específicamente en el modelado de temas. Estos algoritmos identifican automáticamente patrones y agrupan palabras relacionadas en temas dentro de grandes datasets de texto, sin necesitar datos etiquetados.

En esta sección nos centraremos en técnicas de análisis textual más avanzadas que ofrecen los algoritmos integrados BlazingText y Sequence-to-Sequence de Amazon SageMaker.

#### BlazingText

**BlazingText** es un algoritmo eficiente y escalable que ofrece Amazon SageMaker para tareas de NLP, optimizado específicamente para la clasificación de textos y para generar embeddings de palabras con el modelo Word2Vec. BlazingText está diseñado para procesar grandes volúmenes de datos de texto con rapidez, lo que lo hace adecuado para aplicaciones en tiempo real y datasets grandes. Este algoritmo es especialmente beneficioso para tareas como la similitud semántica, el análisis de sentimiento y la clasificación de documentos, donde es crucial entender las relaciones entre las palabras.

> [!note] Recuadro del libro
> Un **embedding**, en el contexto de NLP y ML, es una representación vectorial densa de datos, típicamente palabras, que captura sus significados, sus propiedades sintácticas y sus relaciones con otras palabras. Estos embeddings se aprenden de los datos y se usan para convertir datos categóricos en datos numéricos continuos, que luego pueden procesar los modelos de ML. Por ejemplo, en los embeddings de palabras, cada palabra de un vocabulario se asigna a un espacio vectorial de alta dimensión. Las palabras con significados o usos similares quedan cerca unas de otras en este espacio, lo que permite a los modelos entender la similitud semántica y las relaciones entre palabras. Dicho de otro modo, los embeddings traducen la similitud semántica, tal como la perciben los humanos, en proximidad dentro de un espacio vectorial. Entre las técnicas comunes de embeddings están Word2Vec, GloVe y FastText, que han mejorado significativamente el rendimiento de diversas tareas de NLP, como la clasificación de textos, el análisis de sentimiento y la traducción automática.

Para quien viene de la estadística, un embedding de palabras es comparable a una reducción de dimensionalidad de la matriz de coocurrencias de palabras: se pasa de un one-hot de 50 000 posiciones (una por palabra del vocabulario) a un vector denso de, digamos, 100 o 300 números. El ejemplo clásico de que esos vectores capturan significado es la aritmética $\vec{\text{rey}} - \vec{\text{hombre}} + \vec{\text{mujer}} \approx \vec{\text{reina}}$. La cercanía se mide casi siempre con la **similitud coseno** (el coseno del ángulo entre dos vectores).

BlazingText opera utilizando dos técnicas principales: Word2Vec y la clasificación de textos. El modelo Word2Vec crea representaciones vectoriales densas de las palabras, también conocidas como embeddings de palabras, analizando grandes corpus de texto y captando las relaciones semánticas y sintácticas entre las palabras. Lo hace con una tarea auxiliar: predecir las palabras vecinas a partir de una palabra (variante _skip-gram_) o la palabra a partir de sus vecinas (variante _CBOW_, _continuous bag of words_); los vectores que resuelven bien esa tarea resultan ser buenos embeddings. Estos embeddings de palabras se usan luego para entender las similitudes y analogías entre palabras. Para la clasificación de textos, BlazingText emplea implementaciones eficientes de modelos de aprendizaje profundo que pueden categorizar texto en clases predefinidas a partir de los embeddings de palabras aprendidos; según la documentación, este modo es similar a **fastText**, un clasificador que promedia los embeddings de las palabras (y de fragmentos de palabras) del documento y aplica encima un clasificador lineal. El algoritmo aprovecha el **multihilo** (_multithreading_) y la **aceleración por hardware** para acelerar el proceso de entrenamiento, asegurando un rendimiento rápido y escalable. Multihilo significa repartir el trabajo entre los núcleos de la CPU para que corran en paralelo; aceleración por hardware significa, aquí, usar GPU, procesadores con miles de núcleos pequeños diseñados para operaciones matriciales masivas.

Para usar BlazingText en Amazon SageMaker, empiezas preparando tus datos de texto y subiéndolos a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). A continuación, creas un estimador, especificando BlazingText como algoritmo, y configuras los hiperparámetros necesarios para tu trabajo de entrenamiento. Una vez configurado el estimador, lanzas el trabajo de entrenamiento apuntándolo a la ubicación de tus datos en Amazon S3. Después de terminar el entrenamiento, puedes desplegar el modelo en un endpoint para predicciones en tiempo real o usarlo para inferencia por lotes. El siguiente código en Python puede usarse como referencia:

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

Observa cómo puedes especificar el modo de operación del estimador en la instrucción `bt_estimator.set_hyperparameters()`.

Este método recibe los hiperparámetros como argumentos con nombre (_keyword arguments_), cuyas claves están definidas en la documentación de Amazon SageMaker. (El libro dice que recibe «un objeto diccionario como único parámetro»; en el código se pasan como `clave=valor`, que es equivalente a desempaquetar un diccionario con `**`.)

Los hiperparámetros del algoritmo BlazingText dependen del modo que uses: Word2Vec (no supervisado) o Text Classification (supervisado). Para más información sobre este método, visita https://docs.aws.amazon.com/sagemaker/latest/dg/blazingtext_hyperparameters.html.

Algunos detalles del fragmento que el libro no explica. `mode='supervised'` elige la clasificación de textos; los modos de Word2Vec son `skipgram`, `cbow` y `batch_skipgram`. En modo supervisado, el archivo de entrenamiento debe tener una oración por línea, precedida de su etiqueta con el prefijo `__label__` (por ejemplo, `__label__positivo me encantó el producto`), y el texto conviene tokenizarlo antes (separar la puntuación con espacios). `min_count=2` descarta las palabras que aparecen menos de dos veces en el corpus. La instancia `ml.c4.2xlarge` tiene 8 vCPU optimizadas para cómputo (la «c» de _compute_) y unos 15 GiB de memoria; es de una generación anterior, igual que la `ml.m4.xlarge` del endpoint. El código también requiere la versión 2 del SDK (`pip install "sagemaker<3"`) y el endpoint queda cobrando por hora hasta que se borra.

##### Casos de uso (_Use Cases_)

BlazingText es ideal para escenarios que requieren una clasificación de textos o una creación de embeddings de palabras eficiente y escalable. Es especialmente útil en aplicaciones que involucran grandes volúmenes de datos de texto, como el procesamiento de reseñas de clientes, el análisis de redes sociales y la categorización de documentos. El modelo Word2Vec de BlazingText es excelente para tareas como la similitud semántica, el clustering de palabras y la construcción de representaciones de características para tareas de NLP posteriores. Además, su velocidad y su escalabilidad lo hacen adecuado para aplicaciones en tiempo real y datasets grandes, donde el procesamiento rápido y el alto rendimiento son esenciales.

Sin embargo, BlazingText puede no ser la mejor opción para tareas de comprensión del lenguaje natural más complejas, que requieren contexto más allá de los embeddings de palabras, como el análisis contextual profundo o la generación de lenguaje con matices. Para tareas que implican entender el contexto de oraciones o párrafos, como responder preguntas o la IA conversacional avanzada, modelos fundacionales como `cohere.embed-english-v3` y `cohere.embed-multilingual-v3` podrían ser más apropiados por su capacidad de captar la información contextual con mayor eficacia. Estos modelos fundacionales producen embeddings y están disponibles en Amazon Bedrock. BlazingText también es menos adecuado para tareas que requieren una personalización extensa más allá de la clasificación de textos y los embeddings de palabras, donde modelos más especializados o algoritmos construidos a la medida pueden dar mejores resultados.

La limitación de fondo es que Word2Vec asigna **un solo vector por palabra**, sin importar el contexto: «banco» tiene el mismo embedding en «me senté en el banco» y en «abrí una cuenta en el banco». Los modelos de embeddings basados en Transformers, como los de Cohere, producen un vector por **oración o párrafo** que depende de todas sus palabras, y por eso distinguen esos dos usos. Es la base de la búsqueda semántica y de RAG. En Bedrock, además de los modelos Embed de Cohere (incluida una versión más nueva, Embed v4), hay modelos de embeddings de Amazon como Titan Text Embeddings V2 y Nova Multimodal Embeddings; verifica en la ficha de cada uno su estado y sus regiones.

#### Sequence-to-Sequence

El algoritmo **Sequence-to-Sequence** (Seq2Seq) de Amazon SageMaker es un algoritmo de aprendizaje supervisado diseñado para tareas en las que la entrada es una secuencia de **tokens** (como texto o audio) y la salida es otra secuencia de tokens. Esto lo hace adecuado para aplicaciones como la traducción automática (traducir texto de un idioma a otro), el resumen de textos (crear un resumen conciso de un texto más largo) y la conversión de voz a texto (convertir el lenguaje hablado en texto escrito). Seq2Seq aprovecha arquitecturas avanzadas de redes neuronales, incluidas las RNN y las **redes neuronales convolucionales** (CNN) con **mecanismos de atención**, para modelar y generar secuencias de forma eficaz. La arquitectura clásica tiene dos partes: un **codificador** lee la secuencia de entrada y la resume, y un **decodificador** genera la salida token por token. El mecanismo de atención permite al decodificador, en cada paso, «mirar» con distinto peso cada parte de la entrada (al traducir la tercera palabra, atiende sobre todo a las palabras de origen correspondientes), en lugar de depender de un único resumen de toda la oración. Las CNN se explican en la sección de clasificación de imágenes.

##### Casos de uso (_Use Cases_)

El algoritmo Seq2Seq es muy eficaz para tareas donde la entrada y la salida son secuencias, lo que lo convierte en una excelente opción para aplicaciones como la traducción automática, el resumen de textos y la respuesta a preguntas. Por ejemplo, en la traducción automática, Seq2Seq puede traducir texto de un idioma a otro entendiendo el contexto de la secuencia de entrada y generando una secuencia correspondiente en el idioma de destino. Del mismo modo, en el resumen de textos, los modelos Seq2Seq pueden condensar documentos largos en resúmenes concisos, identificando y conservando la información más relevante. Además, Seq2Seq es útil en el desarrollo de chatbots para generar respuestas coherentes y contextualmente apropiadas, lo que lo convierte en una herramienta versátil para una amplia gama de tareas de NLP que requieren transformar o generar secuencias.

No obstante, Seq2Seq puede no ser la mejor opción para tareas que no involucran principalmente la generación o la transformación de secuencias. Por ejemplo, si el objetivo es realizar una clasificación de textos simple, como identificar el sentimiento de un tuit (positivo o negativo), otros modelos como los clasificadores tradicionales o incluso las CNN podrían ser más eficientes y consumir menos recursos. Del mismo modo, las tareas que requieren el análisis de características estáticas, como la clasificación de imágenes y la predicción de series de tiempo sin necesidad de una salida secuencial, son más adecuadas para otros algoritmos especializados. En esos casos, las capacidades de Seq2Seq pueden ser excesivas y no aportar beneficios adicionales frente a modelos más simples diseñados para esas tareas específicas.

> [!warning] Nota de precisión: Seq2Seq frente a los servicios y los modelos fundacionales
> Entrenar un Seq2Seq requiere un corpus paralelo grande (millones de pares de oraciones origen-destino para traducción, o miles de horas de audio transcrito para voz a texto). Para traducir, transcribir o conversar sin ese corpus, los servicios Translate, Transcribe y Lex, o un modelo fundacional de Bedrock, son hoy la opción habitual. El Seq2Seq integrado tiene sentido cuando necesitas un modelo propio para un dominio muy específico y tienes los datos para entrenarlo.

### Algoritmos de procesamiento de imágenes (_Image Processing Algorithms_)

Amazon SageMaker ofrece un conjunto de algoritmos integrados diseñados para agilizar diversas tareas de procesamiento de imágenes, aprovechando técnicas avanzadas de ML para ofrecer alta precisión y eficiencia. Entre ellos están algoritmos como **Image Classification**, que categoriza imágenes en clases predefinidas; **Object Detection**, que identifica y ubica objetos dentro de las imágenes; **Semantic Segmentation**, que clasifica cada píxel de una imagen para distinguir los distintos objetos, e **Image Embeddings**, que transforma imágenes en vectores de tamaño fijo para usarlos en diversas tareas posteriores. Cada uno de estos algoritmos ofrece herramientas potentes para automatizar y mejorar los flujos de trabajo de procesamiento de imágenes, lo que los hace esenciales para desarrollar aplicaciones sofisticadas de visión por computadora.

> [!warning] Nota de precisión: _Image Embeddings_ no es un algoritmo integrado
> En la lista vigente de algoritmos integrados no hay uno llamado _Image Embeddings_. Los embeddings de imágenes están entre los tipos de problema que cubren los **modelos preentrenados de SageMaker JumpStart**, y en Bedrock hay modelos de embeddings multimodales (texto e imagen).

#### Clasificación de imágenes (_Image Classification_)

La clasificación de imágenes es una tarea fundamental de la visión por computadora, cuyo objetivo es asignar una etiqueta o categoría a una imagen de entrada. Este proceso implica analizar el contenido de la imagen y categorizarla en una de varias clases predefinidas. Por ejemplo, en un dataset de fotos de animales, un modelo de clasificación de imágenes puede etiquetar cada imagen como «gato», «perro», «ave», etc. La clasificación de imágenes se usa ampliamente en diversas aplicaciones, como las imágenes médicas, la conducción autónoma, la videovigilancia de seguridad y muchas otras, donde la identificación precisa de los objetos de las imágenes es clave.

La clasificación de imágenes es un algoritmo de ML supervisado que aprovecha técnicas de aprendizaje profundo, en particular las CNN, para procesar y analizar imágenes. Las CNN están diseñadas para aprender de forma automática y adaptativa **jerarquías espaciales** de características a partir de las imágenes. El proceso implica varias capas de operaciones convolucionales, de **pooling** y **totalmente conectadas**, que trabajan juntas para extraer características y hacer predicciones. Las tres capas, en breve:

- Una **capa convolucional** desliza pequeños filtros (por ejemplo, de 3 × 3 píxeles) por toda la imagen y mide cuánto se parece cada zona al patrón del filtro. Los filtros se aprenden de los datos: los de las primeras capas detectan bordes y texturas; los de capas profundas, partes de objetos (ojos, ruedas). Esa progresión de lo simple a lo complejo es la «jerarquía espacial».
- Una **capa de pooling** reduce la resolución quedándose, por ejemplo, con el valor máximo de cada bloque de 2 × 2, lo que abarata el cálculo y hace al modelo menos sensible a desplazamientos pequeños del objeto.
- Las **capas totalmente conectadas**, al final, combinan todas las características extraídas para producir la probabilidad de cada clase, como una regresión logística multinomial sobre las características que aprendió la red.

Durante el entrenamiento, el modelo aprende a reconocer patrones y características dentro de las imágenes minimizando el error de clasificación sobre un gran dataset de imágenes etiquetadas. Una vez entrenado, el modelo puede predecir con precisión la clase de imágenes nuevas, no vistas, a partir de las características que aprendió.

> [!note] Recuadro del libro
> El algoritmo integrado Image Classification viene en tres «sabores»: el primero está implementado con la popular plataforma de ML TensorFlow, el segundo con el framework de ML PyTorch y el tercero con la biblioteca de aprendizaje profundo MXNet.

> [!warning] Nota de precisión: los sabores de Image Classification (verificado el 24-09-2026)
> La documentación vigente lista **dos** algoritmos integrados de clasificación de imágenes: **Image Classification - MXNet** e **Image Classification - TensorFlow**. No hay una variante integrada de PyTorch; los modelos de PyTorch para clasificar imágenes están disponibles como modelos preentrenados de JumpStart o como tu propio script en un contenedor de framework. La variante de TensorFlow hace **aprendizaje por transferencia** (_transfer learning_): parte de un modelo ya entrenado con millones de imágenes genéricas y solo ajusta las últimas capas con tus imágenes, lo que requiere muchos menos datos etiquetados. Nota adicional: el proyecto Apache MXNet se retiró en 2023 y ya no tiene desarrollo activo, así que para un proyecto nuevo conviene la variante de TensorFlow o JumpStart.

Para usar Image Classification en Amazon SageMaker, empiezas preparando tu dataset etiquetado y subiéndolo a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). A continuación, creas un modelo de clasificación de imágenes con un estimador de Amazon SageMaker, especificando el algoritmo y los hiperparámetros necesarios y apuntando a tu dataset. El estimador se encarga del proceso de entrenamiento, utilizando la potente infraestructura de Amazon SageMaker para entrenar tu modelo de forma eficiente. Una vez terminado el entrenamiento, puedes desplegar el modelo como un endpoint para predicciones en tiempo real o para procesamiento por lotes. Amazon SageMaker también ofrece herramientas para la evaluación y el monitoreo de modelos, que ayudan a asegurar que tu modelo de clasificación de imágenes rinda de forma óptima. Estas herramientas se analizarán en el próximo capítulo. Los algoritmos de visión se entrenan normalmente en instancias con GPU, que son mucho más caras por hora que las de CPU; por eso importa que los datos lleguen rápido (ver FSx for Lustre al inicio de esta sección).

##### Casos de uso (_Use Cases_)

Los algoritmos de clasificación de imágenes son ideales cuando necesitas categorizar imágenes en clases específicas según su contenido. Estos algoritmos son especialmente útiles en escenarios donde se requiere una identificación y un etiquetado precisos de las imágenes. Por ejemplo, en las imágenes médicas, pueden usarse para clasificar radiografías o resonancias magnéticas y detectar enfermedades. En la industria minorista, la clasificación de imágenes puede ayudar a organizar productos identificando los artículos de las imágenes. Además, estos algoritmos son valiosos en los sistemas de seguridad y vigilancia para reconocer rostros, placas de autos u otros objetos de interés. En general, son beneficiosos en cualquier aplicación cuyo objetivo sea categorizar datos visuales de forma automática y precisa.

Los algoritmos de clasificación de imágenes pueden no ser adecuados para tareas que requieren entender el contexto o las relaciones entre múltiples objetos dentro de una imagen. Por ejemplo, si el objetivo es detectar y ubicar varios objetos en una imagen (detección de objetos) o segmentar distintas regiones (segmentación semántica), se necesitarían algoritmos más especializados. Además, las tareas que implican analizar imágenes para extraer información detallada y estructurada, como el OCR, pueden requerir enfoques distintos (Textract, en AWS). Por último, en casos donde las imágenes carecen de distinciones claras entre categorías o donde el objetivo principal no es la clasificación sino otras formas de análisis, los algoritmos de clasificación de imágenes podrían no ser la opción más eficaz.

#### Detección de objetos (_Object Detection_)

Con la detección de objetos, el objetivo no es solo identificar los objetos de una imagen, sino también señalar su ubicación precisa mediante **cuadros delimitadores** (_bounding boxes_). Un cuadro delimitador es un rectángulo definido por coordenadas (por ejemplo, esquina superior izquierda, ancho y alto) que encierra cada objeto detectado, acompañado de su clase y de un puntaje de confianza. Esta doble capacidad de clasificación y localización es fundamental para aplicaciones como la conducción autónoma, los sistemas de seguridad, la inteligencia, la vigilancia, el reconocimiento, el sector minorista y muchas otras. Como algoritmo de aprendizaje supervisado, la detección de objetos requiere datasets etiquetados para el entrenamiento, en los que cada imagen incluye anotaciones detalladas que indican los objetos y sus ubicaciones.

El algoritmo Object Detection de Amazon SageMaker emplea técnicas sofisticadas de aprendizaje profundo, con arquitecturas populares como **Single Shot MultiBox Detector** (SSD), las **redes neuronales convolucionales basadas en regiones** (R-CNN, _region-based convolutional neural networks_) y **You Only Look Once** (YOLO). La diferencia entre familias: R-CNN trabaja en dos etapas (primero propone regiones candidatas, luego clasifica cada una), lo que suele ser más preciso y más lento; SSD y YOLO lo hacen en una sola pasada de la red, lo que las hace más rápidas y adecuadas para video en tiempo real. El enfoque SSD integra, como **red base** (_backbone_), una CNN preentrenada para tareas de clasificación de imágenes. Arquitecturas populares de CNN como VGG-16 y ResNet-50 suelen usarse como backbone. El algoritmo divide la imagen de entrada en una cuadrícula y, para cada celda, predice múltiples cuadros delimitadores y las probabilidades de clase de los objetos dentro de esos cuadros. Durante el entrenamiento, el modelo recibe un dataset de imágenes junto con sus anotaciones de cuadros delimitadores y sus etiquetas de clase correspondientes. Aprende a minimizar el error entre sus predicciones y los datos de la verdad de referencia (_ground truth_: las anotaciones correctas, hechas por personas), refinando eficazmente su capacidad de detectar y clasificar objetos con precisión. La implementación de SSD de Amazon SageMaker también incluye técnicas de **aumento de datos** (_data augmentation_), como el volteo, el reescalado y el _jittering_ (pequeñas alteraciones aleatorias de color, brillo o posición), para mejorar la robustez del modelo y evitar el sobreajuste. El aumento de datos genera variantes de cada imagen de entrenamiento, de modo que el modelo aprende que un auto volteado horizontalmente sigue siendo un auto.

> [!warning] Nota de precisión: qué arquitecturas usa cada variante
> Según la documentación, **Object Detection - MXNet** implementa SSD (con backbones como VGG o ResNet), y **Object Detection - TensorFlow** hace aprendizaje por transferencia a partir de modelos preentrenados de TensorFlow. YOLO y Faster R-CNN aparecen en la documentación como modelos preentrenados de **JumpStart**, no como opciones del algoritmo integrado. Para el examen, basta con saber que el algoritmo integrado de detección de objetos devuelve clases y cuadros delimitadores.

Igual que con otros algoritmos integrados, para utilizar la detección de objetos en Amazon SageMaker empiezas preparando y etiquetando tu dataset, asegurándote de que cada imagen contenga anotaciones de los objetos de interés. Este dataset etiquetado se sube después a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Puedes crear un modelo de detección de objetos con un estimador de Amazon SageMaker, especificando el algoritmo (MXNet o TensorFlow) y los hiperparámetros necesarios y apuntando a tu dataset. El estimador gestiona el proceso de entrenamiento, aprovechando la infraestructura administrada de Amazon SageMaker para optimizar el modelo con tus datos. Después de terminar el entrenamiento, puedes desplegar el modelo como un endpoint para predicciones en tiempo real o para procesamiento por lotes.

##### Casos de uso (_Use Cases_)

La detección de objetos es muy eficaz en escenarios donde son críticas tanto la identificación como la localización precisa de los objetos dentro de las imágenes. Esto la hace ideal para aplicaciones como la conducción autónoma, donde los vehículos necesitan detectar y seguir peatones, otros autos y obstáculos para navegar con seguridad. También es valiosa en los sistemas de seguridad y vigilancia, que dependen de identificar amenazas potenciales o actividades sospechosas reconociendo y siguiendo en tiempo real objetos como rostros, vehículos y bolsas. En el sector minorista, la detección de objetos puede ayudar a gestionar el inventario reconociendo y contando automáticamente los productos en los anaqueles, así como a analizar el comportamiento de los clientes siguiendo sus movimientos dentro de la tienda. Además, se usa en imágenes médicas para ubicar e identificar anomalías o enfermedades en los estudios, lo que ayuda al diagnóstico y a la planeación del tratamiento. En aplicaciones militares, especialmente en inteligencia, vigilancia y reconocimiento (ISR, _intelligence, surveillance, and reconnaissance_), la detección de objetos es crucial para identificar y seguir vehículos, personal y equipo enemigos a partir de imágenes de vigilancia aérea y terrestre, mejorando la conciencia situacional y la toma de decisiones en el campo de batalla.

La detección de objetos puede no ser tu mejor opción para tareas que requieren entender el contexto o las relaciones entre objetos sin necesitar una localización precisa. Por ejemplo, si el objetivo es simplemente clasificar el contenido general de una imagen, como determinar si una imagen contiene un gato o un perro, los algoritmos de clasificación de imágenes son más apropiados. Del mismo modo, para tareas que implican segmentar imágenes en regiones según los límites de los objetos, como distinguir entre distintos tipos de tejido en un estudio médico, los algoritmos de segmentación semántica serían más eficaces. Además, si el objetivo principal es analizar características o patrones estáticos en datos no visuales, como texto o series de tiempo, deberían usarse otros algoritmos especializados. La fortaleza de la detección de objetos está en su capacidad no solo de reconocer objetos, sino también de dar su ubicación exacta, lo que puede ser innecesario para tareas más simples de clasificación o segmentación.

#### Segmentación semántica (_Semantic Segmentation_)

La segmentación semántica es una técnica potente de visión por computadora que consiste en etiquetar cada píxel de una imagen con una etiqueta de clase de un conjunto predefinido de clases. A diferencia de la detección de objetos, que identifica los objetos y sus cuadros delimitadores, y de la clasificación de imágenes, que analiza solo imágenes completas y las clasifica en una de múltiples categorías de salida, la segmentación semántica ofrece una comprensión detallada de la imagen al etiquetar cada píxel según su categoría. Esto la hace invaluable para aplicaciones que requieren una localización y una diferenciación precisas de los objetos dentro de una imagen. Las tres tareas forman una escalera de detalle: la clasificación responde «¿qué hay?» con una etiqueta por imagen; la detección, «¿qué hay y dónde, aproximadamente?» con un rectángulo por objeto; la segmentación, «¿a qué clase pertenece cada píxel?» con un contorno exacto. «Semántica» significa que distingue clases, no individuos: dos peatones pegados quedan en la misma región «peatón».

Los algoritmos de segmentación semántica suelen usar modelos de aprendizaje profundo, en particular CNN con arquitecturas especializadas como las **redes totalmente convolucionales** (FCN, _fully convolutional networks_), el **análisis piramidal de escenas** (PSP, _pyramid scene parsing_) y **DeepLabV3**. Estos modelos parten de una red de clasificación preentrenada, que se modifica para producir **mapas de segmentación** en lugar de probabilidades de clase. La red procesa la imagen de entrada a través de múltiples capas convolucionales para extraer características en varios niveles de abstracción. Durante el entrenamiento, el modelo aprende a relacionar estas características con etiquetas a nivel de píxel mediante una función de pérdida que mide la diferencia entre las etiquetas predichas y las verdaderas. Suelen emplearse técnicas como el **sobremuestreo espacial** (_upsampling_) y las **conexiones de salto** (_skip connections_) para mejorar la resolución espacial de la salida, asegurando que el mapa de segmentación represente con precisión los detalles finos y los límites de los objetos dentro de la imagen. El problema que resuelven es que las capas de pooling reducen la resolución (una imagen de 512 × 512 puede quedar en 16 × 16 en las capas profundas), pero la salida debe tener una etiqueta por cada píxel original. El upsampling vuelve a ampliar esos mapas, y las conexiones de salto llevan directamente la información de alta resolución de las primeras capas hasta la salida, para recuperar los bordes finos.

Para usar la segmentación semántica en Amazon SageMaker, primero necesitas preparar y etiquetar tu dataset, asegurándote de que cada imagen tenga una **máscara de segmentación** correspondiente que etiquete cada píxel. Una máscara es una imagen del mismo tamaño que la original en la que el valor de cada píxel es el número de su clase (0 = fondo, 1 = carretera, 2 = peatón…). Este dataset etiquetado se sube después a un bucket de Amazon S3 (o a Amazon EFS o Amazon FSx for Lustre). Puedes crear un modelo de segmentación semántica con un estimador de Amazon SageMaker, especificando el algoritmo (como U-Net o DeepLab) y los hiperparámetros necesarios y apuntando a tu dataset. El estimador se encarga del proceso de entrenamiento, aprovechando la infraestructura robusta de Amazon SageMaker para optimizar el modelo con tus datos. Una vez terminado el entrenamiento, puedes desplegar el modelo como un endpoint para predicciones en tiempo real o para procesamiento por lotes.

> [!warning] Nota de precisión: U-Net no está entre las opciones del algoritmo integrado
> El algoritmo integrado de segmentación semántica ofrece las arquitecturas que el propio libro menciona antes (**FCN, PSP y DeepLabV3**), sobre backbones ResNet preentrenados, implementadas con MXNet. **U-Net**, muy popular en imágenes médicas, no figura entre sus opciones; para usarla hay que traer tu propio código en un contenedor de framework.

##### Casos de uso (_Use Cases_)

La segmentación semántica se aprovecha mejor en escenarios donde se requiere una comprensión detallada de la imagen, hasta el nivel del píxel. Esto la hace ideal para aplicaciones de imágenes médicas, como segmentar distintos tipos de tejidos, órganos o tumores en resonancias magnéticas o tomografías computarizadas, lo que es crucial para un diagnóstico y una planeación del tratamiento precisos (el volumen de un tumor, por ejemplo, se calcula contando sus píxeles). En la conducción autónoma, la segmentación semántica se usa para entender el entorno identificando y diferenciando objetos como carreteras, banquetas, vehículos y peatones. Esta segmentación detallada ayuda a una navegación y una toma de decisiones seguras. Además, en el monitoreo agrícola, la segmentación semántica puede usarse para diferenciar entre cultivos y maleza, evaluar la salud de las plantas y monitorear las etapas de crecimiento, lo que ayuda a prácticas agrícolas eficientes. Esta técnica también es valiosa en las imágenes satelitales, donde se crean mapas detallados para identificar distintos tipos de cobertura del suelo, como bosques, áreas urbanas y cuerpos de agua, para el monitoreo ambiental y la planeación urbana.

La segmentación semántica no se recomienda para tareas que no requieren precisión a nivel de píxel o donde el objetivo principal es simplemente clasificar una imagen completa o detectar objetos con cuadros delimitadores. Por ejemplo, si el objetivo es categorizar imágenes en clases amplias, como identificar si una imagen contiene un gato o un perro, los algoritmos de clasificación de imágenes son más apropiados. Del mismo modo, para aplicaciones que necesitan ubicar y clasificar objetos dentro de una imagen, como detectar rostros o vehículos, los algoritmos de detección de objetos son más adecuados, porque dan cuadros delimitadores alrededor de los objetos detectados y no requieren anotaciones detalladas a nivel de píxel. Además, la segmentación semántica puede ser menos eficaz en casos donde el costo computacional y la complejidad son prohibitivos, o donde hay pocos datos etiquetados disponibles para el entrenamiento, ya que requiere un número significativo de imágenes anotadas para lograr una alta precisión. Anotar una máscara píxel por píxel puede tomar varios minutos por imagen, frente a segundos para una etiqueta de clasificación, así que el costo de etiquetado es muy superior.

## Criterios para la selección de modelos (_Criteria for Model Selection_)

Elegir el modelo correcto para tu problema de ML implica considerar varios factores críticos. Esta sección describe los criterios clave que te guiarán para seleccionar el mejor modelo para tu caso de uso específico e incluye ejemplos de algoritmos que se ajustan a cada criterio.

- **Precisión (_Accuracy_).** El objetivo principal de cualquier modelo de ML es lograr una alta precisión en la tarea dada. Algoritmos como XGBoost y las SVM son conocidos por su alta precisión en diversas aplicaciones, incluidas la clasificación y la regresión. (Aquí «precisión» se usa en sentido amplio, como calidad predictiva; no confundir con la métrica _precision_ de clasificación, que se ve en el capítulo 5.)
- **Interpretabilidad (_Interpretability_).** La interpretabilidad es tan importante como la precisión, sobre todo en ámbitos donde es esencial entender el proceso de decisión. Los modelos Linear Learner y de regresión logística son muy interpretables y dan una visión clara de cómo se hacen las predicciones. Los árboles de decisión también son interpretables, porque mapean visualmente el proceso de decisión. Estos modelos equilibran la precisión con la interpretabilidad, asegurando que sus decisiones puedan ser fácilmente entendidas por las partes interesadas y que estas confíen en ellas.
- **Escalabilidad (_Scalability_).** La capacidad de un modelo de manejar volúmenes de datos y una complejidad crecientes es crítica para aplicaciones a gran escala. Algoritmos como Linear Learner y el clustering K-means escalan de forma eficiente y pueden manejar datasets grandes. Random Cut Forest también es escalable, lo que lo hace adecuado para la detección de anomalías en escenarios de big data. Estos algoritmos escalan porque procesan los datos en mini-batches o con muestreo (no necesitan tenerlos todos en memoria) y porque pueden repartir el entrenamiento entre varias instancias.
- **Latencia y velocidad (_Latency and speed_).** En aplicaciones en tiempo real, la velocidad con la que un modelo puede dar predicciones es crucial. Algoritmos como el random forest ofrecen tiempos de inferencia rápidos, lo que los hace ideales para escenarios donde las decisiones oportunas son críticas, como la detección de fraude y los sistemas de recomendación.
- **Requisitos de recursos (_Resource requirements_).** Los recursos computacionales necesarios para el entrenamiento y la inferencia pueden afectar la viabilidad de usar ciertos modelos. Los Linear Learners, k-NN y PCA generalmente requieren menos recursos que las redes neuronales profundas. Evaluar los requisitos de recursos asegura que los modelos se ajusten a tu infraestructura y tu presupuesto disponibles.
- **Disponibilidad y calidad de los datos (_Data availability and quality_).** La disponibilidad y la calidad de los datos influyen significativamente en el rendimiento del modelo. Algoritmos como LDA y BlazingText rinden bien con datasets textuales grandes, y K-means puede funcionar eficazmente con datasets más pequeños para tareas de clustering. Asegurar datos de alta calidad para el entrenamiento es crítico para desarrollar modelos precisos y confiables.
- **Consideraciones regulatorias y éticas (_Regulatory and ethical considerations_).** En muchas industrias, los modelos deben cumplir requisitos regulatorios y estándares éticos. La regresión logística y los árboles de decisión suelen preferirse en industrias reguladas por su transparencia y su facilidad de explicación. Las consideraciones éticas son especialmente importantes en campos como la salud y las finanzas, donde las implicaciones de las predicciones de un modelo pueden tener consecuencias significativas. Por ejemplo, en muchos países un banco que rechaza un crédito debe poder explicarle al solicitante los motivos principales de la decisión, lo que es directo con los coeficientes de una regresión logística y difícil con un ensamble de cientos de árboles.
- **Costo (_Cost_).** El costo de implementar y mantener distintos modelos puede variar mucho. Modelos como Linear Learner y random forest son generalmente rentables, mientras que los modelos de aprendizaje profundo como las CNN pueden incurrir en costos más altos por sus exigencias computacionales. Considerar las implicaciones de costo, incluidos los recursos en la nube y el etiquetado de datos, es esencial para proyectos de largo plazo. En la nube, el costo se traduce casi directamente en horas de instancia: entrenar una CNN en una instancia con GPU puede costar decenas de veces más por hora que entrenar un modelo lineal en una instancia de CPU, y un endpoint encendido se paga aunque no reciba tráfico.

> [!warning] Nota de precisión: random forest y la latencia
> Este criterio contradice lo que el propio libro dice en los casos de uso de los árboles: que los random forests con muchos árboles profundos son «menos ideales para aplicaciones que requieren predicciones en tiempo real». Las dos cosas pueden ser ciertas según el tamaño: el tiempo de inferencia crece con el número de árboles y su profundidad. Un bosque de 100 árboles poco profundos responde en microsegundos o pocos milisegundos; uno de miles de árboles muy profundos puede no hacerlo. Recuerda también que random forest no es un algoritmo integrado de SageMaker (ver la nota de la lista de algoritmos).

A menudo hay un compromiso (_trade-off_) entre estos factores; por ejemplo, entre la precisión y la interpretabilidad, donde los modelos complejos que logran una alta precisión pueden ser difíciles de entender, mientras que los modelos más simples, con mejor interpretabilidad, pueden tener menor precisión. Tu trabajo como ingeniero de ML en AWS es equilibrar estos compromisos para satisfacer las necesidades y restricciones específicas de tu problema de ML, asegurando que el modelo elegido se alinee con los objetivos de negocio y los requisitos operativos. Esto implica evaluar y ajustar continuamente el modelo con datos nuevos y retroalimentación para lograr el mejor rendimiento posible, manteniendo la transparencia y la confianza.

Al considerar estos criterios y entender qué algoritmos se alinean con ellos, estarás bien preparado para seleccionar el modelo más apropiado para tu problema de ML. Este enfoque integral asegura que tus modelos no solo tengan un alto rendimiento, sino que también sean interpretables, escalables y estén alineados con tus requisitos específicos.

Además de elegir el algoritmo, este capítulo implica una decisión previa: **servicio de IA, Bedrock o SageMaker**. Una forma práctica de ordenarla, coherente con lo que dice el libro en cada sección:

| Si… | Empieza por… |
| --- | --- |
| La tarea es estándar (leer documentos, transcribir, traducir, detectar rostros) y no tienes datos etiquetados propios | Un servicio de IA (Textract, Transcribe, Translate, Rekognition…) |
| Necesitas generar o resumir contenido, o conversar, y aceptas un modelo de un proveedor | Bedrock |
| Tienes datos etiquetados propios y necesitas controlar el modelo, las características o los hiperparámetros | SageMaker AI (algoritmo integrado, o tu propio código) |

## Escenarios donde este servicio es la opción obligada

Este capítulo no trata de un solo servicio, sino de dos familias: los servicios de IA (y Bedrock) y los algoritmos integrados de SageMaker. Por eso cada escenario se centra en uno distinto, elegido porque, dadas las restricciones del caso, es el único razonable dentro de AWS. Dos son servicios de IA y uno es un algoritmo integrado, para reflejar la decisión central del capítulo. Nova Canvas quedó fuera a propósito: como llega a su fin de vida el 30-09-2026, no puede ser la opción obligada de un proyecto que empieza hoy. Las empresas son ficticias; las restricciones son las que aparecen en proyectos reales. Las características citadas se verificaron en la documentación de AWS el 24-09-2026.

### Escenario 1: apertura de cuentas en un banco digital → Amazon Rekognition (Face Liveness y CompareFaces)

**Contexto.** Un banco digital sin sucursales abre cuentas exclusivamente desde su app (web, iOS y Android). El cliente fotografía su identificación oficial y se toma una selfie. El área de prevención de fraude detectó aperturas con identificaciones robadas: el defraudador usaba una foto impresa del titular o un video en otra pantalla frente a la cámara. Dirección quiere el control en producción en seis semanas.

**Restricciones clave.**

1. **Verificación 1:1.** Hay que confirmar que la persona de la selfie es la misma de la foto de la identificación. No se trata de buscar a alguien en una base de clientes.
2. **Prueba de vida.** Hay que detectar que frente a la cámara hay una persona física y no una foto impresa, un video en una pantalla o una máscara.
3. **Sin datos ni equipo de visión.** El banco no tiene imágenes biométricas etiquetadas para entrenar nada, ni personas con experiencia en visión por computadora, y el plazo es de semanas.
4. **Decisión con umbral ajustable y evidencia.** Riesgo quiere fijar su propio punto de corte entre fraude aceptado y clientes legítimos rechazados, y auditoría exige conservar imágenes que respalden cada decisión.
5. **Experiencia en la app.** La verificación tiene que ocurrir dentro del flujo de apertura, con respuesta en segundos, en las tres plataformas.

**Por qué este servicio.**

- (1) → **CompareFaces** compara el rostro de una imagen de origen con los de una imagen de destino y devuelve un puntaje de similitud. Por defecto solo devuelve coincidencias con similitud de al menos 80, y ese umbral se cambia con el parámetro `SimilarityThreshold`.
- (2) → **Face Liveness** pide al usuario una breve video-selfie guiada y detecta ataques de suplantación presentados a la cámara o que intentan saltarse la cámara. Devuelve un puntaje de confianza de 0 a 100.
- (3) → Ambas son API de un modelo ya entrenado por AWS: no hay datos que etiquetar ni modelo que entrenar, y se paga por uso.
- (4) → Los dos puntajes son continuos y el umbral lo decide el banco, como recomienda la documentación. Face Liveness devuelve además una **imagen de referencia** de alta calidad, que es la que se pasa a CompareFaces, y hasta cuatro **imágenes de auditoría** extraídas del video.
- (5) → El componente `FaceLivenessDetector` del SDK de AWS Amplify (React, Swift para iOS y Android) resuelve la interfaz de captura y la retroalimentación al usuario en tiempo real.

La documentación advierte que Face Liveness es probabilístico y recomienda combinarlo con otros factores (geolocalización, códigos de un solo uso) para una decisión basada en riesgo. Los datos biométricos, además, suelen tener reglas legales propias de consentimiento y conservación; el equipo de cumplimiento debe revisarlas.

**Por qué no las alternativas.**

- **Amazon Textract (AnalyzeID)** lee los campos de texto de un documento de identidad, pero no compara rostros ni detecta suplantaciones; sirve como complemento para extraer el nombre y el número del documento.
- **Los algoritmos integrados de visión de SageMaker** (Image Classification, Object Detection) exigirían recolectar y etiquetar miles de imágenes biométricas, y un clasificador por identidad no puede reconocer a un cliente que nunca vio; además, ninguno ofrece prueba de vida, así que violan las restricciones 2 y 3.
- **Los modelos multimodales de Bedrock** responden preguntas sobre imágenes en texto libre, pero no están diseñados ni validados para verificación biométrica: no dan un puntaje de similitud calibrado ni prueba de vida (restricciones 1, 2 y 4).
- **Amazon Cognito** gestiona usuarios, contraseñas y MFA, pero no hace verificación facial.
- **Rekognition Streaming Video** no hace falta aquí y, en cualquier caso, ya no admite clientes nuevos desde el 30-04-2026.

### Escenario 2: subtítulos en vivo para un canal de noticias → Amazon Transcribe (streaming)

**Contexto.** Un canal de noticias en español que transmite en EE. UU. debe subtitular su programación en vivo las 24 horas, por la normativa de accesibilidad que le aplica. Hoy lo hace un equipo de estenotipistas que no alcanza a cubrir todos los turnos. La señal de audio sale de la mesa de control del estudio y los subtítulos se insertan en la señal de video.

**Restricciones clave.**

1. **En vivo y con poco retraso.** Los subtítulos deben aparecer pocos segundos después de que se pronuncian las palabras.
2. **Lectura estable.** Los subtítulos no pueden reescribirse constantemente en pantalla mientras el espectador los lee.
3. **Vocabulario propio.** Los noticieros están llenos de nombres de políticos, vecindarios, equipos y siglas que un modelo genérico escribe mal.
4. **Sincronía con el video.** Cada palabra debe poder ubicarse en el tiempo para alinearla con la imagen.
5. **Sin equipo de ML ni corpus de audio transcrito**, y el canal quiere pagar por uso, sin comprar ni operar servidores con GPU.
6. **Idioma**: español de EE. UU.

**Por qué este servicio.**

- (1) → Transcribe en modo **streaming** recibe el audio por una conexión bidireccional HTTP/2 o WebSocket y devuelve la transcripción como un flujo de eventos mientras se habla, en forma de **resultados parciales** que se completan al terminar cada segmento de habla.
- (2) → La **estabilización de resultados parciales** (niveles bajo, medio o alto) hace que solo las últimas palabras de un resultado parcial puedan cambiar, y marca cada palabra con un campo `Stable`. La documentación la recomienda explícitamente para subtitular transmisiones en vivo; se puede, por ejemplo, mostrar en cursiva las palabras aún no estables.
- (3) → Los **vocabularios personalizados** se aplican a las transcripciones en streaming (parámetro `VocabularyNames`). Para `es-US`, la tabla de idiomas indica además soporte de **modelos de lenguaje personalizados** en streaming, que se entrenan con texto del dominio (por ejemplo, guiones de noticieros archivados).
- (4) → Cada segmento y cada palabra traen `StartTime` y `EndTime`, que la documentación sugiere usar para sincronizar la transcripción con el video, además de un puntaje de confianza por palabra.
- (5) → Es un servicio de IA administrado y preentrenado: no hay modelo que entrenar ni infraestructura que operar.
- (6) → `es-US` admite streaming según la tabla de idiomas vigente.

**Por qué no las alternativas.**

- **Amazon Polly** hace lo contrario (texto → voz).
- **Amazon Lex** reconoce intenciones en frases cortas de un usuario que conversa con un bot; no está hecho para transcribir de forma continua una transmisión.
- **Amazon Comprehend** y **Amazon Translate** necesitan texto como entrada; pueden trabajar **después** de Transcribe (por ejemplo, Translate para ofrecer subtítulos en inglés), no en su lugar.
- **El algoritmo integrado Sequence-to-Sequence** de SageMaker tendría que entrenarse con miles de horas de audio transcrito, que el canal no tiene, y habría que construir encima la transmisión en vivo con resultados parciales (restricciones 1 y 5).
- **Bedrock**: sus modelos de voz, como Nova 2 Sonic, están pensados para conversaciones de voz a voz con un asistente, y los modelos que aceptan audio responden por petición; construir sobre ellos subtítulos en vivo con resultados parciales estabilizados, vocabularios propios y tiempos por palabra sería desarrollo propio (restricciones 2, 3 y 5).
- **Amazon Rekognition** analiza imágenes y video, no el audio.

### Escenario 3: robo de cuentas en una plataforma de videojuegos → algoritmo IP Insights de SageMaker

**Contexto.** Un estudio de videojuegos en línea tiene unos 20 millones de cuentas. Los objetos virtuales de las cuentas se revenden por dinero real, así que hay robo de cuentas: los atacantes prueban contraseñas filtradas en otros sitios y entran desde direcciones IP que el jugador nunca usó. El equipo de seguridad quiere pedir un segundo factor (MFA) solo cuando un inicio de sesión sea inusual, porque pedirlo siempre hace que muchos jugadores abandonen. Los registros de inicio de sesión contienen, entre otros campos, el identificador del jugador y la dirección IP.

**Restricciones clave.**

1. **Sistema de identidad propio.** La autenticación es un desarrollo interno, integrado con consolas y con socios comerciales, que no se puede migrar a otro sistema en el horizonte del proyecto.
2. **Sin etiquetas confiables.** Los robos se confirman semanas después, cuando el jugador reclama, son pocos y están incompletos.
3. **Decisión por inicio de sesión, en tiempo real**, con un puntaje que permita fijar el umbral para pedir MFA.
4. **Escala.** Unos 20 millones de identificadores de jugador, cada uno con su propio historial de IP.
5. **Reutilización.** El equipo de fraude de pagos quiere usar la señal como característica de su propio modelo.

**Por qué este servicio.**

- (2) → IP Insights es **no supervisado**: aprende de los pares (jugador, IP) observados en los registros y fabrica sus propias muestras negativas emparejando al azar jugadores con direcciones, así que no necesita casos etiquetados.
- (3) → Se despliega en un endpoint de SageMaker y devuelve para cada par (entidad, IPv4) un puntaje (`dot_product`) interpretable como log-odds de que el par sea habitual. La documentación describe justo este uso: si el puntaje indica un par anómalo, el servidor de inicio de sesión dispara la autenticación multifactor.
- (4) → Los identificadores se pasan tal como aparecen en los registros y el algoritmo los convierte con una función hash a un espacio fijo; la documentación incluye recomendaciones de instancia para hasta 50 millones de vectores de entidad, con un tamaño de modelo máximo de 8 GB. El codificador de direcciones aprovecha la estructura de prefijos de IPv4, de modo que las direcciones cercanas (del mismo proveedor) se representan de forma parecida.
- (5) → El algoritmo produce **embeddings** de las direcciones IP, y la documentación sugiere combinar su puntaje con otras características en otro modelo.
- (1) → Solo necesita los registros de inicio de sesión, sin tocar el sistema de identidad.

Dos limitaciones que el equipo debe conocer: IP Insights **solo admite IPv4**, así que los inicios de sesión por IPv6 necesitan otra regla; y los jugadores que se conectan por datos móviles cambian de dirección con frecuencia, lo que debilita la señal para ellos.

**Por qué no las alternativas.**

- **La protección contra amenazas de Amazon Cognito** (autenticación adaptativa, con un puntaje de riesgo por inicio de sesión que considera la IP, el dispositivo y la geografía) haría este trabajo, pero solo para usuarios de un grupo de usuarios de Cognito; exigiría migrar la identidad, lo que prohíbe la restricción 1.
- **AWS WAF Fraud Control ATP** compara usuario y contraseña con una base de credenciales robadas y bloquea clientes que envían demasiados intentos por IP o por sesión; es un buen complemento contra el uso de contraseñas filtradas, pero no aprende qué direcciones usa habitualmente cada jugador, así que un único inicio de sesión correcto desde una IP ajena pasa inadvertido.
- **Amazon Fraud Detector** ya no admite clientes nuevos desde el 07-11-2025.
- **Amazon GuardDuty** detecta amenazas contra tu cuenta e infraestructura de AWS (analiza, por ejemplo, registros de CloudTrail y flujos de red de la VPC), no los inicios de sesión de los jugadores en tu aplicación.
- **Random Cut Forest** detecta puntos atípicos en vectores numéricos; representar la relación entre 20 millones de identificadores y sus IP habituales requeriría una ingeniería de características que IP Insights ya resuelve (restricción 4).
- **XGBoost o Linear Learner** son supervisados y necesitarían las etiquetas que no existen (restricción 2).

## Resumen (_Summary_)

En este capítulo exploramos varios servicios de IA de AWS y los algoritmos integrados de Amazon SageMaker que pueden aprovecharse para distintos casos de uso de ML.

Los servicios de IA de AWS ofrecen una gama de modelos de ML prediseñados y preentrenados, pensados para simplificar y acelerar el despliegue de la IA en tus aplicaciones. Estos servicios incluyen Amazon Rekognition para el análisis de imágenes y video, Amazon Comprehend para NLP, Amazon Polly para la conversión de texto a voz, Amazon Lex para construir interfaces conversacionales, Amazon Textract para extraer texto y datos de documentos, Amazon Transcribe para convertir voz en texto, Amazon Translate para la traducción en tiempo real y Amazon Personalize para crear experiencias de usuario personalizadas. Amazon Bedrock, con los nuevos modelos fundacionales Nova, también ofrece capacidades robustas para tareas de IA generativa, mejorando la flexibilidad y el poder de tus aplicaciones de IA. Estos servicios de IA son totalmente administrados, requieren una experiencia mínima en ML y son ideales para desarrolladores que buscan integrar capacidades de IA rápidamente en sus aplicaciones.

Para una mayor flexibilidad y tareas de ML más especializadas, Amazon SageMaker ofrece una amplia gama de algoritmos integrados y soporte para frameworks populares como TensorFlow, PyTorch y MXNet, lo que te permite adaptar tus modelos de ML para cumplir requisitos específicos y lograr un rendimiento optimizado.

Exploramos los algoritmos de ML supervisado de Amazon SageMaker, diseñados para tareas que usan datos etiquetados para el entrenamiento. Los ejemplos incluyeron Linear Learner para regresión y clasificación, XGBoost para gradient boosting sobre árboles de decisión, k-NN para clasificación y regresión, y Factorization Machines para sistemas de recomendación y tareas de clasificación a gran escala. Estos modelos aprenden de los pares entrada-salida de los datos de entrenamiento para hacer predicciones con datos nuevos, no vistos.

Después aprendimos los algoritmos integrados de ML no supervisado de Amazon SageMaker, diseñados para descubrir patrones y estructuras ocultos en datos sin etiquetar. Estos algoritmos incluyen K-Means, para agrupar puntos de datos similares en grupos distintos; PCA, para la reducción de dimensionalidad; LDA y NTM, para el modelado de temas, y RCF e IP Insights, para la detección de anomalías.

Más adelante en el capítulo se presentaron algoritmos más avanzados que aprovechan redes neuronales profundas para el análisis textual y el procesamiento de imágenes. Entre ellos estaban los algoritmos integrados de Amazon SageMaker BlazingText, para embeddings de palabras y clasificación de textos eficientes; Seq2Seq, para tareas de secuencia a secuencia como la traducción automática y el resumen de textos; Image Classification, para categorizar imágenes en clases predefinidas; Object Detection, para identificar y ubicar múltiples objetos dentro de una imagen, y Semantic Segmentation, para la clasificación a nivel de píxel del contenido de las imágenes. Estos modelos de aprendizaje profundo están diseñados para manejar tareas complejas con alta precisión, lo que permite soluciones de IA robustas en diversos ámbitos como el NLP, la visión por computadora y los sistemas autónomos.

El capítulo concluyó con una lista seleccionada de criterios que debes considerar al elegir tu modelo, incluidos la precisión, la interpretabilidad, la escalabilidad, la latencia, los requisitos de recursos, la disponibilidad de datos, la regulación y el costo.

> [!warning] Actualización del resumen (verificado el 24-09-2026)
> Nova Canvas y Nova Reel llegan a su fin de vida el 30-09-2026 (los modelos Nova de texto Micro, Lite y Pro, y la familia Nova 2, siguen activos). El proyecto Apache MXNet se retiró en 2023; los algoritmos integrados basados en MXNet siguen documentados, pero para proyectos nuevos conviene TensorFlow, PyTorch o JumpStart.

## Puntos esenciales para el examen (_Exam Essentials_)

**Conoce los servicios de IA de AWS para visión.** AWS ofrece dos servicios de IA adaptados a tareas relacionadas con la visión. Amazon Rekognition permite el análisis de imágenes y video, lo que permite a los usuarios detectar objetos, rostros, lugares emblemáticos y texto, así como reconocer celebridades y analizar sentimientos. Para el procesamiento de documentos, Amazon Textract extrae texto, tablas y otros datos de documentos escaneados, lo que facilita digitalizar y gestionar eficientemente los flujos de trabajo documentales.

> [!warning] Nota de precisión: «analizar sentimientos» en Rekognition
> Lo que Rekognition estima es la **expresión facial aparente** (feliz, triste, sorprendido…) como atributo de cada rostro detectado, no el estado emocional real de la persona. El análisis de sentimiento **de texto** es de Comprehend. En una pregunta de examen, «sentimiento de reseñas» → Comprehend; «emociones en rostros de fotos» → Rekognition.

**Conoce los servicios de IA de AWS para voz y chatbots.** AWS ofrece dos servicios de IA para tareas relacionadas con la voz. Amazon Polly ofrece capacidades de texto a voz, transformando el contenido escrito en voz de sonido natural en múltiples idiomas. Amazon Transcribe ofrece reconocimiento automático de voz (ASR), convirtiendo grabaciones de audio en transcripciones de texto precisas. Amazon Lex permite crear interfaces conversacionales, combinando ASR y comprensión del lenguaje natural (NLU) para impulsar chatbots y asistentes de voz.

**Conoce los servicios de IA de AWS para lenguaje.** AWS ofrece dos servicios de IA para tareas de procesamiento del lenguaje. Amazon Comprehend permite el procesamiento de lenguaje natural (NLP) extrayendo conocimiento del texto, como el análisis de sentimiento, el reconocimiento de entidades y la extracción de frases clave. Amazon Translate ofrece servicios de traducción en tiempo real y por lotes, lo que permite una comunicación multilingüe fluida.

**Conoce los servicios de IA de AWS para IA generativa.** AWS ofrece Amazon Bedrock, con una gran selección de modelos fundacionales (FM), incluidos los recién lanzados modelos fundacionales Nova. Amazon Bedrock ofrece una plataforma robusta para crear y desplegar aplicaciones de IA generativa, lo que permite a los desarrolladores construir soluciones sofisticadas como la generación de texto, la síntesis de imágenes, los embeddings y más. Estos servicios permiten a los usuarios aprovechar con facilidad modelos de IA generativa de última generación, lo que hace posibles aplicaciones innovadoras y creativas en diversos ámbitos.

**Conoce la diferencia entre los algoritmos de clasificación y de regresión.** Los algoritmos de clasificación se usan para categorizar datos en clases o etiquetas distintas, como determinar si un correo es spam. Los algoritmos de regresión, en cambio, predicen valores numéricos continuos, como pronosticar precios de acciones o estimar el valor de casas a partir de diversas características.

**Conoce los algoritmos de Linear Learner que ofrece Amazon SageMaker.** Amazon SageMaker ofrece algoritmos de Linear Learner para tareas tanto de clasificación como de regresión. Estos incluyen la regresión lineal, la regresión logística y las máquinas de vectores de soporte. Estos algoritmos son muy interpretables, lo que los hace ideales para aplicaciones donde es crítico entender la contribución de cada característica a las predicciones. (Recuerda: en Linear Learner, las SVM son solo lineales; ver la nota de la sección de Linear Learner.)

**Conoce el algoritmo k-NN y cuándo usarlo.** El algoritmo de k vecinos más cercanos es un algoritmo de ML supervisado simple pero potente para tareas de clasificación y regresión, en el que las predicciones se hacen a partir de los puntos de datos más cercanos en el espacio de características. Es un algoritmo integrado de Amazon SageMaker, especialmente útil cuando la frontera de decisión es compleja y no lineal, y cuando la interpretabilidad es importante, ya que el razonamiento detrás de cada predicción queda claro al examinar los vecinos más cercanos. k-NN se usa mejor con datasets más pequeños, por su complejidad computacional, y en casos donde se entiende con claridad la estructura de los datos y sus relaciones.

**Conoce los algoritmos supervisados de árboles de decisión que ofrece Amazon SageMaker y cuándo usarlos.** Amazon SageMaker ofrece potentes algoritmos supervisados basados en árboles de decisión, como Random Forest y XGBoost. Random Forest es valioso por su capacidad de reducir el sobreajuste y mejorar la precisión creando un ensamble de árboles de decisión y promediando sus predicciones, lo que lo hace adecuado para tareas que involucran un gran número de características de entrada. XGBoost es conocido por su eficiencia computacional y su rendimiento, en particular con datasets grandes y modelos complejos, y se usa a menudo en tareas de clasificación y regresión que requieren alta precisión predictiva y velocidad.

> [!warning] Nota de precisión
> Random Forest no es un algoritmo integrado de SageMaker (se entrena con scikit-learn en un contenedor de framework). Los algoritmos integrados de árboles son XGBoost, LightGBM y CatBoost, además de AutoGluon-Tabular, que combina varios modelos. El concepto de random forest, en cambio, sí puede aparecer en el examen.

**Conoce los algoritmos de clustering que ofrece Amazon SageMaker.** Amazon SageMaker ofrece el algoritmo no supervisado K-Means para clustering, que no debe confundirse con el algoritmo supervisado k-NN, pensado para tareas de clasificación y regresión. La confusión viene de la letra _k_: en K-means es el número de **clusters**; en k-NN, el número de **vecinos**.

**Conoce los algoritmos de reducción de dimensionalidad que ofrece Amazon SageMaker.** Amazon SageMaker ofrece el algoritmo no supervisado Principal Component Analysis para la reducción de dimensionalidad, útil para reducir el número de características de un dataset conservando la mayor variabilidad posible.

**Conoce los algoritmos de modelado de temas que ofrece Amazon SageMaker.** Amazon SageMaker ofrece los algoritmos no supervisados Latent Dirichlet Allocation y Neural Topic Model para casos de uso de modelado de temas, lo que permite a los usuarios descubrir temas ocultos en grandes colecciones de datos textuales sin necesidad de datos etiquetados.

**Conoce los algoritmos de detección de anomalías que ofrece Amazon SageMaker.** Amazon SageMaker ofrece los algoritmos no supervisados Random Cut Forest e IP Insights para casos de uso de detección de anomalías, que son especialmente útiles para identificar patrones o comportamientos inusuales en los datos, como detectar actividades fraudulentas o intrusiones en la red.

**Conoce los algoritmos de análisis textual que ofrece Amazon SageMaker.** Amazon SageMaker ofrece los algoritmos BlazingText y Sequence-to-Sequence para casos de uso de análisis textual, lo que permite manejar eficientemente datos textuales a gran escala y apoya diversas tareas de procesamiento de lenguaje natural, como la clasificación de textos, la traducción y el resumen.

**Conoce los algoritmos de procesamiento de imágenes que ofrece Amazon SageMaker.** Amazon SageMaker ofrece los algoritmos Image Classification, Object Detection y Semantic Segmentation para casos de uso de procesamiento de imágenes, lo que permite a los desarrolladores construir, entrenar y desplegar modelos de machine learning que pueden analizar y entender datos visuales, como reconocer objetos, clasificar imágenes en categorías y segmentar imágenes a nivel de píxel para un análisis detallado.

## Preguntas de repaso (_Review Questions_)

1. ¿Qué servicio de IA de AWS usarías para analizar grandes volúmenes de texto no estructurado y extraer conocimiento como el reconocimiento de entidades y el análisis de sentimiento?
   - A. Amazon Textract
   - B. Amazon Lex
   - C. Amazon Comprehend
   - D. Amazon Polly

2. ¿Qué servicio de IA de AWS está diseñado específicamente para aplicaciones de IA generativa, y ofrece una plataforma robusta para crear texto, imágenes y otros resultados creativos?
   - A. Amazon Rekognition
   - B. Amazon Bedrock
   - C. Amazon Translate
   - D. Amazon Transcribe

3. Para detectar objetos y personas en transmisiones de video en tiempo real, ¿qué servicio de IA de AWS sería el más apropiado?
   - A. Amazon Textract
   - B. Amazon Lex
   - C. Amazon Rekognition
   - D. Amazon Comprehend

   > [!warning] Estado del servicio
   > El análisis de video en streaming de Rekognition ya no admite clientes nuevos desde el 30-04-2026 (ver la sección de Rekognition). Como pregunta de examen basada en el libro, el razonamiento sobre qué servicio analiza video no cambia.

4. ¿Qué servicio de AWS ofrece una solución muy eficiente y escalable para la reducción de dimensionalidad en datasets grandes?
   - A. Amazon Translate
   - B. Amazon Polly
   - C. Principal Component Analysis (PCA) en Amazon SageMaker
   - D. K-Means en Amazon SageMaker

5. ¿Qué algoritmo de Amazon SageMaker es especialmente útil para tareas de clasificación y regresión por su eficiencia y su alto rendimiento?
   - A. Random Forest
   - B. XGBoost
   - C. k-Nearest Neighbors
   - D. Principal Component Analysis

6. ¿Qué algoritmo de ML supervisado que ofrece Amazon SageMaker es ideal para reducir el sobreajuste promediando múltiples árboles de decisión?
   - A. Linear Learner
   - B. BlazingText
   - C. Random Forest
   - D. Latent Dirichlet Allocation

   > [!warning] Nota de precisión
   > La premisa es imprecisa: Random Forest no es un algoritmo integrado de SageMaker (ver la nota de la lista de algoritmos). La pregunta evalúa el concepto de promediar árboles.

7. ¿Para qué tipo de tareas es especialmente adecuado el algoritmo Linear Learner de Amazon SageMaker?
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

11. Para tu problema de ML necesitas un algoritmo no supervisado y muy interpretable en Amazon SageMaker para reducir la dimensionalidad de los datos conservando la máxima varianza. ¿Qué algoritmo integrado usarías?
    - A. K-Means
    - B. Random Cut Forest
    - C. Principal Component Analysis
    - D. Neural Topic Model

12. Para detectar eventos raros y anomalías con alta precisión en flujos de datos, ¿qué algoritmo no supervisado de Amazon SageMaker elegirías?
    - A. Latent Dirichlet Allocation
    - B. IP Insights
    - C. Random Cut Forest
    - D. Factorization Machines

13. Cuando necesitas descubrir temas ocultos en datasets de texto grandes con alta interpretabilidad y precisión, ¿qué algoritmo de Amazon SageMaker seleccionarías?
    - A. K-Means
    - B. Latent Dirichlet Allocation
    - C. Principal Component Analysis
    - D. Random Cut Forest

14. Para agrupar con precisión datasets grandes en grupos predefinidos según la similitud de sus características, ¿qué algoritmo muy interpretable de Amazon SageMaker usarías?
    - A. Random Cut Forest
    - B. Principal Component Analysis
    - C. K-Means
    - D. Neural Topic Model

15. ¿Qué algoritmo de Amazon SageMaker es conocido por su alta precisión y su rendimiento en tareas de clasificación de textos a gran escala, además de ser rentable?
    - A. BlazingText
    - B. Sequence-to-Sequence
    - C. Latent Dirichlet Allocation
    - D. IP Insights

16. Identifica el algoritmo de Amazon SageMaker que ofrece alto rendimiento y precisión en tareas de traducción de textos, además de ser rentable e interpretable.
    - A. Random Cut Forest
    - B. Sequence-to-Sequence
    - C. BlazingText
    - D. Principal Component Analysis

17. Para una identificación y una localización de alta precisión de múltiples objetos dentro de una imagen, ¿qué algoritmo de Amazon SageMaker destaca en rendimiento y eficiencia de costos?
    - A. Image Classification
    - B. Object Detection
    - C. Semantic Segmentation
    - D. Factorization Machines

18. ¿Qué algoritmo de Amazon SageMaker ofrece alta precisión y rentabilidad para clasificar imágenes en categorías predefinidas, asegurando interpretabilidad y rendimiento?
    - A. Latent Dirichlet Allocation
    - B. Image Classification
    - C. Object Detection
    - D. IP Insights

19. Identifica el algoritmo de Amazon SageMaker que permite un análisis detallado de las imágenes a nivel de píxel, ofreciendo alta precisión e interpretabilidad y siendo eficiente en rendimiento.
    - A. Random Cut Forest
    - B. Image Classification
    - C. Semantic Segmentation
    - D. BlazingText

20. ¿Qué algoritmo de Amazon SageMaker utiliza embeddings de palabras para tareas de procesamiento de lenguaje natural, equilibrando precisión, rentabilidad, interpretabilidad y rendimiento?
    - A. Random Cut Forest
    - B. Principal Component Analysis
    - C. Latent Dirichlet Allocation
    - D. BlazingText

> [!note] Sobre la redacción de las preguntas
> Varias preguntas añaden adjetivos como «muy interpretable» o «rentable» a algoritmos de los que no son rasgos distintivos (una red Seq2Seq o una CNN de segmentación no son especialmente interpretables). En estas preguntas, la clave está en la **tarea** (traducir, segmentar píxeles, pronosticar series): los adjetivos no discriminan entre las opciones.

## Glosario

- **Algoritmo integrado (_built-in algorithm_).** Implementación de un algoritmo que AWS empaqueta en un contenedor; tú aportas datos e hiperparámetros y SageMaker entrena.
- **Almacenamiento de archivos.** Carpeta compartida por red que varias máquinas montan a la vez (en AWS: EFS, FSx).
- **Almacenamiento de bloques.** Disco virtual que se conecta a una sola máquina y el sistema operativo usa como disco propio (en AWS: EBS).
- **Almacenamiento de objetos.** Archivos guardados enteros, con una clave, dentro de buckets y accesibles por API web (en AWS: S3).
- **Amazon Connect.** Servicio de centro de contacto de AWS; se integra con Lex para atender llamadas con bots.
- **API.** Contrato para que un programa pida algo a otro; en AWS, peticiones HTTPS con parámetros y respuestas en JSON.
- **Aprendizaje por transferencia (_transfer learning_).** Partir de un modelo ya entrenado con datos genéricos y ajustar solo sus últimas capas con tus datos.
- **Aprovisionar.** Reservar y poner en marcha los recursos (instancias, discos, red) que necesita un trabajo.
- **Árbol de decisión.** Modelo que divide recursivamente los datos por umbrales de las características; sus hojas dan la predicción.
- **ARI (índice de Rand ajustado).** Medida de coincidencia entre dos particiones; 1 es coincidencia perfecta y alrededor de 0, azar.
- **Arranque en frío (_cold start_).** Problema de recomendar a usuarios o ítems nuevos, sin historial del cual aprender.
- **ASR (reconocimiento automático de voz).** Conversión de audio hablado en texto; en AWS, Transcribe.
- **Atención (mecanismo de).** Parte de una red que, al generar cada salida, pondera qué partes de la entrada mirar.
- **Athena (Amazon Athena).** Motor SQL serverless que consulta archivos directamente en S3.
- **Aumento de datos (_data augmentation_).** Generar variantes de los datos de entrenamiento (volteos, reescalados, cambios de color) para robustecer el modelo.
- **Backbone (red base).** CNN preentrenada que extrae las características sobre las que trabaja un detector o segmentador.
- **Backend.** Parte de un sistema que el usuario no ve: bases de datos, reglas de negocio y sistemas internos.
- **Bagging.** Entrenar modelos con muestras bootstrap y promediarlos; reduce la varianza.
- **Base64.** Codificación que representa datos binarios con caracteres de texto para enviarlos dentro de JSON.
- **Bedrock (Amazon Bedrock).** Servicio serverless para usar modelos fundacionales de varios proveedores mediante API.
- **Bedrock Marketplace.** Catálogo de modelos especializados que se despliegan en endpoints administrados y se cobran por instancia.
- **`bedrock-runtime`.** Cliente de boto3 para invocar modelos de Bedrock (el cliente `bedrock` sirve para administrar).
- **BlazingText.** Algoritmo integrado de SageMaker para Word2Vec y clasificación de textos al estilo fastText.
- **Bolsa de palabras (_bag of words_).** Representación de un documento como conteos de palabras, sin orden.
- **Bootstrap.** Muestreo con reemplazo de $n$ observaciones del dataset original.
- **boto3.** SDK de AWS para Python.
- **Búsqueda de rostros (1:N).** Identificar a qué persona de una colección registrada corresponde un rostro.
- **Centro de contacto (_call center_).** Área que atiende a los clientes por teléfono, chat o correo.
- **Centroide.** Media de los puntos de un cluster; representa su centro.
- **Chatbot.** Programa que conversa con personas por texto o voz.
- **Churn (pérdida de clientes).** Cancelación del servicio por parte de un cliente; se modela como clasificación binaria.
- **Clustering (agrupamiento).** Técnica no supervisada que agrupa observaciones similares sin etiquetas.
- **CNN (red neuronal convolucional).** Red para imágenes que aprende filtros de bordes, texturas y partes de objetos.
- **Code Editor.** Aplicación de SageMaker Studio basada en Code-OSS (VS Code de código abierto).
- **Codificador y decodificador.** Partes de una red que, respectivamente, resumen la entrada en vectores y generan la salida a partir de ellos.
- **Comparación de rostros (1:1).** Determinar si dos imágenes muestran a la misma persona; en Rekognition, CompareFaces.
- **Compliance (cumplimiento normativo).** Capacidad de demostrar ante auditores o reguladores que se cumplen las normas aplicables.
- **Comprehend (Amazon Comprehend).** Servicio de NLP que extrae sentimiento, entidades, frases clave e idioma de un texto.
- **Comprehend Medical.** Servicio aparte para extraer entidades médicas (medicamentos, padecimientos) de texto clínico.
- **Conexiones de salto (_skip connections_).** Atajos que llevan información de capas tempranas a capas tardías de una red para recuperar detalle.
- **Contenedor (imagen de).** Paquete (típicamente Docker) con un programa y todas sus dependencias, que corre igual en cualquier máquina.
- **Converse API.** Interfaz de Bedrock con un formato de mensajes común a todos los modelos compatibles.
- **Coocurrencia.** Aparición conjunta de palabras en los mismos documentos o contextos.
- **Corpus.** Colección de documentos de texto que se analiza o con la que se entrena.
- **CSV en SageMaker.** Para algoritmos integrados: sin encabezado, objetivo en la primera columna y tipo de contenido `text/csv`.
- **CTR (tasa de clics).** Proporción de veces que un anuncio mostrado recibe un clic.
- **Cuadro delimitador (_bounding box_).** Rectángulo, dado por coordenadas, que encierra un objeto detectado en una imagen.
- **Cuantización de color.** Reducir los colores de una imagen a $k$ representativos, por ejemplo con K-means.
- **Datos dispersos (_sparse_).** Datos en que la gran mayoría de los valores son cero; se guardan registrando solo los distintos de cero.
- **DBSCAN.** Algoritmo de clustering por densidad que admite clusters de cualquier forma y marca como ruido los puntos aislados.
- **Deep learning (aprendizaje profundo).** Aprendizaje con redes neuronales de muchas capas.
- **DeepAR.** Algoritmo integrado de SageMaker que pronostica muchas series relacionadas con una RNN y da pronósticos probabilísticos.
- **Detección de rostros.** Localizar los rostros de una imagen y estimar sus atributos.
- **Distancias euclidiana, Manhattan y Minkowski.** Medidas de distancia entre vectores; la de Minkowski generaliza a las otras dos ($p = 2$ y $p = 1$).
- **Distribución de Dirichlet.** Distribución sobre vectores de probabilidades que suman uno; es la base de LDA.
- **Dominio de SageMaker.** Unidad de configuración de Studio que agrupa usuarios, permisos, red y almacenamiento.
- **Dropout.** Regularización que apaga al azar una fracción de neuronas en cada paso de entrenamiento.
- **EBS (Amazon Elastic Block Store).** Almacenamiento de bloques: discos virtuales para instancias; es el disco de los espacios del Studio actual.
- **EFS (Amazon Elastic File System).** Sistema de archivos compartido por NFS; era el almacenamiento de Studio Classic.
- **Embedding.** Vector numérico denso que representa un objeto de modo que la cercanía entre vectores refleja similitud.
- **Embeddings contextuales.** Embeddings de oraciones o párrafos que dependen de todo el contexto, a diferencia de Word2Vec.
- **Endpoint (de inferencia).** Servidor administrado que responde predicciones en tiempo real y se cobra por hora mientras exista.
- **Ensamble.** Combinación de muchos modelos (por ejemplo, árboles) para mejorar la predicción.
- **Entidades con nombre.** Menciones de personas, lugares, organizaciones, fechas o cantidades dentro de un texto.
- **Entrenamiento distribuido.** Reparto del entrenamiento entre varias instancias o GPU.
- **EOL (_end of life_).** Fecha a partir de la cual un modelo de Bedrock deja de estar disponible.
- **Épocas (_epochs_).** Pasadas completas del algoritmo sobre los datos de entrenamiento.
- **Error y residuo.** El error $\varepsilon$ es la perturbación no observable del modelo; el residuo $y_i - \hat{y}_i$ es su estimación.
- **Escalabilidad.** Capacidad de seguir funcionando con costo y tiempo razonables cuando crecen los datos o las peticiones.
- **Espacio (_space_) de Studio.** Instancia administrada con su disco donde corre una aplicación de Studio, como Code Editor.
- **Estabilización de resultados parciales.** Opción de Transcribe en streaming que fija las palabras ya transcritas para que no cambien.
- **Estimador (_estimator_).** Objeto del SDK de SageMaker que describe un trabajo de entrenamiento: contenedor, rol, instancias, hiperparámetros y salida.
- **Face Liveness.** Función de Rekognition que verifica, con una video-selfie, que frente a la cámara hay una persona física.
- **Factores latentes.** Variables no observadas que un modelo infiere para describir usuarios, ítems o características.
- **Factorization Machines.** Algoritmo integrado que modela interacciones de segundo orden con vectores latentes; adecuado para datos dispersos.
- **FAISS.** Biblioteca de búsqueda de vecinos más cercanos que usa el k-NN integrado de SageMaker.
- **fastText.** Clasificador de textos rápido que promedia embeddings de palabras y aplica un clasificador lineal.
- **Filtrado basado en contenido.** Recomendación de ítems con atributos parecidos a los que le gustaron al usuario.
- **Filtrado colaborativo.** Recomendación basada en lo que hicieron usuarios con comportamiento similar.
- **Fine-tuning (ajuste fino).** Continuar el entrenamiento de un modelo con tus ejemplos para obtener una versión propia.
- **FSx for Lustre (Amazon FSx for Lustre).** Sistema de archivos paralelo administrado para alimentar entrenamientos intensivos; puede vincularse a S3.
- **Fully managed (totalmente administrado).** Servicio cuya infraestructura opera AWS; el usuario solo lo consume y paga por uso.
- **Función logit.** $\ln(p/(1-p))$: transforma una probabilidad en log-odds.
- **Función sigmoide (logística).** $1/(1+e^{-z})$: transforma cualquier número real en una probabilidad entre 0 y 1.
- **Ganancia de información.** Reducción de la entropía que logra una división de un árbol.
- **Gini (impureza de).** $1 - \sum_c p_c^2$: medida de mezcla de clases en un nodo; vale 0 en un nodo puro.
- **GPU.** Procesador con miles de núcleos pequeños, diseñado para operaciones matriciales masivas; acelera el aprendizaje profundo.
- **Gradient boosting.** Ensamble secuencial en el que cada árbol se ajusta al gradiente de la pérdida (los residuos) de los anteriores.
- **Ground truth (verdad de referencia).** Etiquetas correctas, normalmente hechas por personas, con las que se entrena y evalúa.
- **GuardDuty (Amazon GuardDuty).** Servicio que detecta amenazas contra tu cuenta e infraestructura de AWS.
- **Hiperparámetro.** Ajuste que se fija antes de entrenar, a diferencia de los parámetros que se estiman de los datos.
- **Hiperplano.** Generalización de una recta o un plano a cualquier número de dimensiones.
- **IA generativa.** IA que produce contenido nuevo (texto, imágenes) a partir de patrones aprendidos.
- **IDE (entorno de desarrollo integrado).** Editor de código con explorador de archivos, terminal, depurador y ejecución integrados.
- **Importancia de características.** Medida de cuánto aporta cada característica al modelo; en XGBoost, por número de divisiones (`weight`) o por ganancia (`gain`).
- **Inferencia (en ML).** Uso de un modelo entrenado para producir predicciones sobre datos nuevos; no es la inferencia estadística.
- **Inferencia entre regiones (_cross-region inference_).** Enrutamiento de peticiones de Bedrock a otras regiones mediante perfiles de inferencia.
- **Instancia (tipo de).** Máquina virtual en la nube; su nombre (p. ej., `ml.t3.medium`) indica servicio, familia, generación y tamaño.
- **Intención (_intent_).** En Lex, lo que el usuario quiere lograr, definido con frases de ejemplo.
- **IOPS.** Operaciones de lectura o escritura por segundo que admite un disco.
- **IP (dirección), IPv4 e IPv6.** Número que identifica a un dispositivo en internet; IPv4 son cuatro números de 0 a 255 (p. ej., `203.0.113.45`) e IPv6, el formato más nuevo.
- **IP Insights.** Algoritmo integrado no supervisado que aprende qué direcciones IPv4 usa cada entidad y puntúa pares inusuales.
- **IVR (respuesta de voz interactiva).** Menú telefónico automatizado.
- **JumpStart (Amazon SageMaker JumpStart).** Catálogo de modelos preentrenados y soluciones que se despliegan o ajustan con pocos pasos.
- **K-means.** Algoritmo de clustering que asigna cada punto al centroide más cercano y recalcula centroides hasta converger.
- **k-means++.** Inicialización de K-means que elige centroides iniciales separados entre sí.
- **k-NN (k vecinos más cercanos).** Algoritmo supervisado no paramétrico que predice a partir de los $k$ puntos de entrenamiento más cercanos.
- **Kernel (truco del).** Cálculo de productos internos en un espacio transformado sin construirlo explícitamente; permite SVM no lineales.
- **Lambda (AWS Lambda).** Servicio serverless que ejecuta funciones de código ante eventos y cobra por tiempo de ejecución.
- **Latencia.** Tiempo entre una petición y su respuesta.
- **LDA (asignación latente de Dirichlet).** Modelo generativo de temas: cada documento es una mezcla de temas y cada tema, una mezcla de palabras.
- **Legacy (estado).** Estado de un modelo de Bedrock programado para retirarse: los clientes nuevos ya no pueden usarlo.
- **Lex (Amazon Lex).** Servicio para construir chatbots de voz y texto con intenciones y slots.
- **Linear Learner.** Algoritmo integrado de SageMaker para regresión lineal, clasificación logística y SVM lineal.
- **Log-odds.** Logaritmo de los momios, $\ln(p/(1-p))$; la escala en la que la regresión logística es lineal.
- **Maldición de la dimensionalidad.** Pérdida de significado de las distancias cuando crece el número de dimensiones.
- **Mantenimiento predictivo.** Intervenir un equipo antes de que falle, a partir de señales de sus sensores.
- **Máscara de segmentación.** Imagen del mismo tamaño que la original cuyos píxeles indican su clase.
- **Método del codo.** Heurística que elige el número de clusters donde la WCSS deja de bajar bruscamente.
- **MFA (autenticación multifactor).** Pedir un segundo factor, como un código al celular, además de la contraseña.
- **Mini-batch (minilote).** Grupo de observaciones con el que se estima el gradiente en cada actualización de parámetros.
- **Modelo fundacional (FM).** Modelo muy grande, preentrenado con datos generales, que sirve de base para muchas tareas.
- **Modelo global (series de tiempo).** Un solo modelo ajustado con muchas series a la vez, que comparte información entre ellas.
- **Moderación de contenido.** Detección de contenido inapropiado en imágenes o video.
- **Modo File y modo Pipe.** Formas de entregar datos de S3 a un entrenamiento: copiarlos antes al disco o transmitirlos mientras se entrena.
- **Montar.** Hacer que una carpeta remota aparezca como un directorio más del sistema de archivos local.
- **MSE (error cuadrático medio).** Promedio de los residuos al cuadrado.
- **MTBF (tiempo medio entre fallas).** Métrica de confiabilidad: horas promedio de operación entre fallas.
- **Muestras negativas.** Ejemplos artificiales (pares inventados) con los que un modelo aprende a distinguir lo real de lo aleatorio.
- **Muestreo de reservorio.** Técnica para obtener una muestra aleatoria uniforme de un flujo de datos de tamaño desconocido.
- **Multicolinealidad.** Alta correlación entre predictores; infla la varianza de los coeficientes estimados.
- **Multihilo (_multithreading_).** Reparto del trabajo entre los núcleos de la CPU para ejecutarlo en paralelo.
- **Multimodal.** Que combina tipos de datos distintos, como texto e imagen.
- **MXNet (Apache MXNet).** Biblioteca de aprendizaje profundo retirada en 2023; base de varios algoritmos integrados de SageMaker.
- **NFS (Network File System).** Protocolo estándar en Linux para montar una carpeta remota como si fuera local.
- **Niveles de servicio (_service tiers_) de Bedrock.** Opciones de precio y prioridad bajo demanda: Standard, Priority, Flex y Reserved.
- **NLP (procesamiento de lenguaje natural).** Análisis y generación de texto humano por computadora.
- **NLU (comprensión del lenguaje natural).** Parte del NLP que extrae la intención de una frase.
- **NMT (traducción automática neuronal).** Traducción de oraciones completas con redes neuronales que consideran el contexto.
- **No paramétrico.** Método que no resume los datos en un número fijo de parámetros; k-NN es además «perezoso»: trabaja al predecir.
- **Nova (Amazon Nova).** Familia de modelos fundacionales de Amazon en Bedrock.
- **NTM (modelo neuronal de temas).** Modelado de temas con una red neuronal entrenada por inferencia variacional sobre bolsas de palabras.
- **Object2Vec.** Algoritmo integrado que aprende embeddings a partir de pares de objetos relacionados.
- **Object Detection.** Algoritmo integrado que identifica objetos y los ubica con cuadros delimitadores.
- **OCR (reconocimiento óptico de caracteres).** Tecnología que convierte la imagen de un texto en texto de computadora.
- **Optimizador.** Método que actualiza los pesos de un modelo a partir del gradiente (SGD, Adam, etc.).
- **Óptimo local.** Solución que no puede mejorarse con cambios pequeños, pero no es la mejor posible; K-means solo garantiza uno.
- **Parada temprana (_early stopping_).** Detener el entrenamiento cuando la métrica de validación deja de mejorar.
- **Parámetros de concentración.** Parámetros de la Dirichlet que controlan si las mezclas se concentran en pocos componentes o se reparten.
- **Pares clave-valor.** En un formulario, la etiqueta impresa (clave) y el dato llenado (valor).
- **Perfil de inferencia.** Identificador de Bedrock que define un modelo y las regiones a las que se pueden enrutar sus peticiones.
- **Personalize (Amazon Personalize).** Servicio de IA administrado para recomendaciones personalizadas.
- **Pipeline.** Cadena automatizada de pasos de datos y ML.
- **Poda (_pruning_).** Eliminación de ramas poco útiles de un árbol para reducir el sobreajuste.
- **Polly (Amazon Polly).** Servicio de texto a voz.
- **Pooling.** Capa de una CNN que reduce la resolución, por ejemplo tomando el máximo de cada bloque.
- **Preentrenado.** Modelo que ya fue ajustado con datos masivos por su proveedor; se usa sin entrenarlo.
- **Prompt.** Instrucción o texto de entrada que se da a un modelo fundacional.
- **Pronóstico probabilístico.** Pronóstico que entrega una distribución (o cuantiles) en lugar de un solo valor.
- **Provisioned Throughput.** Capacidad dedicada de Bedrock contratada por un periodo, con descuento por compromiso.
- **Pureza (de un nodo).** Grado en que las observaciones de un nodo pertenecen a una misma clase.
- **RAG (generación aumentada por recuperación).** Insertar en el prompt fragmentos recuperados de tus documentos para que el modelo responda con ellos.
- **RCF (Random Cut Forest).** Algoritmo integrado no supervisado de detección de anomalías basado en árboles de cortes aleatorios.
- **re:Invent.** Conferencia anual de AWS donde suelen anunciarse servicios nuevos.
- **Receta (Personalize).** Algoritmo preconfigurado de Personalize para un tipo de recomendación.
- **RecordIO-protobuf.** Formato binario de SageMaker que empaqueta cada observación como números de 4 bytes; admite datos dispersos.
- **Región (de AWS).** Zona geográfica con sus propios centros de datos, como `us-east-1`.
- **Reglas de asociación.** Técnica para descubrir productos que se compran juntos (análisis de la canasta de compra).
- **Rekognition (Amazon Rekognition).** Servicio de visión por computadora para imágenes y video: objetos, rostros, texto y moderación.
- **Residencia de datos.** Requisito de que los datos se almacenen o procesen dentro de un país o bloque.
- **Resiliencia.** Capacidad de seguir funcionando cuando una parte del sistema falla o se satura.
- **Resultados parciales.** Transcripciones provisionales que Transcribe en streaming envía antes de terminar cada segmento.
- **RNN (red neuronal recurrente).** Red que procesa secuencias paso a paso manteniendo un estado oculto.
- **Rol de IAM.** Identidad con permisos que asumen temporalmente personas o servicios, como SageMaker al leer S3.
- **S3 (Amazon Simple Storage Service).** Almacenamiento de objetos de AWS; fuente habitual de los datos de entrenamiento.
- **SDK (software development kit).** Biblioteca para manejar un servicio desde código, como boto3 o el SDK de SageMaker.
- **Segmentación semántica.** Asignación de una clase a cada píxel de una imagen.
- **Semilla (_seed_).** Valor que fija el generador aleatorio para obtener resultados reproducibles.
- **Separación de hablantes (_speaker diarization_).** Marcar qué fragmentos de un audio dijo cada hablante, sin identificarlos.
- **Seq2Seq (Sequence-to-Sequence).** Algoritmo integrado que transforma una secuencia de tokens en otra (traducción, resumen).
- **Serverless.** Modelo en que no se administran servidores: el servicio asigna cómputo al vuelo y cobra por uso.
- **Similitud coseno.** Coseno del ángulo entre dos vectores; mide cercanía entre embeddings.
- **Skip-gram y CBOW.** Variantes de Word2Vec: predecir el contexto a partir de la palabra, o la palabra a partir del contexto.
- **Slot.** En Lex, dato que el bot debe reunir para cumplir una intención.
- **SSD, R-CNN y YOLO.** Arquitecturas de detección de objetos; R-CNN en dos etapas, SSD y YOLO en una sola pasada.
- **SSML.** Lenguaje de marcado para controlar tono, velocidad, pausas y pronunciación en la síntesis de voz.
- **Streaming (modo).** Procesamiento de datos a medida que llegan por una conexión abierta, en lugar de archivos completos.
- **Studio (Amazon SageMaker Studio).** IDE web de SageMaker para el ciclo de vida de ML.
- **Studio Classic.** Versión anterior de la interfaz de SageMaker Studio, que usaba EFS como almacenamiento.
- **SVM (máquina de vectores de soporte).** Clasificador que busca el hiperplano de margen máximo entre clases.
- **SVR (regresión de vectores de soporte).** Versión de regresión de la SVM, con pérdida ε-insensible.
- **t-SNE.** Técnica no lineal de reducción de dimensionalidad que conserva vecindarios; se usa para visualizar.
- **Tasa de aprendizaje (_learning rate_, `eta`).** Tamaño del paso con que se actualizan los parámetros; en boosting, fracción de la corrección que aporta cada árbol.
- **Tasa de conversión.** Porcentaje de visitas que terminan en compra.
- **Textract (Amazon Textract).** Servicio que extrae texto, formularios y tablas de documentos, conservando su estructura.
- **Throughput.** Cantidad de trabajo completado por unidad de tiempo: peticiones o tokens por minuto, o megabytes por segundo de un disco.
- **Tipo MIME.** Etiqueta estándar del formato de un contenido, como `application/json`.
- **Token.** Unidad en que se divide un texto (suele ser un fragmento de palabra); es la unidad de cobro de los modelos de texto en Bedrock.
- **Trabajo de entrenamiento (_training job_).** Ejecución administrada de un entrenamiento en SageMaker, que se cobra por segundo.
- **Transcribe (Amazon Transcribe).** Servicio de reconocimiento automático de voz, por lotes o en streaming.
- **Transformación por lotes (_batch transform_).** Trabajo que puntúa un archivo completo de S3 y se apaga al terminar.
- **Translate (Amazon Translate).** Servicio de traducción automática neuronal.
- **TTS (texto a voz).** Conversión de texto escrito en audio hablado; en AWS, Polly.
- **U-Net.** Arquitectura de segmentación popular en imágenes médicas; no es una opción del algoritmo integrado de SageMaker.
- **Upsampling (sobremuestreo espacial).** Ampliación de la resolución de los mapas internos de una red para producir una salida por píxel.
- **Vectores de soporte.** Observaciones más cercanas al hiperplano de una SVM; solo ellas determinan la solución.
- **Vectores propios (de la covarianza).** Direcciones de los componentes principales; su valor propio es la varianza que explica cada uno.
- **Vocabulario personalizado.** Lista de palabras que se entrega a Transcribe para que reconozca términos propios.
- **WAF Fraud Control ATP.** Reglas administradas de AWS WAF que detectan credenciales robadas y ataques de volumen en los inicios de sesión.
- **WCSS (suma de cuadrados dentro de los clusters).** Suma de las distancias al cuadrado de cada punto a su centroide; en scikit-learn, `inertia_`.
- **Word2Vec.** Técnica que aprende un embedding por palabra a partir de las palabras que la rodean.
- **XGBoost.** Implementación eficiente y regularizada de gradient boosting con árboles; algoritmo integrado de SageMaker.
