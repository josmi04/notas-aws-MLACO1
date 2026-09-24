---
tema: "Capítulo 3 — Transformación de datos e ingeniería de características (versión explicada)"
fuente: Guia Oficial/03_data_transformation_and_feature_engineering.txt
guia-mla-c01:
  [
    Dominio 1,
    1.2 Transform data and perform feature engineering,
    1.3 Ensure data integrity and prepare data for modeling,
  ]
perfil-lector: profesional de datos/estadística sin experiencia administrando infraestructura
versiones-codigo:
  python: "3.12.6"
  scikit-learn: "1.5.2"
  pandas: "2.2.2"
  numpy: "2.0.2"
  scipy: "1.14.1"
  category_encoders: "2.11.1"
verificado: 2026-09-23
tags:
  [
    aws,
    mla-c01,
    feature-engineering,
    limpieza-de-datos,
    sagemaker,
    feature-store,
    data-wrangler,
    glue-databrew,
    ground-truth,
    clarify,
    lake-formation,
    s3,
  ]
---

> [!info] Cómo leer esta versión
>
> - **Qué es.** Traducción íntegra al español del capítulo 3 de la guía oficial de estudio, con los mismos encabezados, ejemplos y afirmaciones. No se eliminó nada. Los encabezados conservan entre paréntesis el título original en inglés, porque el examen se presenta en inglés.
> - **Qué se añadió.** Explicaciones de los términos de AWS y de infraestructura integradas en el texto, aclaraciones de razonamientos que el libro da por sentados, **notas de precisión** (recuadros amarillos) cuando el original es impreciso o está desactualizado, las salidas reales del código, la sección «Escenarios donde este servicio es la opción obligada» y un glosario al final.
> - **Figuras.** El `.txt` no incluye las imágenes, así que solo quedan los pies de figura. Cuando la figura mostraba la salida de un programa, aquí va la salida real obtenida al ejecutar ese mismo código con las versiones que indica el encabezado del archivo.
> - **Código.** Se corrigieron defectos de la extracción del PDF (comillas tipográficas `‘ ’ “ ”` en lugar de comillas rectas, el signo menos tipográfico `−` y líneas partidas a la mitad) para que el código se pueda ejecutar. La lógica no se modificó.
> - **Fórmulas.** En el `.txt` se perdieron varias fórmulas, que aparecen como huecos. Se reconstruyeron a partir del contexto y se indica cuando la reconstrucción implica una suposición.
> - **Cifras y estado de los servicios.** Se verificaron en la documentación de AWS el 23 de septiembre de 2026. Cambian con frecuencia, así que confírmalos en la documentación oficial vigente antes de usarlos en una decisión real.

> [!warning] Cambios en AWS posteriores a la edición del libro (verificado el 23-09-2026)
>
> - **Amazon SageMaker** pasó a llamarse **Amazon SageMaker AI** (el nombre «Amazon SageMaker» designa ahora una plataforma más amplia de datos y analítica). En este texto se usa el nombre del libro.
> - **SageMaker Data Wrangler** se integró en **Amazon SageMaker Canvas**, el entorno visual de SageMaker AI. La versión clásica dentro de _Studio Classic_ sigue documentada.
> - **Amazon SageMaker Ground Truth** y **Amazon SageMaker Clarify** ya no admiten clientes nuevos. Los clientes existentes pueden seguir usándolos, pero AWS no planea añadirles funciones. Para Clarify, AWS recomienda calcular las mismas métricas de sesgo con pandas y scikit-learn, y usar la biblioteca SHAP para la explicabilidad. Las preguntas de examen basadas en el libro pueden seguir mencionándolos, así que conviene conocerlos igual.
> - **Amazon Mechanical Turk**, la fuerza de trabajo pública que el libro menciona para el etiquetado, cierra definitivamente el **30 de septiembre de 2026**.

**Capítulo 3**

# Transformación de datos e ingeniería de características (_Data Transformation and Feature Engineering_)

> LOS OBJETIVOS DEL EXAMEN AWS CERTIFIED MACHINE LEARNING (ML) ENGINEER – ASSOCIATE QUE CUBRE ESTE CAPÍTULO PUEDEN INCLUIR, ENTRE OTROS, LOS SIGUIENTES:
>
> ✔ Dominio 1: Preparación de datos para machine learning
>
> - 1.2 Transformar datos y realizar ingeniería de características
> - 1.3 Asegurar la integridad de los datos y prepararlos para el modelado

## Introducción (_Introduction_)

En el capítulo 2 aprendiste a ingerir datos de distintas fuentes y a almacenarlos en AWS. _Ingerir_ significa traer los datos desde donde se generan (bases de datos, sensores, aplicaciones) hasta la plataforma donde se van a procesar. Ahora que ya recolectaste los datos y los resguardaste en AWS, tu trabajo como ingeniero de machine learning (ML) es dejarlos listos para entrenar tu modelo. Este paso corresponde a la fase _Process Data_ (procesar datos) del ciclo de vida de ML, como se muestra en la Figura 3.1.

_Figura 3.1 El ciclo de vida de ML._

Tus datos suelen estar almacenados en su estado crudo en un **_data lake_** (lago de datos). Un data lake es un repositorio central donde los archivos se guardan tal como llegan (CSV, JSON, Parquet, imágenes, registros de aplicaciones) sin exigirles un esquema al escribirlos. El esquema se aplica recién al leerlos, lo que se conoce como _schema-on-read_. Se contrapone al _data warehouse_ (almacén de datos), que exige tablas con un esquema definido antes de cargar nada (_schema-on-write_), como una base de datos analítica. Para ML la diferencia importa: muchos proyectos usan datos que no son tablas (imágenes, texto), y conservar el dato crudo permite rehacer la preparación cuando cambia el modelo. Una arquitectura de data lake puede proporcionar una base sólida para construir una solución de ML, porque está diseñada para almacenar cantidades masivas de datos en un repositorio central, de modo que estén disponibles para que distintos grupos de tu organización los categoricen, procesen, enriquezcan y consuman.

Las siguientes son las características clave de un data lake:

- **Almacenamiento a escala.** Un data lake debe poder acomodar datos que llegan en cualquier momento, ya sea a intervalos predefinidos o en tiempo real, en cargas pequeñas o en grandes lotes. En la jerga, una carga pequeña es un **_payload_**, el contenido útil de un envío individual, como el evento JSON de pocos kilobytes que manda un sensor. Un lote (**_batch_**) es un bloque grande que se carga de una vez, como el volcado nocturno de una base de datos. El almacenamiento tiene que absorber ambos patrones sin que haya que rediseñarlo.
- **Movimiento de datos.** Igual que un lago real tiene un río de entrada que trae el agua y un río de salida que se la lleva, un data lake debe permitir que los datos entren y salgan de él.
- **Datos crudos.** Los datos que llegan al data lake pueden provenir de fuentes distintas y ser estructurados, semiestructurados o no estructurados. El formato de los datos es irrelevante durante las fases de ingesta y almacenamiento de la recolección de datos. Es irrelevante precisamente por el _schema-on-read_: las decisiones sobre el formato se posponen a la fase de procesamiento, que es el tema de este capítulo.
- **Seguridad.** Como al data lake pueden llegar datos sensibles, es crítico que ningún usuario no autorizado tenga acceso a ellos. Por eso el data lake debe protegerse con controles robustos de gestión de identidades y accesos (**IAM**, _identity and access management_). En AWS, IAM es el servicio donde se define quién puede hacer qué acción sobre qué recurso. Ese «quién» son usuarios, grupos y _roles_, es decir, identidades que asumen temporalmente personas o servicios, y los permisos se escriben en políticas en formato JSON. Se parece a los permisos de una base de datos (`GRANT SELECT ON tabla TO usuario`), pero se aplica a toda la cuenta de AWS; sin él, una sola credencial filtrada podría leer el lago completo. El cifrado en uso y en tránsito no es menos importante que el cifrado en reposo. El **cifrado en reposo** protege los datos mientras están guardados: quien obtenga el disco o una copia del archivo no puede leerlos sin la clave. El **cifrado en tránsito** los protege mientras viajan por la red, normalmente con TLS, el mismo protocolo que pone el candado de HTTPS en el navegador, y evita que alguien que intercepte el tráfico los lea. El **cifrado en uso** los protege mientras se procesan en memoria. Es el más difícil de lograr y requiere técnicas especiales, como entornos de cómputo aislados (_enclaves_). Por lo tanto, deben establecerse y hacerse cumplir barreras de seguridad (**_guardrails_**) que protejan los datos mientras se producen y se consumen. Los guardrails son reglas preventivas o de detección que acotan lo que cualquiera puede hacer, como «ningún bucket puede ser público» o «todo objeto debe estar cifrado». Igual que las barandillas de una carretera, no dirigen el trabajo diario: impiden salirse del camino.
- **Catálogo.** Los data lakes deben permitirte almacenar datos relacionales (p. ej., bases de datos operacionales o datos de aplicaciones de línea de negocio) y no relacionales (p. ej., datos de aplicaciones móviles, dispositivos IoT y redes sociales). Una _base de datos operacional_ es la que usa una aplicación en su día a día para registrar transacciones (pedidos, pagos, altas de clientes), a diferencia de las bases pensadas para análisis. Las _aplicaciones de línea de negocio_ (_line-of-business_, LOB) son las que sostienen los procesos centrales de la empresa: el ERP, el CRM o la nómina. Los data lakes también te dan la capacidad de entender qué datos hay en el lago mediante el rastreo (_crawling_), la catalogación y la indexación de los datos. Sin un catálogo, un lago con millones de archivos se convierte en un «pantano de datos» donde nadie sabe qué hay ni dónde. Rastrear significa que un proceso automático (en AWS, un **_crawler_** de AWS Glue) recorre los archivos, infiere su esquema (columnas, tipos de dato, particiones) y lo registra como tablas en un catálogo de metadatos. Indexar es organizar esos metadatos para que se puedan buscar.

Para crear un data lake en AWS, normalmente usarías una combinación de los siguientes servicios: Amazon S3 (para almacenamiento), AWS Glue (para catalogación), Amazon Athena (para consultas) y AWS Lake Formation (para la gestión central de IAM, la seguridad de los datos y la gobernanza).

Para entender por qué S3 ocupa el lugar del almacenamiento, conviene distinguir las tres familias de almacenamiento en la nube:

- **Almacenamiento de bloques.** Es un disco virtual que se conecta a una máquina, como el SSD de tu laptop. El sistema operativo lo formatea y lo usa como disco propio. En AWS es Amazon EBS.
- **Almacenamiento de archivos.** Es una carpeta compartida por red que varias máquinas «montan» a la vez, como la unidad de red de una oficina. En AWS son Amazon EFS y la familia Amazon FSx.
- **Almacenamiento de objetos.** Cada archivo se guarda entero como un _objeto_, junto con sus metadatos, identificado por una clave (algo como `ventas/2025/enero.parquet`) dentro de un contenedor llamado **_bucket_**. Se lee y se escribe mediante una API web, no montándolo como un disco. No se modifica un fragmento de un objeto: se reemplaza el objeto completo.

**Amazon S3** (_Simple Storage Service_) es el almacenamiento de objetos de AWS. Escala prácticamente sin límite y es la opción más barata por gigabyte de las tres, y por eso es la base natural de un data lake. **AWS Glue** es un servicio de integración de datos. La pieza que importa aquí es su _Data Catalog_, donde se registran las definiciones de tabla: el esquema y la ubicación de los archivos en S3. **Amazon Athena** es un motor SQL que consulta directamente los archivos de S3 usando esas definiciones. Se paga por los datos que escanea cada consulta y no hay servidor ni base de datos que administrar, lo que en AWS se llama **_serverless_**: el servicio asigna cómputo al vuelo y cobra por uso. **AWS Lake Formation** es una capa de gobernanza sobre el catálogo que permite conceder permisos a nivel de base de datos, tabla, columna y fila desde un único lugar, en vez de combinar a mano políticas de IAM y de bucket. La _gobernanza_ es el conjunto de reglas y procesos que determinan quién puede usar qué datos, para qué y cómo se audita.

Con excepción de AWS Lake Formation, ya cubrimos estos servicios en el capítulo anterior. Para el examen, necesitas saber que Amazon S3 es la opción de almacenamiento preferida para un data lake en AWS. Ofrece almacenamiento altamente durable (99.999999999 %), altamente disponible y seguro, y se integra de forma transparente con varios servicios de procesamiento de datos y plataformas de ML de AWS. La **durabilidad** es la probabilidad de no perder un dato guardado. La **disponibilidad** es la probabilidad de poder acceder a él en un momento dado; un servicio puede estar momentáneamente inaccesible sin haber perdido nada. Los «once nueves» de durabilidad son difíciles de imaginar, y AWS los ilustra así: si guardas 10 millones de objetos, en promedio esperarías perder uno cada 10 000 años. Como referencia de orden de magnitud, las estadísticas públicas de grandes flotas de discos duros muestran tasas de falla anual del orden del 1 % por disco. S3 alcanza su cifra replicando cada objeto en varios dispositivos repartidos entre distintas **zonas de disponibilidad** (uno o más centros de datos físicamente separados dentro de una misma región). Amazon S3 puede usarse como almacenamiento de fuente única de verdad (**_single source of truth_**) para la mayoría de los servicios de ML de AWS. Esto significa que existe una sola copia autorizada de los datos a la que apuntan todos los servicios, en lugar de copias que divergen en cada herramienta, y así se evita que un equipo entrene con una versión de los datos y otro equipo con otra. Además, con la clase de almacenamiento S3 Intelligent-Tiering puedes reducir el costo de almacenamiento dejando que AWS determine automáticamente cuándo mover los datos a la clase de almacenamiento más adecuada. Las **_clases de almacenamiento_** de S3 son niveles de precio según la frecuencia de acceso: S3 Standard (acceso frecuente, más caro por GB), S3 Standard-IA (acceso infrecuente, más barato por GB pero con cargo por recuperar) y las clases Glacier (archivo: muy baratas, con recuperación más lenta o más cara). **Intelligent-Tiering** observa el patrón de acceso de cada objeto y lo mueve entre niveles sin que cambie la forma de leerlo. Esto encaja con ML, porque los datos crudos se leen intensamente al principio de un proyecto y después casi nunca.

> [!warning] Nota de precisión: S3 (verifica las cifras en la documentación vigente)
>
> - El 99.999999999 % es el **objetivo de diseño** de durabilidad que AWS publica para S3, no una garantía contractual. El acuerdo de nivel de servicio (SLA) de S3 cubre la disponibilidad, no la durabilidad. Las clases de una sola zona (p. ej., S3 One Zone-IA) guardan los datos en una única zona de disponibilidad, por lo que no resisten la pérdida de esa zona.
> - Intelligent-Tiering cobra una pequeña tarifa mensual de monitoreo por objeto, y los objetos de menos de 128 KB no se mueven automáticamente entre niveles. En datasets formados por millones de archivos diminutos, el ahorro puede no compensar.

AWS Lake Formation complementa muy bien los servicios mencionados: centraliza los permisos sobre los datos, simplifica la gestión de la seguridad y la gobernanza a escala, monitorea el acceso a los datos y ayuda a asegurar el cumplimiento normativo (**_compliance_**). Compliance significa poder demostrar ante auditores o reguladores que se cumplen normas como el RGPD europeo, la HIPAA estadounidense para datos de salud o la regulación bancaria. Lake Formation ayuda porque los permisos quedan declarados en un solo lugar y los accesos quedan registrados de forma auditable.

En las próximas secciones aprenderás las técnicas que te permiten procesar los datos de tu data lake y volverlos aptos para entrenar tu modelo de ML, de modo que produzca inferencias significativas. Una advertencia de vocabulario importante para quien viene de la estadística: en la jerga de ML y de AWS, **_inferencia_** no es la inferencia estadística (estimar parámetros, intervalos de confianza o pruebas de hipótesis). Significa **usar un modelo ya entrenado para producir predicciones sobre datos nuevos**. Cuando AWS habla de un «endpoint de inferencia» o de «inferencia por lotes», se refiere a servir predicciones.

El objetivo último de la fase de procesamiento de datos es producir datos de calidad para entrenar eficazmente tu modelo de ML, de modo que aprenda más rápido de tus datos y produzca predicciones precisas. Cuanto mayor sea la calidad de tu conjunto de datos de entrenamiento, más precisas serán las inferencias que produzca tu modelo de ML.

## Comprender la ingeniería de características (_Understanding Feature Engineering_)

Antes de profundizar en la ingeniería de características, centrémonos en entender los distintos tipos de datos:

- **Datos categóricos.** Los datos categóricos contienen un número finito de categorías distintas, cada una representada con una cadena de texto. Pueden tener un orden lógico, como las tallas de una camisa: Small, Medium, Large, X-Large. A este tipo de dato categórico se le llama **ordinal**. Si no tiene un orden lógico, se le llama **nominal**. Por ejemplo, los 50 estados de EE. UU. son datos categóricos nominales.
- **Datos numéricos.** En ML, los datos numéricos son cualquier tipo de dato que pueda representarse con números. Esto incluye datos discretos y datos continuos.
  - Los **datos discretos** tienen una cantidad contable de valores entre dos valores cualesquiera. Una variable discreta siempre es numérica, como el número de quejas de clientes o el número de fallas o defectos.
  - Los **datos continuos** tienen una cantidad infinita de valores entre dos valores cualesquiera. Una variable continua puede ser numérica o de fecha y hora, como la longitud de una pieza o la fecha y hora en que se recibe un pago.
- **Datos textuales.** Los datos textuales son contenido escrito que puede procesarse y analizarse: desde oraciones y párrafos de artículos o libros hasta publicaciones en redes sociales, reseñas y más. Se usan en diversas tareas de procesamiento de lenguaje natural (NLP), como el análisis de sentimiento, la traducción de idiomas y la clasificación de textos.
- **Datos de imagen.** Los datos de imagen consisten en valores de píxeles que pueden analizarse para extraer características como bordes, formas, colores y patrones. Se usan a menudo en tareas de visión por computadora, como la detección de objetos, el reconocimiento facial y la clasificación de imágenes.
- **Datos de series de tiempo.** Los datos de series de tiempo son una colección de observaciones o mediciones registradas a intervalos regulares. Cada observación está asociada a una marca de tiempo (_timestamp_) o periodo específico, lo que forma una secuencia de datos ordenada cronológicamente. El orden de los datos es crucial para entender las tendencias, los patrones y las variaciones que ocurren a lo largo de ese periodo.

> [!warning] Nota de precisión: categorías representadas con números
> El libro dice que cada categoría «se representa con una cadena». En la práctica, muchas variables categóricas vienen codificadas con números: códigos postales, identificadores de producto, códigos de sucursal. Siguen siendo categóricas y deben tratarse como tales. Si se dejan como números, el modelo supondría que el código postal 90210 es «mayor» que el 10001. Esto es especialmente relevante al elegir la codificación (ver «Ingeniería de características para datos categóricos»).

Entender los distintos tipos de datos es el primer paso hacia nuestro objetivo de entrenar un modelo de ML. Este paso es necesario porque los datos de tu data lake combinan múltiples datasets ingeridos desde fuentes distintas. Cada dataset puede estar compuesto por datos estructurados, semiestructurados o no estructurados. E incluso si todos tus datasets fueran estructurados, no está garantizado que todos compartan el mismo esquema. Por eso estos datos necesitan transformarse adecuadamente antes de alimentar a un algoritmo de ML.

Aquí es donde entra la ingeniería de características (**_feature engineering_**). La ingeniería de características es la ciencia (y el arte) de extraer más información de los datos existentes para mejorar el poder predictivo de tu modelo de ML y ayudarlo a aprender más rápido. Durante la ingeniería de características no agregas datos nuevos, sino que haces más útiles los datos que ya tienes. Cuando el libro dice que «no agregas datos nuevos», quiere decir que no sumas observaciones ni fuentes externas: derivas variables nuevas a partir de las existentes. En términos estadísticos equivale a construir covariables derivadas, transformaciones o términos de interacción antes de ajustar un modelo de regresión. Al reorganizar tus datos en un conjunto de características que pueda alimentar directamente a un algoritmo de ML, tu modelo producirá mejores inferencias. La ingeniería de características suele apoyarse en el conocimiento del dominio de los datos para que tu modelo de ML produzca resultados más eficaces.

Los tipos de datos que acabas de conocer (categóricos, numéricos, textuales, de imagen y de series de tiempo) determinarán el enfoque de ingeniería de características más apropiado.

### Definir las características (_Defining Features_)

En ML, las **características** (_features_) son propiedades o atributos individuales y medibles de los datos que analizas. Cada atributo único de los datos se considera una característica (también llamada _atributo_). Piensa en ellas como las entradas que tu modelo usa para hacer predicciones. En vocabulario estadístico, las características son las variables explicativas, covariables o predictores, y lo que el modelo predice (la **variable objetivo**, _target_ o _label_) es la variable respuesta. Por ejemplo, en un dataset de ventas, las características podrían incluir la fecha, el número de visitantes y el número de pedidos (ventas).

La Figura 3.2 ilustra un dataset simplificado de datos de ventas.

_Figura 3.2 Ejemplo de un dataset._

Supón que tu modelo de ML usará estos datos para predecir las ventas de un día dado. A primera vista, habrás notado que las dos primeras filas tienen un número de ventas notablemente mayor que las tres restantes. Un análisis más detallado indicó que los clientes tienden más a comprar los fines de semana. Por lo tanto, es el día de la semana lo que influye en los hábitos de compra. Así que podemos construir una característica que indique el día de la semana y luego escribir un script sencillo que complete ese dato automáticamente, como se muestra en la Figura 3.3. El original usa aquí el verbo _impute_, pero no en el sentido estadístico de rellenar valores faltantes: se refiere a **calcular la columna nueva** a partir de la fecha, por ejemplo con `df["dia_semana"] = df["fecha"].dt.day_name()` en pandas.

_Figura 3.3 Agregar una característica a un dataset._

Este ejemplo sencillo muestra cómo la información «oculta» en tu dataset puede ayudar a tu modelo de ML a aprender más rápido. La ingeniería de características consiste en descubrir dónde está esta información y cómo transformar tu dataset para aprovecharla al máximo.

### Seleccionar características para el entrenamiento del modelo (_Selecting Features for Model Training_)

Las siguientes son las principales áreas de enfoque de la ingeniería de características:

- Extracción de características (_feature extraction_)
- Selección de características (_feature selection_)
- Creación y transformación de características (_feature creation and transformation_)

El objetivo de la extracción y la selección de características es reducir la **dimensionalidad** de tu dataset. El término dimensionalidad indica el número de características (o entradas) de tu dataset. Cuanto mayor es la dimensionalidad de un dataset, más difícil resulta entrenar eficazmente tu modelo de ML. A los modelos les cuesta encontrar los patrones que quieres que reconozcan cuando hay que examinar muchas dimensiones distintas de los datos (muchas características). El razonamiento detrás de esta afirmación es la conocida «maldición de la dimensionalidad»: con más dimensiones, las observaciones quedan más dispersas en el espacio de características. Se necesitan muchas más observaciones para cubrirlo, crece el riesgo de sobreajuste y aumenta el costo de cómputo.

Por eso son importantes la extracción y la selección de características.

#### Extracción de características (_Feature Extraction_)

La extracción de características es el proceso de reducir automáticamente la dimensionalidad de tu dataset creando características nuevas a partir de las existentes. Es un proceso común en datasets con una gran cantidad de características, y aparece sobre todo cuando se trabaja con datos de imagen, audio o texto.

La Figura 3.4 muestra un ejemplo de reconocimiento de imágenes.

_Figura 3.4 Ejemplo de reconocimiento de imágenes._

Antes de la llegada de las redes neuronales, una de las formas de analizar datasets de imágenes era extraer características de cada imagen. Si la imagen es un auto, extraes algunos de sus aspectos útiles (el parabrisas, los faros, las direccionales y las llantas) como características independientes. Así, en lugar de que tu dataset esté formado por píxeles crudos, tienes características o columnas como `windshield_present` y `headlight_present`, como se muestra en la Figura 3.5. Estas características facilitarán que el algoritmo de ML aprenda de los datos de imagen y, con el tiempo, empiece a reconocer rostros.

_Figura 3.5 Extracción de características de una imagen._

> [!warning] Nota de precisión: ¿autos o rostros?
> El ejemplo trata de reconocer **autos** (parabrisas, faros, llantas), pero el original cierra diciendo que el algoritmo «empezará a reconocer rostros». Parece un resto de otro ejemplo. La idea correcta es que el algoritmo empezará a reconocer autos.

En la mayoría de los casos, los propios datos te ayudarán a determinar qué técnica específica de extracción de características usar.

Para datos de imagen, podría ser extraer rasgos clave como los que vimos antes. En NLP, podría ser extraer características útiles como las palabras más frecuentes del texto, sin contar artículos ni preposiciones.

#### Selección de características (_Feature Selection_)

La selección de características es otra técnica para reducir la dimensionalidad de tu dataset y se usa con frecuencia junto con la extracción de características.

La selección de características clasifica las características existentes del dataset según su importancia predictiva y se queda solo con las más relevantes según esa clasificación.

Como tu dataset contiene datos crudos almacenados en un data lake, es probable que algunas características sean más importantes que otras para la precisión del modelo. Algunas también serán redundantes por estar correlacionadas con otras características. En el ejemplo de la Figura 3.6, los ingresos (_revenue_) se mueven en paralelo a las ventas, así que es poco probable que aporten al modelo mucha más información de la que ya obtiene de los datos de ventas.

_Figura 3.6 Selección de características._

Hay que eliminar las características irrelevantes para el problema. La selección de características resuelve estos problemas filtrando del dataset las características irrelevantes o redundantes. Como resultado, el algoritmo de ML solo recibe un subconjunto de las características más útiles para el problema.

Los algoritmos de filtrado pueden usar una medida estadística para identificar las características que tienen una relación fuerte con la variable objetivo. Estos son los llamados métodos de filtro, que usan por ejemplo la correlación, la información mutua o la prueba χ². En el ejemplo, la variable objetivo que queremos que prediga el modelo es la venta de un día dado, así que las utilidades netas de las ventas probablemente no son relevantes para esa predicción. Por lo tanto, podemos eliminar esa característica.

> [!warning] Nota de precisión: redundancia frente a fuga de información
> El texto justifica quitar los ingresos o las utilidades por **redundancia** o **irrelevancia**. Hay una razón más fuerte que el libro no menciona: si la variable objetivo son las ventas del día, los ingresos o utilidades de ese mismo día **se conocen solo después** de que ocurren esas ventas. Usarlos como predictores sería **fuga de información del objetivo** (_target leakage_). El modelo parecería excelente en la evaluación y fallaría en producción, donde ese dato no existe todavía al momento de predecir. El concepto de fuga de datos (_data leakage_) se trata más adelante en este capítulo.

Recuerda que los algoritmos de ML no se usan solo con los datasets estructurados típicos. A menudo trabajamos con datasets no estructurados, en forma de imágenes, audio o video. Estos formatos requieren técnicas de filtrado para reducir la dimensionalidad del dataset. Por ejemplo, conservar solo ciertos coeficientes de frecuencia de una señal de audio o reducir la resolución de una imagen.

#### Creación y transformación de características (_Feature Creation and Transformation_)

A diferencia de la extracción y la selección de características, la creación y transformación de características no es una técnica de reducción de dimensionalidad. Es el proceso de generar características nuevas a partir de las existentes. Por ejemplo, supongamos que tenemos «fecha» como característica, con formato de día, mes y año de dos dígitos cada uno (dd-mm-aa).

Podrías descubrir que combinar día, mes y año en una sola característica no ayuda mucho a tus predicciones. En su lugar, podrías generar tres características distintas, una para el día, otra para el mes y otra para el año. Así podrías descubrir una relación significativa entre alguna de ellas y la variable objetivo que quieres que prediga tu modelo de ML.

### Uso de Amazon SageMaker Feature Store (_Using Amazon SageMaker Feature Store_)

Como las características son las entradas (o variables) de los modelos de ML durante el entrenamiento, ¿no sería útil tener un lugar central donde seleccionar, refinar, almacenar y gestionar todas las características de tu dataset?

Aquí es donde entra Amazon SageMaker Feature Store. **Amazon SageMaker** es la plataforma de ML de AWS: agrupa entornos de notebooks, herramientas de preparación de datos, entrenamiento, despliegue y monitoreo de modelos (hoy se llama Amazon SageMaker AI; ver el recuadro del inicio). Amazon SageMaker Feature Store es un repositorio **_fully managed_** (totalmente administrado) para almacenar, compartir y gestionar características de modelos de ML. Que un servicio sea _fully managed_ significa que AWS opera los servidores, el almacenamiento, la replicación, las actualizaciones de seguridad y el escalado. Tú solo usas el servicio desde la consola web, la API o el SDK y pagas por uso, sin acceso a las máquinas subyacentes. Es la diferencia entre instalar PostgreSQL en tu propio servidor y contratar una base de datos que otro mantiene. Por ejemplo, en una aplicación que recomienda libros sobre un tema dado, las características podrían incluir la afinidad del libro con el tema, las calificaciones del libro y su fecha de publicación.

## Amazon SageMaker Feature Store

Un modelo de ML recibe como entrada **características** (_features_). Por ejemplo, para detectar fraude en una transacción podríamos usar el importe de la compra, el saldo disponible, el número de compras realizadas en las últimas 24 horas o el gasto promedio del cliente durante el último mes. Estas características cambian con el tiempo. Si un cliente hace una nueva compra, por ejemplo, `numero_compras_24h` puede pasar de 7 a 8.

Amazon SageMaker Feature Store sirve para **almacenar y recuperar estas características** de forma que puedan reutilizarse tanto durante el entrenamiento como durante la inferencia.

Las características relacionadas se organizan en **feature groups**, que pueden imaginarse aproximadamente como tablas:

| customer_id | saldo | compras_24h | gasto_medio_30d | event_time       |
| ----------- | ----: | ----------: | --------------: | ---------------- |
| 123         | 8,400 |           8 |             620 | 2026-09-23 14:03 |

Aquí, `customer_id` identifica al cliente; `saldo`, `compras_24h` y `gasto_medio_30d` son las características; y `event_time` indica **cuándo esos valores eran válidos o fueron observados**. No representa necesariamente el momento de una predicción. Si una compra realizada a las 14:03 hace que `compras_24h` cambie de 7 a 8, ese nuevo valor puede quedar asociado a `event_time = 14:03`.

Cada feature group puede utilizar dos almacenes. El **online store** conserva los valores más recientes y está optimizado para lecturas de muy baja latencia. Su uso principal es la **inferencia en tiempo real**. Si llega una nueva transacción, la aplicación puede consultar rápidamente las características actuales del cliente y pasarlas al modelo:

```text
Nueva transacción
→ consultar features actuales
→ Online Store
→ modelo
→ predicción
```

Una consulta analítica sobre S3 mediante Athena sería demasiado lenta para este caso, porque puede tardar segundos, mientras que una aplicación en tiempo real suele necesitar respuestas en milisegundos.

El **offline store**, en cambio, conserva el historial de las características. Si `compras_24h` fue cambiando durante el día:

```text
10:00 → 5
12:00 → 6
13:20 → 7
14:03 → 8
```

el online store necesita principalmente el valor actual, `8`, mientras que el offline store puede conservar todos los registros históricos. Ese historial se almacena en S3 y se utiliza para análisis, entrenamiento e inferencia por lotes.

La diferencia esencial es:

```text
Online Store  → valor actual → inferencia en tiempo real
Offline Store → historial    → entrenamiento, análisis y batch inference
```

El historial del offline store es especialmente importante para construir datasets de entrenamiento correctamente. Supongamos que queremos recrear una transacción ocurrida ayer a las 12:00. Si en ese momento `compras_24h = 6`, debemos usar ese valor, aunque hoy la misma característica valga `15`. Utilizar el valor actual introduciría información que todavía no existía cuando ocurrió la observación, produciendo **data leakage**.

Como el offline store conserva la `event_time`, puede reconstruirse qué valor tenía una característica en un instante concreto. Esto se conoce como una consulta **point-in-time**.

Feature Store también ayuda a reducir el **training-serving skew**. Este problema aparece cuando una misma característica se calcula de una forma durante el entrenamiento y de otra durante producción. Por ejemplo:

```text
Entrenamiento:
compras de las últimas 24 horas exactas

Producción:
compras desde las 00:00 del día anterior
```

Ambas variables podrían llamarse `compras_24h`, pero representan cosas distintas. El modelo habría sido entrenado con una definición y recibiría otra durante la inferencia. Feature Store ayuda a reducir este riesgo al centralizar las características y permitir reutilizarlas en distintos puntos del ciclo de vida del modelo.

La imagen mental útil es:

```text
datos → cálculo de features → Feature Store
                              ↙           ↘
                    Online Store     Offline Store
                         ↓                ↓
                 inferencia online   entrenamiento
```

## Limpieza y transformación de datos

Antes de construir características conviene corregir problemas básicos del dataset. La limpieza intenta evitar que errores, inconsistencias o valores mal representados terminen convertidos en features defectuosas.

Los **valores faltantes** pueden imputarse, eliminarse o tratarse explícitamente según el problema. Por ejemplo, una `edad = NULL` puede imputarse y, si resulta útil, acompañarse de una variable como `edad_missing = 1`.

Los **valores atípicos** deben identificarse, pero no eliminarse automáticamente. Un valor como `edad = 350` probablemente sea un error; una compra de `$150,000` puede ser completamente válida y además muy importante para un modelo de fraude.

La **deduplicación** evita que observaciones repetidas alteren conteos, medias y otras características derivadas. Si una misma transacción aparece tres veces, por ejemplo, `numero_transacciones_24h` quedará artificialmente inflado.

También es necesario **estandarizar formatos y unidades**. Fechas, monedas, zonas horarias, categorías y unidades de medida deben representar la información de forma consistente. `1.80 metros`, `180 centímetros` y `70.9 pulgadas` describen prácticamente lo mismo, pero no deberían llegar así directamente a una feature.

Finalmente, deben corregirse **errores e inconsistencias**, como fechas imposibles, categorías mal codificadas o combinaciones contradictorias entre variables.

El flujo general es:

```text
datos crudos
→ limpieza y transformación
→ ingeniería de características
→ Feature Store
→ entrenamiento e inferencia
```

La limpieza corrige los datos; la ingeniería de características construye variables útiles; y Feature Store permite almacenarlas y reutilizarlas de manera consistente.

### Gestión de valores faltantes (_Managing Missing Values_)

No es raro que tu dataset tenga datos faltantes. Por ejemplo, a algunas columnas pueden faltarles valores por un error en la recolección, o quizá el dato simplemente no estaba disponible en la fuente.

Los datos faltantes pueden dificultar que el algoritmo de ML elegido interprete con precisión la relación entre la característica afectada y la variable objetivo. Por eso es importante abordar el problema.

Por desgracia, la mayoría de los algoritmos de ML no pueden manejar automáticamente los valores faltantes. Se requiere conocimiento humano para reemplazar los valores faltantes con algo significativo y relevante para el problema.

> [!warning] Nota de precisión: algoritmos que aceptan valores faltantes
> La afirmación vale para muchos algoritmos (regresión lineal y logística, SVM, k-NN y la mayoría de los estimadores clásicos de scikit-learn), pero no para todos. Varias implementaciones de _gradient boosting_, como XGBoost, LightGBM o `HistGradientBoostingClassifier` de scikit-learn, aceptan `NaN` y aprenden durante el entrenamiento hacia qué rama enviar los valores faltantes.

Los enfoques para abordar los datos faltantes varían según la cantidad de datos faltantes. Las principales técnicas son:

- **Recolectar.** Si la cantidad de datos faltantes es considerable y afecta a varias características, deberías intentar volver a recolectar los datos. Sin embargo, el proceso de ingesta puede ser caro y lento. En ese caso, debes sopesar el costo de recolectar de nuevo, el costo de tener un modelo de ML de bajo rendimiento y alternativas como la imputación o la eliminación.
- **Imputar.** Si tienes pocos valores faltantes, distribuidos aleatoriamente entre las características de tu dataset, puede deberse a una falla en el proceso de ingesta. En ese caso, la imputación probablemente sea una buena opción. Con la imputación, rellenas los datos faltantes con la media, la mediana o el valor más frecuente observado de tu característica. Si la distribución de tu característica es normal, usa la media; si no, usa la mediana o el valor más frecuente. En términos estadísticos, el libro describe un mecanismo de falta completamente al azar (MCAR), que es el caso en que la imputación simple no introduce sesgo en la media. Si la falta depende del propio valor no observado (MNAR), por ejemplo cuando los ingresos altos se declaran menos, imputar con la media sesga la característica. El valor más frecuente (la moda) es la opción natural para características categóricas.
- **Eliminar.** Si la cantidad de datos faltantes es considerable pero se concentra en una misma característica, podrías eliminar la característica completa. Sin embargo, hay que tener cuidado: si eliminas demasiadas características, quizá no te queden datos suficientes para alimentar el modelo de ML.

Con Amazon SageMaker puedes usar bibliotecas de Python como `SimpleImputer` de scikit-learn (`sklearn`) para rellenar valores faltantes. Aquí «con SageMaker» significa que trabajas en un notebook de Jupyter que corre en una máquina de AWS gestionada por SageMaker, donde puedes usar cualquier biblioteca de Python; `SimpleImputer` no es una función de AWS. El siguiente fragmento muestra cómo funciona:

```python
from sklearn.impute import SimpleImputer
import numpy as np

# Create a sample dataset
A = np.array([[1, 2], [None, 4], [5, None]])

# Create a SimpleImputer object
imputer = SimpleImputer(strategy='mean')

# Fit and transform the data
B = imputer.fit_transform(A)

print(B)
```

El siguiente arreglo de entrada

```text
[[1, 2], [None, 4], [5, None]]
```

se transformó en este:

```text
[[1, 2], [3, 4], [5, 3]]
```

Salida real al ejecutar el código (scikit-learn 1.5.2): el resultado es el mismo, pero en punto flotante. Cada hueco se rellena con la media de su columna: (1 + 5)/2 = 3 en la primera y (2 + 4)/2 = 3 en la segunda.

```text
[[1. 2.]
 [3. 4.]
 [5. 3.]]
```

`fit_transform` hace dos cosas: `fit` calcula el valor de reemplazo (aquí la media de cada columna) y `transform` lo aplica. Esa separación es la que después permite calcular la media **solo con los datos de entrenamiento** y aplicarla igual a los de prueba (ver «Fuga de datos» en la sección de división de datos).

AWS Glue DataBrew también puede usarse para rellenar valores faltantes mediante varias transformaciones integradas. **AWS Glue DataBrew** es una herramienta visual de preparación de datos, sin código, que forma parte de la familia AWS Glue. Muestras tus datos en una cuadrícula parecida a una hoja de cálculo, eliges transformaciones con clics (la documentación habla de más de 250; verifica la cifra vigente) y las guardas como una **receta** (_recipe_) reutilizable que después se ejecuta como un trabajo serverless sobre el dataset completo en S3.

### Detección y tratamiento de valores atípicos (_Detecting and Treating Outliers_)

Un valor atípico (_outlier_) es un punto de tu dataset que se distingue de todos los demás por desviarse significativamente de la media.

¿Qué significa exactamente «significativamente»? La respuesta depende del tipo de dato y de su distribución de probabilidad, pero el consenso general es que un punto se considera atípico si se encuentra a más de tres desviaciones estándar de la media. Esto se basa en las propiedades de la distribución normal, en la que aproximadamente el 99.7 % de los datos se encuentra a menos de tres desviaciones estándar de la media.

Los algoritmos de ML son muy sensibles a la distribución y al rango de los valores de tus características. Como los valores atípicos se apartan del patrón de todos los demás puntos, tienden a engañar al algoritmo de ML durante el entrenamiento.

Por ejemplo, considera el siguiente dataset:

```text
[x,y] = [[4, 11], [3.8, 12], [4.5, 12.5], [8, 8], [9, 8.5], [9.5, 7.5], [13, 5], [14, 4.7], [13.7, 6], [25, 23]]
```

En este ejemplo, supón que x indica tu consumo diario de agua e y tu consumo diario de energía. Los números son solo ilustrativos.

Como puedes ver en la Figura 3.7, este dataset se distribuye en tres grupos.

_Figura 3.7 Ejemplo de un valor atípico._

Un punto destaca claramente y actúa como valor atípico. Los tres grupos son los puntos con x cercano a 4, a 9 y a 13–14; el punto [25, 23] queda aislado.

Aunque algunos valores atípicos se deben a errores artificiales, otros aparecen en tu dataset como resultado de fenómenos naturales. Un atípico natural no es producto de un error artificial, sino que refleja alguna verdad presente en los datos.

Tu trabajo como ingeniero de ML es determinar si un valor atípico debe quedarse en tu dataset o si hay que cambiarlo o eliminarlo. Tomas esa decisión apoyándote en técnicas estadísticas que te ayudan a entender qué tan relevante es el valor atípico respecto a los demás puntos de tu dataset.

Es importante distinguir entre ruido y valores atípicos. El primero denota un grupo de puntos erróneos, mientras que los segundos son puntos que se desvían significativamente de la media de tu dataset. En los siguientes capítulos aprenderás a aprovechar el amplio ecosistema de servicios de ML de AWS para determinar qué significa «significativamente».

El tratamiento de valores atípicos se apoya en tres enfoques principales:

- **Eliminar.** Si los valores atípicos se deben a ruido o a errores artificiales, puedes simplemente eliminarlos de tu dataset sin afectar la calidad ni la precisión de tu modelo de ML.
- **Transformación logarítmica.** Al reemplazar el valor atípico por su logaritmo en una base dada (p. ej., base _e_, base 2, base 10, etc.), comprimes el rango de valores de una característica, lo que reduce la variación extrema entre los valores. Como resultado, el valor atípico no quedará tan lejos de los demás valores de esa característica. Por ejemplo, el logaritmo en base 10 de 1000 es 3, es decir, $\log_{10}(1000) = 3$, porque $10^3 = 1000$. Eso lo acerca a otros valores más pequeños sin que pierda su significado.
- **Imputar.** Igual que con los valores faltantes, podrías usar, por ejemplo, la media de la característica e imputar ese valor en lugar del valor atípico. Este sería un enfoque excelente si el valor atípico se debiera a errores artificiales.

> [!warning] Nota de precisión: el logaritmo se aplica a toda la característica
> La redacción («reemplazar **el valor atípico** por su logaritmo») sugiere transformar solo ese punto. En la práctica, la transformación logarítmica se aplica a **todos los valores de la característica**. Si solo se transformara el atípico, la columna mezclaría dos escalas distintas y perdería sentido. Así lo hace la Figura 3.8, que cambia la escala del eje completo. Las fórmulas de este párrafo se perdieron en el `.txt` y se reconstruyeron; la base _e_ es la que falta en «base , base 2».

Los valores atípicos suelen crear una distribución sesgada (asimétrica), como se muestra en la Figura 3.7. La transformación logarítmica puede ayudar a normalizar esa distribución, haciéndola más simétrica y mejorando el rendimiento de tu modelo de ML.

La Figura 3.8 muestra el mismo dataset con el eje de ordenadas _y_ en escala logarítmica. El punto atípico [25, 23] tiene x = 25 e y = 23. Como $\log_{10}(23) \approx 1.36$, puedes ver que este punto ya no se desvía significativamente de la media. (Reconstrucción: el `.txt` perdió la fórmula; se asume base 10 por coherencia con el ejemplo anterior.)

_Figura 3.8 Tratamiento de valores atípicos con transformación logarítmica._

Para dar números: en la escala original, el valor 23 está a 2.58 desviaciones estándar de la media de _y_. Tras aplicar $\log_{10}$ a toda la columna, los valores pasan a estar entre 0.67 y 1.36, y el punto queda a 2.11 desviaciones estándar. Se sigue distinguiendo, pero bastante menos.

Para el examen, necesitas conocer los servicios de AWS que ayudan a detectar y tratar valores atípicos: Amazon SageMaker Data Wrangler y AWS Glue DataBrew. **SageMaker Data Wrangler** es la herramienta visual de SageMaker para importar, explorar, transformar y analizar datos con poco o nada de código (_low-code_). Construyes un **flujo de datos** (_data flow_) paso a paso y después lo exportas a S3, a un pipeline de SageMaker, a Feature Store o a un script de Python. Hoy vive dentro de SageMaker Canvas. Entre sus transformaciones integradas para atípicos hay detección por desviación estándar, por desviación estándar robusta, por cuantiles y por umbrales mínimo y máximo, con opciones para recortar, eliminar o invalidar los valores detectados.

Puedes usar los notebooks de Jupyter de Amazon SageMaker para cargar tu dataset con bibliotecas de Python como pandas. Esos notebooks son entornos de Jupyter que corren en instancias administradas por SageMaker, dentro de SageMaker Studio o como _notebook instances_: tú eliges el tamaño de la máquina y SageMaker la aprovisiona, sin que instales nada en tu computadora.

Para decidir qué algoritmo de detección de atípicos usar, necesitas analizar el histograma de cada característica. La Figura 3.9 muestra los histogramas de nuestro dataset, generados con el siguiente fragmento de Python:

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

# Load dataset
data = np.array([[4, 11], [3.8, 12], [4.5, 12.5], [8, 8],
[9, 8.5], [9.5, 7.5], [13, 5], [14, 4.7], [13.7, 6], [25,
23]])
df = pd.DataFrame(data, columns=['Feature1', 'Feature2'])

# Plot histograms
plt.figure(figsize=(12, 5))

plt.subplot(1, 2, 1)
sns.histplot(df['Feature1'], kde=True)
plt.title('Water Usage Distribution (m³/day)')

plt.subplot(1, 2, 2)
sns.histplot(df['Feature2'], kde=True)
plt.title('Energy Usage Distribution (kWh/day)')

plt.show()
```

_Figura 3.9 Histogramas del consumo de agua y energía._

En la Figura 3.9, los histogramas muestran una distribución sesgada a la derecha en ambas características. Una distribución sesgada a la derecha significa que la mayoría de los puntos se concentran en el extremo inferior del rango, con una cola larga que se extiende hacia la derecha. Esa cola representa un pequeño número de puntos con valores mucho más altos que el resto del dataset. En esencia, la mayoría de los valores son bajos y solo unos pocos son excepcionalmente altos, que es exactamente nuestro escenario, con el punto [25, 23] como el par de valores más alto.

Para datos sesgados, el método del rango intercuartílico (IQR, _interquartile range_) suele considerarse uno de los mejores para detectar valores atípicos. El IQR se ve menos afectado por el sesgo y proporciona una medida robusta para identificar valores extremos.

Si $Q_1$ y $Q_3$ denotan el primer y el tercer cuartil de nuestro dataset, el método funciona así (fórmulas reconstruidas; el `.txt` las perdió):

1. Calcula $Q_1$ y $Q_3$.
2. Calcula $\text{IQR} = Q_3 - Q_1$.
3. Todo punto cuyo valor sea menor que $Q_1 - 1.5 \cdot \text{IQR}$ o mayor que $Q_3 + 1.5 \cdot \text{IQR}$ se considera atípico.

En nuestro ejemplo, podemos aprovechar el método integrado `quantile`, disponible en la clase `DataFrame` de las bibliotecas numpy y pandas, para detectar los atípicos del dataset, como se muestra en el siguiente fragmento:

> [!warning] Nota de precisión: `DataFrame` es de pandas
> NumPy no tiene una clase `DataFrame`. El método `DataFrame.quantile` es de pandas; en NumPy la función equivalente es `np.quantile`, que opera sobre arreglos.

```python
import numpy as np
import pandas as pd

# Load dataset
data = np.array([[4, 11], [3.8, 12], [4.5, 12.5], [8, 8],
[9, 8.5], [9.5, 7.5], [13, 5], [14, 4.7], [13.7, 6], [25,
23]])
df = pd.DataFrame(data, columns=['Feature1', 'Feature2'])

# Calculate IQR
Q1 = df.quantile(0.25)
Q3 = df.quantile(0.75)
IQR = Q3 - Q1

# Define outlier conditions
outliers = ((df < (Q1 - 1.5 * IQR)) | (df > (Q3 + 1.5 *
IQR))).any(axis=1)
df['Outlier'] = outliers

print("Original Data with Outlier Flags:")
print(df)

# Treat outliers by removing them
df_cleaned = df[~df['Outlier']].drop(columns='Outlier')

print("\nCleaned Data without Outliers:")
print(df_cleaned)
```

Este programa produce la salida que se muestra en la Figura 3.10, donde puedes ver cómo el script detectó el punto [25, 23] como atípico y luego lo eliminó.

_Figura 3.10 Detección y tratamiento de valores atípicos._

Salida real (la primera tabla; la segunda, _Cleaned Data_, son las filas 0 a 8, sin la fila 9):

```text
Original Data with Outlier Flags:
   Feature1  Feature2  Outlier
0       4.0      11.0    False
1       3.8      12.0    False
2       4.5      12.5    False
3       8.0       8.0    False
4       9.0       8.5    False
5       9.5       7.5    False
6      13.0       5.0    False
7      14.0       4.7    False
8      13.7       6.0    False
9      25.0      23.0     True
```

Un detalle que la figura no deja ver son los límites que calcula el código:

| Característica     | $Q_1$ | $Q_3$  | IQR   | Límite inferior | Límite superior |
| ------------------ | ----- | ------ | ----- | --------------- | --------------- |
| Feature1 (agua)    | 5.375 | 13.525 | 8.150 | −6.850          | 25.750          |
| Feature2 (energía) | 6.375 | 11.750 | 5.375 | −1.688          | 19.813          |

x = 25 **no** supera su límite (25.75); lo que marca la fila es y = 23 > 19.81. La fila completa se marca como atípica porque `.any(axis=1)` basta con que **una** columna esté fuera de rango.

Para datos con distribución normal, el método de la puntuación Z (_Z-score_) es muy eficaz para detectar valores atípicos. Así funciona (fórmulas reconstruidas):

1. Calcula la media $\mu$ de tu dataset.
2. Calcula la desviación estándar $\sigma$ de tu dataset.
3. Para cada punto $x_i$, la puntuación Z es $z_i = \dfrac{x_i - \mu}{\sigma}$.

Como una distribución normal se caracteriza por lo siguiente:

- Una desviación estándar cubre alrededor del 68 % de los datos.
- Dos desviaciones estándar cubren alrededor del 95 % de los datos.
- Tres desviaciones estándar cubren alrededor del 99.7 % de los datos.

los puntos con puntuaciones Z mayores que 3 o menores que −3 se consideran atípicos. En otras palabras, es estadísticamente razonable suponer que el 0.3 % de los puntos cuya puntuación Z es mayor que 3 o menor que −3 están lo bastante lejos de la media como para considerarse atípicos.

El método de la puntuación Z también puede calcularse con código. El siguiente fragmento muestra cómo:

```python
import numpy as np
import pandas as pd
from scipy import stats

# Load dataset
data = np.array([[4, 11], [3.8, 12], [4.5, 12.5], [8, 8],
[9, 8.5], [9.5, 7.5], [13, 5], [14, 4.7], [13.7, 6], [25,
23]])
df = pd.DataFrame(data, columns=['Feature1', 'Feature2'])

# Detect outliers using Z-score
z_scores = np.abs(stats.zscore(df))
outliers = (z_scores > 3).any(axis=1)
df['Outlier'] = outliers

print("Original Data with Outlier Flags:")
print(df)

# Treat outliers by removing them
df_cleaned = df[~df['Outlier']].drop(columns='Outlier')

print("\nCleaned Data without Outliers:")
print(df_cleaned)
```

El código anterior produce la salida representada en la Figura 3.11.

_Figura 3.11 Detección de atípicos fallida._

Salida real: todas las filas quedan con `Outlier = False`. Las puntuaciones |z| del punto [25, 23] son 2.39 en `Feature1` y 2.58 en `Feature2`, ambas por debajo de 3.

Como puedes ver, el método de la puntuación Z no detectó el atípico [25, 23]. Esto se debe a que la distribución de los puntos de nuestro dataset no es normal, sino sesgada a la derecha.

> [!warning] Nota de precisión: con 10 datos, z > 3 es imposible
> Hay una razón más fuerte que el sesgo, y es aritmética. Con la desviación estándar poblacional (la que usa `scipy.stats.zscore` por defecto, `ddof=0`), la mayor puntuación |z| posible en una muestra de tamaño _n_ es $(n-1)/\sqrt{n}$ (resultado de Shiffler, 1988). Para _n_ = 10 eso da 2.85, así que **ningún** punto de este dataset podría superar el umbral de 3, fuera cual fuera su valor. La regla de las tres desviaciones solo tiene sentido con muestras de tamaño razonable, y además la media y la desviación estándar que usa se ven infladas por el propio atípico (efecto de enmascaramiento). El IQR no sufre ninguno de los dos problemas.

Con AWS Glue DataBrew también puedes detectar valores atípicos en tus datos y manejarlos con varias transformaciones. Estas incluyen reemplazar, eliminar, reescalar o marcar los valores atípicos con los métodos que acabas de aprender, es decir, la puntuación Z y el IQR.

### Deduplicación (_Performing Deduplication_)

La deduplicación es el proceso de eliminar los puntos duplicados de tu dataset. No se trata solo de ordenar tus datos: se trata de que tus modelos sean más precisos, tengan mejor rendimiento y sean más confiables.

Desde el punto de vista de la calidad de los datos, los duplicados pueden distorsionar los resultados y llevar a predicciones inexactas. Eliminarlos asegura que los datos usados para entrenar los modelos estén limpios y sean precisos.

El rendimiento y la confiabilidad del modelo son factores clave en las fases de evaluación y despliegue del ciclo de vida de ML. Los duplicados pueden causar sobreajuste, en el que el modelo aprende el ruido en lugar de la señal. Los datos limpios y deduplicados ayudan a crear modelos que generalizan mejor.

Desde el punto de vista de la confiabilidad, los duplicados pueden introducir sesgo, en particular si ciertas entradas se repiten más que otras, lo que produce datos de entrenamiento desbalanceados.

Por todo esto, una estrategia de deduplicación eficaz es crítica para asegurar la calidad de los datos de entrenamiento de tu modelo de ML.

Hay una razón adicional, implícita en el texto, para deduplicar **antes** de dividir los datos en entrenamiento y prueba. Si el mismo registro queda en ambos conjuntos, el modelo se evalúa con ejemplos que ya «memorizó» y las métricas salen infladas, lo que es una forma de fuga de datos. Conviene distinguir también entre duplicados exactos (filas idénticas) y casi duplicados, como el mismo cliente escrito «Juan Pérez» y «JUAN PEREZ». Estos últimos requieren normalizar texto o hacer una comparación aproximada.

Por simplicidad y facilidad de uso, AWS Glue DataBrew es probablemente tu mejor opción para eliminar duplicados de tu dataset. Ofrece una interfaz visual y directa para limpiar y preparar tus datos sin necesidad de código complejo. Es fácil de usar y eficiente, lo que hace que tareas de limpieza como eliminar duplicados sean rápidas y sin complicaciones.

### Estandarización y reformateo (_Standardizing and Reformatting_)

Al aplicar la estandarización, evitas que las características con valores grandes influyan en el modelo de forma desproporcionada frente a las características con valores pequeños. Esta técnica es especialmente importante para algoritmos de ML sensibles a la escala de las características, como la regresión lineal y las máquinas de vectores de soporte (SVM), que se verán en el próximo capítulo.

Reformatear, en ML, significa reestructurar tus datos en un formato consistente y adecuado para el análisis. Incluye convertir tipos de datos, armonizar formatos y asegurar que todas las entradas sigan la misma estructura.

Para realizar la estandarización y el reformateo en AWS, puedes usar Amazon SageMaker y AWS Glue. Los pasos principales son:

- **Cargar los datos.** Puedes usar notebooks de SageMaker para cargar tu dataset desde un bucket de S3, como se muestra en el siguiente fragmento:

  ```python
  import pandas as pd
  data = pd.read_csv('your_dataset.csv')
  ```

  Así escrito, el fragmento lee un archivo local. Para leer directamente desde S3, la ruta se escribe como `'s3://nombre-del-bucket/ruta/your_dataset.csv'`. pandas lo resuelve si está instalada la biblioteca `s3fs` y si el rol de IAM del notebook tiene permiso de lectura sobre ese bucket. Ese rol es la identidad con la que el notebook actúa ante AWS.

- **Estandarizar los datos.** Puedes usar la clase `StandardScaler` del módulo `sklearn.preprocessing` para estandarizar tus características, como se muestra en el siguiente fragmento:

  ```python
  from sklearn.preprocessing import StandardScaler
  scaler = StandardScaler()
  data_scaled = scaler.fit_transform(data)
  ```

  En la práctica se aplica solo a las columnas numéricas (`data[columnas_numericas]`); con columnas de texto, `fit_transform` da error.

  Más técnicas de normalización y estandarización se verán en las próximas secciones.

- **Exportar los datos.** Puedes guardar el dataset estandarizado en S3, por ejemplo con `df.to_csv('s3://...')` (mismos requisitos que para leer, pero con permiso de escritura) o con `boto3`, el SDK de AWS para Python.
- **Crear un trabajo de AWS Glue.** Crea un trabajo (_job_) de AWS Glue para reformatear los datos según sea necesario. Un **Glue job** es un script (normalmente en PySpark, la interfaz de Python de Apache Spark para procesamiento distribuido) que Glue ejecuta en infraestructura serverless. Tú defines el script y la capacidad, Glue levanta las máquinas, corre el trabajo, las apaga y cobra por el tiempo de cómputo usado.
- **Cargar los datos limpios.** Usa los datos limpios y estandarizados para entrenar tu modelo de ML.

El libro combina las dos herramientas sin explicar por qué. El reparto habitual es que el notebook sirve para explorar y probar la transformación sobre una muestra, y el Glue job ejecuta esa misma lógica de forma repetible sobre el volumen completo, por ejemplo cada noche, sin que nadie tenga un notebook abierto.

### Eliminación de ruido y errores (_Removing Noise and Errors_)

El ruido natural y los errores artificiales se presentaron brevemente antes. Como el objetivo principal de la ingeniería de características es producir los mejores datos de entrenamiento posibles para tu modelo de ML, uno de sus aspectos clave es la capacidad de manejar el ruido y los errores.

Al crear características nuevas o transformar las existentes, haces que los datos de entrenamiento sean más informativos y relevantes para tu modelo de ML. Este proceso ayuda a mejorar la precisión, la eficiencia y la capacidad de generalización del modelo, lo que aumenta su poder predictivo.

Un modelo entrenado con datos ruidosos a menudo aprende peculiaridades o errores específicos del dataset de entrenamiento en lugar de los patrones subyacentes. Esto lleva al sobreajuste (_overfitting_): el modelo rinde muy bien con los datos de entrenamiento, pero mal con datos nuevos que no ha visto. Al manejar adecuadamente el ruido y los errores, ayudas a que el modelo generalice mejor en escenarios del mundo real. En el capítulo 5 veremos en detalle los métodos para abordar el sobreajuste.

## Técnicas de ingeniería de características (_Feature Engineering Techniques_)

En la sección anterior se presentaron los métodos para limpiar tu dataset. La limpieza de datos es un paso preliminar en la preparación de los datos de entrenamiento de tu modelo.

El objetivo del siguiente paso es transformar eficazmente los datos crudos en características significativas que mejoren el rendimiento de tu modelo de ML. Este es el foco principal de la ingeniería de características. En última instancia, después de la ingeniería de características, tu modelo de ML estará listo para el entrenamiento.

Dominar la ingeniería de características es crucial para demostrar tu experiencia en la construcción y el despliegue de soluciones de ML escalables y eficientes en AWS. Las próximas secciones cubren las técnicas de ingeniería de características que necesitas conocer para el examen.

Empezaremos con las técnicas para datos estructurados y seguiremos con las técnicas para datos no estructurados, en forma de imágenes, texto y series de tiempo.

### Técnicas para datos estructurados (_Techniques for Structured Data_)

Con datos estructurados, puedes usar varias técnicas de extracción de características para reducir la dimensionalidad de tu dataset. Algo de lo que debes cuidarte al usar la extracción de características es que, cuando pongas este modelo en producción o automatices el pipeline, estas características puedan replicarse fácilmente y a la vez sigan reduciendo las altas dimensiones de los datos.

La frase original es confusa. Lo que quiere decir es lo siguiente. Estar **en producción** significa que el modelo ya está sirviendo predicciones reales a usuarios o sistemas. Un **pipeline** es una cadena automatizada de pasos (ingerir → transformar → entrenar → desplegar) que se ejecuta sin intervención manual. Toda transformación que se «ajusta» durante el entrenamiento, como los componentes de un PCA, la media y la desviación de un escalador o el vocabulario de un codificador, debe guardarse y aplicarse **exactamente igual** a cada dato nuevo en el momento de la inferencia. Si la extracción no puede reproducirse de forma idéntica fuera del notebook donde se inventó, aparece el _training-serving skew_ descrito en la sección de Feature Store.

#### Ingeniería de características para datos numéricos (_Feature Engineering for Numerical Data_)

Los datos numéricos son, en última instancia, el tipo de dato con el que quieres alimentar tu modelo de ML. Sean tus datos categóricos, textuales o de imagen, el resultado de tu ingeniería de características tendrá que ser un conjunto de números distinto y bien organizado con el que pueda entrenarse tu algoritmo de ML.

La forma en que conviertes ese conjunto de números en características significativas puede tener un impacto notable en el rendimiento y la precisión de tu modelo.

En las próximas secciones profundizaremos en los conceptos de ingeniería de características presentados antes.

##### Normalización (_Normalization_)

La normalización es un enfoque de ingeniería de características que lleva los datos numéricos de todas tus características a una escala consistente, normalmente entre 0 y 1. Su forma habitual es $x' = \dfrac{x - x_{\min}}{x_{\max} - x_{\min}}$. Esto tiene varios beneficios clave:

- **Ponderación equitativa.** La normalización asegura que ninguna característica domine el modelo por su escala. Esto es especialmente importante para algoritmos de ML sensibles a la escala de los datos, como k vecinos más cercanos (k-NN) y las redes neuronales, que calculan distancias entre puntos.
- **Convergencia más rápida.** En la optimización por descenso de gradiente, los datos normalizados ayudan a que el algoritmo converja más rápido, porque todas las características contribuyen por igual al gradiente. Esto da lugar a un entrenamiento más eficiente y a un mejor rendimiento de tu modelo.
- **Mejor interpretabilidad.** Los datos normalizados facilitan la interpretación de la importancia de las características y de los coeficientes del modelo, ya que todas las características están en una escala comparable.

> [!warning] Nota de precisión: las redes neuronales no calculan distancias
> k-NN sí depende de distancias entre puntos. Las redes neuronales, en general, no las calculan. Son sensibles a la escala por otra razón, la del punto siguiente: la optimización por descenso de gradiente se vuelve lenta e inestable cuando las entradas tienen escalas muy distintas, y algunas funciones de activación se saturan con entradas grandes.

Esta técnica es útil cuando quieres que tus datos estén en una escala común, pero no necesitan estar centrados en cero.

Como resultado de la normalización, los modelos de ML rinden mejor cuando todas las características de tu dataset están en la misma escala. La normalización puede dar mayor precisión y estabilidad, especialmente en algoritmos basados en distancias.

La normalización se implementa típicamente con la función de escalado MinMax, que se verá en detalle en la sección «Escalado».

##### Estandarización (_Standardization_)

La estandarización transforma las características para que tengan media 0 y desviación estándar 1. Se usa cuando quieres asegurar que los datos de tus características estén centrados alrededor de la media ($\mu$) y escalados según la desviación estándar ($\sigma$).

Esta técnica es ideal para algoritmos de ML que asumen datos con distribución normal, como la regresión lineal, la regresión logística o los algoritmos que usan descenso de gradiente.

> [!warning] Nota de precisión: estandarizar no normaliza la distribución
> Dos matices para quien viene de la estadística. (1) La estandarización es una transformación lineal: **no cambia la forma** de la distribución, y si los datos eran asimétricos siguen siéndolo (el propio libro lo dice en «Puntos esenciales para el examen»). (2) La regresión lineal no asume normalidad de las **características**, sino de los **residuos**, y la regresión logística no asume normalidad. Las razones reales por las que la estandarización ayuda a estos modelos son la convergencia del descenso de gradiente y que las penalizaciones de regularización (L1/L2) traten a todos los coeficientes en la misma escala.

La estandarización se implementa con la función de puntuación Z.

**Puntuación Z.** Esta solución transforma el dataset de la característica para que tenga media 0 y desviación estándar 1 con la siguiente fórmula (reconstruida; se perdió en el `.txt`):

$$z = \frac{x - \mu}{\sigma}$$

donde

- $x$ es un punto de tu característica.
- $\mu$ es la media del dataset de tu característica.
- $\sigma$ es la desviación estándar del dataset de tu característica.

La desviación estándar resultante es 1 por la naturaleza de la transformación.

Al restar la media ($\mu$) de cada punto ($x$), centras los datos en 0: cada punto se ajusta en relación con el valor promedio del dataset.

Al dividir por la desviación estándar ($\sigma$), escalas los puntos en relación con la dispersión de los datos. Este paso asegura que la varianza (el cuadrado de la desviación estándar) de los datos estandarizados sea 1.

##### Escalado (_Scaling_)

El escalado es el término general para ajustar el rango o la distribución de las características. La normalización se ocupa sobre todo del rango del dataset transformado (escala de 0 a 1 para todas las características), mientras que la estandarización se centra en la distribución (media $\mu$ igual a 0 y desviación estándar $\sigma$ igual a 1 en el dataset transformado). Ambas técnicas forman parte del escalado.

Si los datos de tu característica no tienen distribución normal, necesitas técnicas que manejen el sesgo o los atípicos de forma distinta a la estandarización por puntuación Z. Las que necesitas conocer para el examen son las siguientes:

**Escalado robusto (_robust scaling_).** El escalado robusto centra el dataset de la característica restando la mediana y lo escala según el IQR, es decir, $x' = \dfrac{x - \text{mediana}}{\text{IQR}}$. Esto lo hace especialmente útil para datasets con atípicos, porque asegura que la tendencia central y la dispersión sean robustas frente a ellos. Como resultado, esta técnica de estandarización es menos sensible a los atípicos y funciona bien con datos sesgados.

Puedes usar la clase `RobustScaler` del módulo `sklearn.preprocessing` para escalar tus características con esta técnica, como se muestra en el siguiente ejemplo:

```python
import numpy as np
import pandas as pd
from sklearn.preprocessing import RobustScaler

# Load dataset
data = np.array([[4, 11], [3.8, 12], [4.5, 12.5], [8, 8],
[9, 8.5], [9.5, 7.5], [13, 5], [14, 4.7], [13.7, 6], [25,
23]])
df = pd.DataFrame(data, columns=['Feature1', 'Feature2'])

# Apply RobustScaler
scaler = RobustScaler()
data_scaled = scaler.fit_transform(df)

# Convert the scaled data back to a DataFrame for easier viewing
df_scaled = pd.DataFrame(data_scaled, columns=['Feature1',
'Feature2'])

print("Original Data:")
print(df)
print("\nScaled Data:")
print(df_scaled)
```

El código anterior produce la salida representada en la Figura 3.12.

_Figura 3.12 Escalado robusto._

Salida real (parte _Scaled Data_, redondeada a 3 decimales):

```text
   Feature1  Feature2
0    -0.644     0.512
1    -0.669     0.698
2    -0.583     0.791
3    -0.153    -0.047
4    -0.031     0.047
5     0.031    -0.140
6     0.460    -0.605
7     0.583    -0.660
8     0.546    -0.419
9     1.933     2.744
```

Como puedes ver en la Figura 3.12, con el escalado robusto el data frame escalado es menos sensible a los atípicos que el original. Como resultado de la transformación, atípicos como [25, 23] no sesgan el nuevo dataset de tu característica.

> [!warning] Nota de precisión: qué es exactamente lo «robusto»
>
> - El escalado robusto es lineal, así que **el atípico sigue siendo atípico**. En la salida, 2.744 sigue muy separado del resto, cuyo máximo es 0.791. Lo robusto son los **parámetros** de la transformación: la mediana y el IQR casi no cambian por la presencia de [25, 23], de modo que el resto de los puntos queda bien escalado. Con la media y la desviación estándar, el atípico inflaría $\sigma$ y «aplastaría» a los demás puntos. Tampoco corrige el sesgo de la distribución.
> - El libro llama a esta técnica, y más abajo a MinMax, «técnica de estandarización». Según sus propias definiciones, MinMax es **normalización**. Conviene tener clara la terminología del libro para el examen: normalización = rango fijo; estandarización = media 0 y desviación 1.

**Escalado MinMax.** Esta técnica de estandarización transforma las características a un rango fijo, normalmente entre 0 y 1. Preserva las relaciones entre las características, pero no corrige el sesgo. Más precisamente, como es lineal, preserva la forma de la distribución de cada característica y las distancias relativas entre sus valores. Su punto débil es que depende del mínimo y del máximo observados, así que un solo atípico comprime a todos los demás valores en una franja estrecha.

Puedes usar la clase `MinMaxScaler` del módulo `sklearn.preprocessing` para escalar tus características con este enfoque, como se muestra en el siguiente fragmento:

```python
import numpy as np
import pandas as pd
from sklearn.preprocessing import MinMaxScaler

# Sample data
data = np.array([[1, 2], [3, 4], [5, 6], [7, 8], [-1, -2],
[-3, -4]])
df = pd.DataFrame(data, columns=['Feature1', 'Feature2'])

# Initialize MinMaxScaler
scaler = MinMaxScaler()

# Fit and Transform
data_scaled = scaler.fit_transform(df)
df_scaled = pd.DataFrame(data_scaled, columns=['Feature1',
'Feature2'])

# Print results
print("Original Data:\n", df)
print("\nScaled Data:\n", df_scaled)
```

Salida real (parte _Scaled Data_, redondeada). El mínimo (−3 y −4) pasa a 0 y el máximo (7 y 8) pasa a 1:

```text
   Feature1  Feature2
0       0.4     0.500
1       0.6     0.667
2       0.8     0.833
3       1.0     1.000
4       0.2     0.167
5       0.0     0.000
```

**Escalado MaxAbs.** El escalado MaxAbs es una técnica para estandarizar datos escalando cada punto de la característica respecto de su valor absoluto máximo, es decir, dividiéndolo entre ese máximo: $x' = x / \max|x|$. Como resultado, cada punto cae dentro del rango [−1, 1].

Esta técnica es la más adecuada para estandarizar datasets dispersos (_sparse_) o con muchos ceros, porque preserva las entradas en 0 y mantiene la distribución original de los datos. Un dataset _disperso_ es uno donde la gran mayoría de los valores son cero, como una matriz de «qué palabras aparecen en cada documento». Estos datos se guardan en formatos que solo almacenan los valores distintos de cero. Como MaxAbs solo divide y no resta nada, un 0 sigue siendo 0 y la matriz sigue siendo dispersa. MinMax o la puntuación Z, en cambio, restan una constante y convertirían todos esos ceros en valores distintos de cero, lo que dispararía el uso de memoria.

Puedes usar la clase `MaxAbsScaler` del módulo `sklearn.preprocessing` para escalar tus características con este enfoque, como se muestra en el siguiente fragmento:

```python
import numpy as np
import pandas as pd
from sklearn.preprocessing import MaxAbsScaler

# Sample data
data = np.array([[1, 2], [3, 4], [5, 6], [7, 8], [-1, -2],
[-3, -4]])
df = pd.DataFrame(data, columns=['Feature1', 'Feature2'])

# Initialize MaxAbsScaler
scaler = MaxAbsScaler()

# Fit and Transform
data_scaled = scaler.fit_transform(df)
df_scaled = pd.DataFrame(data_scaled, columns=['Feature1',
'Feature2'])

# Print results
print("Original Data:\n", df)
print("\nScaled Data:\n", df_scaled)
```

Salida real (parte _Scaled Data_, redondeada). Todo se divide entre 7 y entre 8, respectivamente, y los signos se conservan:

```text
   Feature1  Feature2
0     0.143      0.25
1     0.429      0.50
2     0.714      0.75
3     1.000      1.00
4    -0.143     -0.25
5    -0.429     -0.50
```

**Transformaciones de potencia (Box-Cox, Yeo-Johnson).** La varianza de un dataset es el cuadrado de la desviación estándar ($\sigma^2$). Por definición, la varianza siempre es un número positivo y mide la dispersión de los puntos de un dataset, es decir, qué tan lejos está cada punto de la media ($\mu$). Matemáticamente, es el promedio de las diferencias al cuadrado respecto de la media. Una varianza alta significa que los puntos están dispersos lejos de la media; una varianza baja, que están cerca de ella.

> [!warning] Nota de precisión: menor detalle
> La varianza es **no negativa**, no estrictamente positiva: vale 0 cuando todos los valores son iguales.

Las transformaciones de potencia ayudan a estabilizar la varianza y a que los datos tengan una distribución más cercana a la normal. Dos tipos comunes son la transformación de Box-Cox y la de Yeo-Johnson. La primera solo funciona con datos positivos, mientras que la segunda maneja datos positivos y negativos. Box-Cox es $y = (x^\lambda - 1)/\lambda$ para $\lambda \neq 0$ y $y = \ln x$ para $\lambda = 0$, y exige $x > 0$ estrictamente. Yeo-Johnson extiende la idea a ceros y negativos. En ambas, el parámetro $\lambda$ se estima con los datos (por máxima verosimilitud en scikit-learn).

Puedes usar la clase `PowerTransformer` del módulo `sklearn.preprocessing` para escalar tus características con este enfoque, como se muestra en el siguiente fragmento:

```python
import numpy as np
import pandas as pd
from sklearn.preprocessing import PowerTransformer

# Sample data
data = np.array([[1, 2], [3, 4], [5, 6], [7, 8], [-1, -2],
[-3, -4]])
df = pd.DataFrame(data, columns=['Feature1', 'Feature2'])

# Initialize Yeo-Johnson Transformation
pt = PowerTransformer(method='yeo-johnson')

# Fit and Transform
data_scaled = pt.fit_transform(df)
df_scaled = pd.DataFrame(data_scaled, columns=['Feature1',
'Feature2'])

# Print results
print("Original Data:\n", df)
print("\nScaled Data:\n", df_scaled)
```

Salida real (parte _Scaled Data_, redondeada). Las $\lambda$ estimadas son 0.915 y 1.023. Como están cerca de 1, la transformación apenas cambia la forma, lo cual es esperable porque estos datos de ejemplo son simétricos. Los valores tienen media 0 y desviación estándar 1 porque `PowerTransformer` **estandariza la salida por defecto** (`standardize=True`):

```text
   Feature1  Feature2
0    -0.232    -0.098
1     0.336     0.383
2     0.879     0.868
3     1.406     1.357
4    -0.854    -1.029
5    -1.535    -1.480
```

Las transformaciones de potencia hacen que tus datos sean más aptos para el modelado al reducir el sesgo y estabilizar la varianza, lo que da lugar a modelos más robustos y precisos.

##### Transformación logarítmica (_Logarithmic Transformation_)

Como aprendiste antes, la transformación logarítmica es un método para detectar y gestionar atípicos. Como técnica de ingeniería de características para datos numéricos, ayuda a reducir el sesgo, limitar el impacto de los atípicos y reforzar las relaciones lineales. Así:

- **Reducir el sesgo.** Muchos datasets del mundo real están sesgados a la derecha: la mayoría de los puntos se agrupan en valores bajos y unos pocos atípicos extienden la «cola» hacia valores altos. Ejemplos comunes son la distribución del ingreso, donde un pequeño número de personas con ingresos altos crea una cola larga a la derecha, o las calificaciones de un examen difícil, donde la mayoría de los estudiantes obtiene notas bajas y unos pocos destacan arriba. Aplicar una transformación logarítmica comprime esa cola, lo que hace la distribución más simétrica y más cercana a una normal, que muchos algoritmos de ML manejan mejor.
- **Limitar el impacto de los atípicos.** Los atípicos de valor alto pueden dominar y distorsionar la escala de los datos numéricos. La transformación logarítmica reduce la magnitud de esos atípicos y los acerca al grueso de los datos sin eliminarlos.
- **Reforzar las relaciones lineales.** En algunos datos, la relación entre las características y la variable objetivo puede ser multiplicativa o exponencial en lugar de lineal. La transformación logarítmica puede linealizar esas relaciones, lo que facilita que los modelos lineales capten los patrones subyacentes.

Por ejemplo, considera el siguiente dataset:

```text
[x,y] = [[3, 1], [4, 10], [5, 100], [6, 1000]]
```

Al transformar $y$ en $\log_{10}(y)$ (fórmula reconstruida), el nuevo dataset queda así:

```text
[x,y] = [[3, 0], [4, 1], [5, 2], [6, 3]]
```

Aquí se ve la linealización: _y_ se multiplica por 10 cada vez que _x_ aumenta en 1 (relación exponencial), y tras el logaritmo la relación es una recta exacta, $\log_{10} y = x - 3$.

Al convertir los datos a escala logarítmica, las variaciones se vuelven más manejables y la estructura general de los datos se ve con más claridad. Esta transformación puede mejorar notablemente el rendimiento y la precisión de muchos modelos.

> [!note] Recuadro del libro: ceros y negativos
> La transformación logarítmica no puede aplicarse directamente a características con valores 0 o negativos, porque el logaritmo de 0 y de los números negativos no está definido, sea cual sea la base. En esos casos, considera desplazar tus datos para que todos los valores sean positivos, o usar transformaciones alternativas (p. ej., la raíz cúbica), que pueden ser más adecuadas para tus datos.
>
> El desplazamiento más usado para conteos con ceros es $\log(x + 1)$, disponible como `np.log1p` en NumPy.

##### Raíz cuadrada o cúbica (_Square or Cube Root_)

Para manejar una varianza alta en datasets numéricos (además de la transformación de potencia vista antes), considera las transformaciones de raíz cuadrada o raíz cúbica. La raíz cuadrada y la cúbica de una característica afectan su distribución, aunque ese efecto no es tan marcado como el de la transformación logarítmica. La raíz cúbica tiene una ventaja propia: puede aplicarse a valores negativos, incluido el 0. La raíz cuadrada solo puede aplicarse a valores positivos y a 0.

##### Discretización (_Binning_)

La discretización (_binning_, también llamada _bucketing_) es el proceso de dividir datos numéricos continuos en intervalos discretos o «contenedores» (_bins_). Esta técnica puede simplificar el desarrollo de modelos de ML y hacerlos más robustos frente a atípicos y ruido. Con el binning, transformas datos numéricos en datos categóricos.

Por ejemplo, si trabajas con datos de automóviles, puedes convertir el peso del vehículo en cinco columnas: `is_minicompact`, `is_subcompact`, `is_compact`, `is_midsize` e `is_large`. Aquí hay en realidad dos pasos encadenados: primero el binning (peso → una de cinco categorías) y después una codificación _one-hot_ de esa categoría (una columna 0/1 por contenedor), técnica que se explica en la sección siguiente.

#### Ingeniería de características para datos categóricos (_Feature Engineering for Categorical Data_)

Los datos categóricos pueden captar información importante sobre las relaciones y características de tus datos que los datos numéricos podrían pasar por alto. Las características categóricas bien construidas pueden mejorar notablemente el rendimiento de tus modelos de ML.

La ingeniería de características para datos categóricos empieza por la **codificación** (_encoding_), que es el proceso de transformar los datos de tu característica de formato texto a formato numérico. El formato numérico puede ser un entero, un arreglo de enteros, una matriz o incluso un tensor de enteros (un tensor es un arreglo multidimensional, la generalización de vectores y matrices a más ejes, como un `ndarray` de NumPy de tres o más dimensiones). La codificación se hace para que los algoritmos de ML puedan interpretar y usar estos datos como entrada, ya que la mayoría de los algoritmos solo entienden valores numéricos.

Por ejemplo, si tus categorías son White, Black y Red, podrías codificar estos datos en tres vectores: [1, 0, 0] para representar White, [0, 1, 0] para Black y [0, 0, 1] para Red. Este ejemplo es, de hecho, la codificación _one-hot_ que se describe más abajo.

##### Codificación por etiquetas (_Label Encoding_)

La codificación por etiquetas asigna, como indica su nombre, un entero único a cada categoría de tu característica categórica. Se llama así porque le pone a cada categoría una «etiqueta» numérica. Por ejemplo, la característica categórica «Color», con valores como estos:

```text
["White", "Black", "Red"]
```

se convierte en esto:

```text
[0, 1, 2]
```

Aunque es sencilla, esta técnica puede ser problemática con algoritmos que asumen relaciones ordenadas en los datos, porque impone una estructura ordinal a los valores categóricos. Un modelo lineal interpretaría que Red (2) es «el doble» de Black (1) y que Black está «entre» White y Red, relaciones que no existen.

Esta técnica de codificación es la más adecuada para algoritmos de ML basados en árboles, porque estos algoritmos pueden manejar la relación ordinal implícita en la codificación por etiquetas. Los árboles solo hacen cortes del tipo «¿código ≤ 1?». Con suficientes cortes pueden aislar cualquier categoría, así que el orden artificial les estorba poco.

> [!warning] Nota de precisión: `LabelEncoder` ordena alfabéticamente
> El mapeo [White, Black, Red] → [0, 1, 2] es ilustrativo. `sklearn.preprocessing.LabelEncoder` asigna los códigos en **orden alfabético**, así que da Black = 0, Red = 1, White = 2; ejecutado sobre esa lista devuelve `[2, 0, 1]`. Además, está pensado para codificar la **variable objetivo**, no las características. Para características ordinales en las que el orden importa (Small < Medium < Large), usa `OrdinalEncoder(categories=[["Small", "Medium", "Large", "X-Large"]])` para fijar el orden correcto explícitamente. Data Wrangler llama a esta transformación _Ordinal encode_.

##### Codificación one-hot (_One-Hot Encoding_)

La codificación one-hot convierte una característica categórica en un conjunto de características binarias. Cada característica binaria representa una categoría posible y vale 1 si la categoría está presente y 0 si no.

Puedes usar la clase `OneHotEncoder` del módulo `sklearn.preprocessing` para codificar tus características con este enfoque, como se muestra en el siguiente fragmento:

```python
import pandas as pd
from sklearn.preprocessing import OneHotEncoder

# Sample data
data = {'Color': ['White', 'Black', 'Red', 'Blue',
'Green', 'Yellow', 'Pink', 'Brown', 'White', 'Black']}
df = pd.DataFrame(data)

# Initialize OneHotEncoder
encoder = OneHotEncoder(sparse_output=False)

# Fit and transform the data
one_hot = encoder.fit_transform(df[['Color']])

# Convert to DataFrame for easier viewing
one_hot_df = pd.DataFrame(one_hot,
columns=encoder.get_feature_names_out(['Color']))

# Print results
print("Original Data:\n", df)
print("\nOne-Hot Encoded Data:\n", one_hot_df)
```

La Figura 3.13 muestra el resultado producido por este programa.

_Figura 3.13 Codificación one-hot._

Salida real (parte _One-Hot Encoded Data_):

```text
   Color_Black  Color_Blue  Color_Brown  Color_Green  Color_Pink  Color_Red  Color_White  Color_Yellow
0          0.0         0.0          0.0          0.0         0.0        0.0          1.0           0.0
1          1.0         0.0          0.0          0.0         0.0        0.0          0.0           0.0
2          0.0         0.0          0.0          0.0         0.0        1.0          0.0           0.0
3          0.0         1.0          0.0          0.0         0.0        0.0          0.0           0.0
4          0.0         0.0          0.0          1.0         0.0        0.0          0.0           0.0
5          0.0         0.0          0.0          0.0         0.0        0.0          0.0           1.0
6          0.0         0.0          0.0          0.0         1.0        0.0          0.0           0.0
7          0.0         0.0          1.0          0.0         0.0        0.0          0.0           0.0
8          0.0         0.0          0.0          0.0         0.0        0.0          1.0           0.0
9          1.0         0.0          0.0          0.0         0.0        0.0          0.0           0.0
```

Como puedes ver, el dataset original tenía una característica categórica, «Color», con 10 puntos (categorías): White, Black, Red, Blue, Green, Yellow, Pink, Brown, White y Black.

El dataset codificado con one-hot se transformó en ocho características numéricas, igual al número de colores distintos: `Color_Black`, `Color_Blue`, `Color_Brown`, `Color_Green`, `Color_Pink`, `Color_Red`, `Color_White` y `Color_Yellow`. Cada una tiene 10 puntos.

> [!warning] Nota de precisión: observaciones frente a categorías
> Los 10 elementos son **observaciones** (filas), no categorías. Hay **8 categorías distintas**, porque White y Black se repiten. Por eso salen 8 columnas y 10 filas. La misma confusión reaparece en los ejemplos de binary encoding y feature hashing.

La codificación one-hot es una buena opción para características con un conjunto pequeño de categorías y para algoritmos de ML no basados en árboles, como la regresión lineal, k-NN y las redes neuronales. Estos algoritmos se verán en el próximo capítulo.

> [!note] Recuadro del libro: dimensionalidad y cardinalidad
> La codificación one-hot aumenta la dimensionalidad del dataset de tus características. Esto puede volverse un problema, sobre todo con características de **alta cardinalidad**, es decir, con muchas categorías únicas. Por ejemplo, hay más de 40 000 códigos postales en EE. UU., y un catálogo puede tener cientos de miles de identificadores de producto: one-hot crearía una columna por cada uno. Sin embargo, el impacto depende en gran medida del algoritmo de ML que uses y de la naturaleza de tus datos.
>
> Puedes mitigar el riesgo de alta dimensionalidad aplicando en su lugar técnicas como la codificación binaria, que se explica en la siguiente sección. Otro enfoque es usar el algoritmo de análisis de componentes principales (PCA) después de la codificación one-hot. Consulta el capítulo 4 para más información.

##### Codificación binaria (_Binary Encoding_)

La codificación binaria es una técnica eficaz para manejar datos categóricos sin hacer «explotar» la dimensionalidad del dataset de tus características, como podría hacer la codificación one-hot. Funciona convirtiendo cada valor de categoría en su número binario correspondiente y luego separando ese número binario en bits individuales, cada uno como una característica (columna) aparte.

Piensa en la codificación binaria como una forma de compactar el número de características que resulta de la técnica one-hot.

En el siguiente ejemplo usamos la clase `BinaryEncoder` del módulo `category_encoders` para codificar en binario el mismo dataset del ejemplo anterior (`category_encoders` es una biblioteca externa a scikit-learn; se instala con `pip install category_encoders`):

```python
import category_encoders as ce
import pandas as pd

# Sample data
data = {'Color': ['White', 'Black', 'Red', 'Blue',
'Green', 'Yellow', 'Pink', 'Brown', 'White', 'Black']}
df = pd.DataFrame(data)

# Initialize Binary Encoder
encoder = ce.BinaryEncoder(cols=['Color'])

# Fit and Transform the data
df_binary_encoded = encoder.fit_transform(df)

# Print results
print("Original Data:\n", df)
print("\nBinary Encoded Data:\n", df_binary_encoded)
```

La Figura 3.14 muestra el resultado de esta transformación.

_Figura 3.14 Codificación binaria._

Salida real (con la columna original añadida al lado para leerla mejor; category_encoders 2.11.1):

```text
    Color  Color_0  Color_1  Color_2  Color_3
0   White        0        0        0        1
1   Black        0        0        1        0
2     Red        0        0        1        1
3    Blue        0        1        0        0
4   Green        0        1        0        1
5  Yellow        0        1        1        0
6    Pink        0        1        1        1
7   Brown        1        0        0        0
8   White        0        0        0        1
9   Black        0        0        1        0
```

Como resultado de codificar en binario nuestras 10 categorías no distintas, el nuevo dataset tiene solo cuatro características numéricas (en lugar de ocho).

¿Por qué cuatro? Porque el dataset original tenía ocho categorías distintas, que pueden representarse en formato binario con cuatro bits. Observa que el número de puntos (10) no cambia al transformar los datos.

> [!warning] Nota de precisión: 8 categorías caben en 3 bits
> Con 3 bits se representan $2^3 = 8$ valores (del 0 al 7), así que 8 categorías **caben en 3 bits**. Salen 4 columnas porque `BinaryEncoder` primero numera las categorías **empezando en 1**, en orden de aparición: White = 1, Black = 2… Brown = 8. Como 8 en binario es `1000`, se necesitan 4 bits. En la salida se ve que la columna `Color_0` solo vale 1 para Brown. En general, el número de columnas crece como $\lceil \log_2(k+1) \rceil$ para _k_ categorías: 1000 categorías producen 10 columnas en vez de 1000.

La codificación binaria es la más adecuada para resolver las limitaciones de la codificación one-hot, porque mitiga la «explosión» de dimensionalidad que resulta de crear una característica nueva por cada categoría única.

Sin embargo, comparada con la codificación one-hot, la codificación binaria puede provocar una pérdida de información dentro de los datos categóricos, lo que podría afectar negativamente el rendimiento de tu modelo. Esto se debe a que la codificación binaria comprime los datos categóricos en menos bits, lo que hace que las distinciones entre categorías se difuminen.

> [!warning] Nota de precisión: no se pierde información, se inventa parecido
> En sentido estricto, la codificación binaria **no pierde información**: cada categoría recibe un código único y se puede recuperar exactamente. Lo que ocurre es que introduce **parecidos artificiales**. Red (`0011`) y Pink (`0111`) comparten tres bits y el modelo puede tratarlas como similares, aunque ese parecido sea un accidente de la numeración. Es eso lo que el libro llama «difuminar las distinciones».

##### Hashing de características (_Feature Hashing_)

Esta técnica usa una **función hash** para convertir datos categóricos de alta cardinalidad en un número fijo de características numéricas que tú, como ingeniero de ML, eliges. Una función hash es una función determinista que convierte cualquier entrada (por ejemplo, el texto «Green») en un número entero grande. La misma entrada siempre produce el mismo número, pero entradas distintas pueden producir el mismo número, lo que se llama una **colisión**. Se parece al `hash()` de Python, pero es estable entre ejecuciones.

El hashing de características es eficiente y muy escalable porque el resultado de la transformación es un vector de tamaño fijo, lo que mantiene bajo control el uso de memoria. A grandes rasgos, funciona así:

- **Característica de entrada.** La función hash recibe un valor de la característica, que puede ser una palabra, una categoría o cualquier otro atributo identificable de tus datos.
- **Aplicación de la función hash.** La función hash procesa la característica y devuelve un valor entero único (valor hash) basado en el valor de la característica.
- **Operación módulo (opcional).** Para que el valor hash caiga dentro del rango deseado (normalmente el tamaño del vector de características), a menudo se toma el valor hash módulo un número predefinido.
- **Actualización del vector.** El valor hash calculado se usa como índice para actualizar el elemento correspondiente del vector de características.

Al aprovechar una función hash, este enfoque transforma los datos rápidamente, lo que lo hace adecuado para datasets grandes. Aunque pueden ocurrir colisiones, la agregación de valores suele mitigar su impacto, sobre todo con contenedores hash (_hash buckets_) lo bastante grandes.

Por lo tanto, el hashing de características logra un equilibrio entre preservar la información y mantener la eficiencia computacional. Es una solución excelente para construir características a partir de datos categóricos de alta cardinalidad de forma eficiente y económica. Es económica porque no necesita guardar un diccionario «categoría → columna», que con millones de categorías sería enorme, y porque una categoría nueva que aparezca en producción se procesa sin reentrenar el codificador.

Puedes usar la clase `FeatureHasher` del módulo `sklearn.feature_extraction` para aplicar hashing a tus características con este enfoque, como se muestra en el siguiente fragmento:

```python
import pandas as pd
from sklearn.feature_extraction import FeatureHasher

# Sample data
data = [{'Color': 'White'}, {'Color': 'Black'}, {'Color':
'Red'}, {'Color': 'Blue'}, {'Color': 'Green'}, {'Color':
'Yellow'}, {'Color': 'Pink'}, {'Color': 'Brown'},
{'Color': 'White'}, {'Color': 'Black'}]

# Convert to DataFrame for viewing before transformation
df = pd.DataFrame(data)

print("Original Data:")
print(df)

# Initialize FeatureHasher
hasher = FeatureHasher(n_features=3, input_type='dict')

# Fit and transform the data
hashed_features = hasher.transform(data)

# Convert the hashed features to a DataFrame for easier viewing
hashed_df = pd.DataFrame(hashed_features.toarray(),
columns=[f'feature_{i}' for i in
range(hashed_features.shape[1])])

print("\nHashed Data:")
print(hashed_df)
```

La Figura 3.15 muestra las características hash generadas por este programa.

_Figura 3.15 Hashing de características._

Salida real (parte _Hashed Data_, con la columna `Color` añadida al lado para leerla):

```text
   feature_0  feature_1  feature_2   Color
0        0.0       -1.0        0.0   White
1        0.0        1.0        0.0   Black
2       -1.0        0.0        0.0     Red
3        0.0       -1.0        0.0    Blue
4        0.0        0.0       -1.0   Green
5        0.0        0.0        1.0  Yellow
6        0.0        0.0       -1.0   Pink
7        0.0        0.0       -1.0   Brown
8        0.0       -1.0        0.0   White
9        0.0        1.0        0.0   Black
```

Observa cómo la clase `FeatureHasher` aplica el hashing de características para convertir los datos categóricos (10 categorías) en un vector de tamaño fijo para cada categoría. En este ejemplo, fijamos el número de características en el constructor de la clase `FeatureHasher`. Ese número se fijó en 3.

Como resultado, las 10 categorías se convirtieron en 10 vectores. Cada vector está formado por tres enteros, uno por cada una de las tres características.

> [!warning] Nota de precisión: el hash no es único y las colisiones son seguras aquí
>
> - El texto dice que la función hash devuelve un valor «único». No lo es: es **determinista** (misma entrada, mismo valor), pero entradas distintas pueden coincidir. Con 8 categorías y solo 3 columnas, las colisiones son inevitables por el principio del palomar. En la salida real, **White y Blue** reciben exactamente el mismo vector, y **Green, Pink y Brown** también. El modelo no podrá distinguirlas. En la práctica se usan muchas más columnas, del orden de $2^{10}$ a $2^{20}$.
> - El módulo no es realmente «opcional»: siempre hace falta para que el índice caiga dentro del vector.
> - `FeatureHasher` usa por defecto un **signo** derivado de otro hash (`alternate_sign=True`), por eso aparecen −1 y +1. El signo hace que, al sumarse, las colisiones tiendan a cancelarse en promedio. Los valores son números de punto flotante, no enteros.
> - Otra vez, son 10 **observaciones** de 8 categorías distintas.

#### Ingeniería de características para series de tiempo (_Feature Engineering for Time-Series Data_)

Los datos de series de tiempo se presentaron al comienzo de este capítulo. Se diferencian de los datos tabulares estándar porque se capturan repetidamente a lo largo del tiempo y cada punto sucesivo depende de sus valores pasados. Piensa en una serie de tiempo como en una película: cada escena depende de las anteriores. La dimensión temporal tiene un papel clave en las series de tiempo.

Amazon SageMaker Data Wrangler ofrece una solución _low-code_ para procesar series de tiempo, con la capacidad de limpiar, transformar y preparar datos más rápido. _Low-code_ significa que la mayor parte del trabajo se hace eligiendo y configurando pasos en una interfaz visual, con la opción de escribir código solo donde haga falta; _no-code_ sería no escribir nada. También permite a los científicos de datos construir las características de las series de tiempo respetando los requisitos de formato de entrada de su modelo de pronóstico.

Para hacer ingeniería de características sobre un dataset de series de tiempo, primero necesitamos entender los patrones presentes en el dataset. Amazon SageMaker Data Wrangler ofrece varias visualizaciones que dan a científicos y analistas de datos pistas valiosas sobre los patrones existentes y que pueden ayudarte a elegir una estrategia de modelado. Una vez que entendemos los patrones del dataset, podemos empezar a construir características nuevas orientadas a aumentar la precisión de los modelos de pronóstico.

##### Descomponer la fecha y hora (_Featurize Datetime_)

Es buena práctica empezar la ingeniería de características de series de tiempo separando las características de fecha y hora. Estas características se crean a partir de la columna de marca de tiempo y son un punto de partida óptimo para el ingeniero de ML. La transformación de series de tiempo _Featurize datetime_ permite descomponer una característica de fecha y hora agregando al dataset las nuevas características `date_month`, `date_day`, `date_week_of_year`, `date_day_of_year` y `date_quarter`. Al proporcionar los componentes de la fecha y hora como características separadas, permitimos que los algoritmos de ML detecten patrones que mejoran la precisión de las predicciones.

La documentación vigente de Data Wrangler (dentro de Canvas) agrega un detalle útil: la transformación ofrece un **modo de codificación** _ordinal_ o _cyclic_. Recomienda _cyclic_ para modelos lineales y redes profundas, y _ordinal_ para algoritmos basados en árboles. La codificación cíclica representa, por ejemplo, el mes con un par seno/coseno, de modo que diciembre (12) quede cerca de enero (1). Con la codificación ordinal, un modelo lineal vería esos dos meses como los más alejados.

##### Codificar como categórica (_Encode Categorical_)

Las características de fecha y hora no se limitan a valores enteros. También puedes considerar ciertas características de fecha y hora extraídas como variables categóricas y representarlas como características one-hot, con cada columna conteniendo valores binarios. Para ello, usa la transformación _One-hot encode_. La nueva característica `date_quarter` contiene valores de 0 a 3 y puede codificarse con one-hot en cuatro columnas binarias, una por cada trimestre.

> [!warning] Nota de precisión: rango de `date_quarter`
> No pude confirmar en la documentación vigente si Data Wrangler numera los trimestres de 0 a 3. pandas, por ejemplo, usa 1 a 4 (`Series.dt.quarter`). Para la codificación one-hot da igual: son cuatro columnas en ambos casos.

##### Característica de rezago (_Lag Feature_)

Para aumentar la precisión del modelo, es buena práctica crear características de rezago (_lag features_) para la variable objetivo (o etiqueta). Las características de rezago son valores en marcas de tiempo anteriores que ayudan a predecir valores futuros. También ayudan a identificar patrones de autocorrelación (también llamada correlación serial) en la serie de residuos, al cuantificar la relación de cada observación con las observaciones de pasos de tiempo anteriores. La autocorrelación es similar a la correlación normal, pero entre los valores de una serie y sus propios valores pasados. Amazon SageMaker Data Wrangler ofrece la transformación _Lag features_, que ayuda a crear múltiples características de rezago sobre un tamaño de ventana especificado.

Un rezago solo puede usar valores **anteriores** al momento de la predicción. Un rezago mal alineado que «vea» el valor actual o uno futuro es una forma clásica de fuga de datos en series de tiempo. La documentación de Data Wrangler menciona dos usos típicos: tomar los últimos valores ($t, t-1, t-2, \ldots$) y tomar valores estacionales, por ejemplo la misma hora del día anterior.

##### Características de ventana móvil (_Rolling Window Features_)

Amazon SageMaker Data Wrangler implementa capacidades automáticas de extracción de características de series de tiempo usando el paquete de código abierto `tsfresh`. Con la transformación de extracción de características de series de tiempo, puedes automatizar el proceso de extracción. Esto elimina el tiempo y el esfuerzo que de otro modo dedicarías a implementar manualmente bibliotecas de procesamiento de señales. Para el examen, necesitas saber que las características pueden extraerse con la transformación _Rolling window features_, que calcula propiedades estadísticas sobre un conjunto de observaciones definido por el tamaño de la ventana. Por ejemplo, con una ventana de 3, a la fila del instante _t_ se le agregan estadísticas (media, mínimo, máximo, etc.) calculadas sobre $t-3$, $t-2$ y $t-1$.

> [!warning] Nota de precisión: `tsfresh`
> La página vigente de transformaciones de Data Wrangler que revisé describe _Rolling window features_ y _Extract features_, pero no menciona `tsfresh` por nombre. La afirmación del libro puede venir de material de AWS anterior. No la contradigo, pero no pude confirmarla en la documentación actual.

Después de hacer la ingeniería de características del dataset de series de tiempo, estamos listos para usar el dataset transformado como entrada de un algoritmo de ML de pronóstico.

### Técnicas para datos no estructurados (_Techniques for Unstructured Data_)

Los datos no estructurados, como imágenes, texto, audio y video, no caben fácilmente en tablas y a menudo carecen de un formato predefinido. AWS ofrece un ecosistema completo de productos y servicios que hacen más manejable y eficiente la ingeniería de características para datos no estructurados.

En las próximas secciones cubriremos las técnicas que necesitas conocer para el examen y que se aplican a datos de imagen y de texto.

#### Ingeniería de características para datos de imagen (_Feature Engineering for Image Data_)

Igual que con los datos estructurados, la ingeniería de características para imágenes es un paso crítico en el desarrollo de modelos de ML eficaces. Este proceso consiste en extraer de las imágenes características significativas, como bordes, texturas y colores, para mejorar el rendimiento de las tareas de reconocimiento y clasificación de imágenes, proporcionando al modelo información más relevante sobre los datos.

Por ejemplo, al comienzo del capítulo usamos la imagen de un auto (Figura 3.4) para mostrarte qué características podían extraerse de estos datos no estructurados.

Para extraer características de imágenes, lo principal es aprovechar los modelos de visión preentrenados de **Amazon SageMaker JumpStart**, a menudo procesándolas con una red neuronal convolucional (CNN) y recuperando los vectores de características resultantes. JumpStart es un catálogo, dentro de SageMaker, de modelos ya entrenados por AWS y por terceros que puedes desplegar o ajustar (_fine-tune_) con pocos clics o pocas líneas de código, sin entrenarlos desde cero. Una **red neuronal convolucional** (_convolutional neural network_, CNN) es una red diseñada para imágenes: sus primeras capas aplican pequeños filtros que detectan bordes y texturas, y las capas siguientes combinan esos patrones en formas cada vez más complejas. Si se quita la última capa, la que decide «gato» o «perro», la salida de la penúltima es un **vector de características** (también llamado _embedding_) de cientos o miles de números que resume la imagen. Ese vector puede usarse como entrada tabular de otro modelo. Estos modelos de visión preentrenados incluyen modelos de detección de objetos, clasificación de imágenes y muchos otros.

También puedes usar servicios como **Amazon Rekognition** para el análisis básico de imágenes y la extracción de características, según tu caso de uso. Rekognition es un servicio de visión por computadora ya entrenado que se usa mediante una API: le envías una imagen o un video y te devuelve lo que detectó, sin que entrenes ni administres ningún modelo. Amazon Rekognition puede detectar objetos, escenas, texto y rostros en imágenes, y proporciona características de alto nivel sin necesidad de un gran esfuerzo manual. Por ejemplo, puede identificar y extraer características como etiquetas y cuadros delimitadores (**_bounding boxes_**) alrededor de los objetos, que son esenciales para tareas como la detección de objetos y la clasificación de imágenes. Un bounding box es el rectángulo, descrito por sus coordenadas dentro de la imagen, que encierra un objeto detectado. Rekognition devuelve, por ejemplo, «Car, confianza 98 %, en el rectángulo que empieza en (0.12, 0.40) con ancho 0.35 y alto 0.20», y cada uno de esos datos puede convertirse en una columna de tu dataset.

Una vez extraídas las características, es esencial transformarlas a un formato adecuado. Amazon SageMaker Data Wrangler te permite hacer análisis exploratorio de datos (EDA) y aplicar transformaciones como la normalización y la reducción de dimensionalidad. Este paso asegura que tus características estén optimizadas para el entrenamiento. Por ejemplo, puedes normalizar los valores de los píxeles a una escala común o aplicar PCA para reducir la dimensionalidad de tu conjunto de características. Los píxeles de una imagen de 8 bits van de 0 a 255, así que la normalización típica es dividir entre 255. Una foto modesta de 224 × 224 píxeles en color ya tiene 224 × 224 × 3 ≈ 150 000 valores, lo que muestra por qué la reducción de dimensionalidad importa tanto en imágenes.

Almacenar y gestionar de forma eficiente tus características construidas es crucial para tener flujos de trabajo ágiles. Como aprendiste al comienzo del capítulo, Amazon SageMaker Feature Store proporciona un repositorio centralizado de características, que facilita compartirlas y reutilizarlas entre distintos modelos y proyectos. Esto asegura la consistencia y ahorra tiempo, porque no tienes que volver a extraer las características para cada modelo nuevo.

#### Ingeniería de características para datos textuales (_Feature Engineering for Textual Data_)

En el mundo actual, impulsado por los datos, el texto está en todas partes, desde las publicaciones en redes sociales hasta las reseñas de clientes y más. Extraer conocimiento significativo de estos datos no estructurados requiere una ingeniería de características eficaz. AWS ofrece un ecosistema potente de productos y servicios para agilizar este proceso y convertir texto crudo en características valiosas para modelos de ML.

**Amazon Comprehend** es una herramienta potente para extraer características de alto nivel a partir de texto. Es un servicio de NLP ya entrenado que se usa por API, igual que Rekognition, pero para texto. Puede detectar entidades, frases clave, sentimiento e idioma, lo que proporciona información rica sin un gran esfuerzo manual. Las _entidades_ son menciones de cosas con nombre propio o tipo reconocible: personas, lugares, organizaciones, fechas, cantidades. Cada resultado, como «sentimiento = NEGATIVE con 0.91 de confianza», puede convertirse directamente en una característica. Además, **Amazon Textract** puede extraer texto de documentos, lo que convierte datos no estructurados en información estructurada. Textract es un servicio de reconocimiento óptico de caracteres (OCR), la tecnología que convierte la imagen de un texto (un escaneo, una foto, un PDF sin texto seleccionable) en texto de computadora. Además del texto, Textract reconoce la estructura del documento: pares clave-valor de formularios («Nombre: Ana») y tablas.

Para una extracción de características más personalizada, Amazon SageMaker ofrece algoritmos y frameworks integrados. Estos algoritmos pueden usarse para entrenar modelos y extraer características específicas, como _word embeddings_ (p. ej., Word2Vec), o para modelar tópicos (p. ej., _Latent Dirichlet Allocation_, LDA). Word2Vec es un método que aprende un vector numérico por palabra a partir de los contextos en que aparece; se explica más abajo, en «Word embeddings». LDA es el modelo probabilístico que representa cada documento como una mezcla de tópicos. Estos algoritmos se verán en detalle en el capítulo 4.

Las siguientes son algunas técnicas esenciales que necesitas conocer para el examen.

##### Tokenización (_Tokenization_)

La tokenización es el proceso de dividir el texto en unidades más pequeñas, como palabras o frases, llamadas **tokens**. Es el primer paso para preparar datos de texto para el modelado.

Por ejemplo, la oración «Machine learning is fascinating» se tokeniza como `["Machine", "learning", "is", "fascinating"]`.

La tokenización es un componente clave que usan los modelos fundacionales (FM, _foundation models_) de **Amazon Bedrock** para entender y procesar los _prompts_ de los usuarios. Un **modelo fundacional** es un modelo muy grande, preentrenado con cantidades masivas de datos generales, que sirve de base para muchas tareas distintas sin entrenarse de nuevo para cada una. Los grandes modelos de lenguaje (LLM) y los modelos que generan imágenes a partir de texto son ejemplos. **Amazon Bedrock** es el servicio _fully managed_ de AWS que da acceso por API a modelos fundacionales de varios proveedores: tú envías una petición y recibes la respuesta, sin desplegar ni administrar servidores. El **prompt** es la instrucción o el texto de entrada que le das al modelo. Al descomponer el texto de entrada en tokens más pequeños, estos modelos pueden interpretar eficazmente el significado y el contexto de la petición del usuario. Por ejemplo, un prompt como «Generate an image of a sunset» se dividiría en palabras y símbolos individuales, lo que permite al modelo analizar cada elemento y entender la instrucción en su conjunto. Este enfoque basado en tokens asegura que el modelo pueda manejar instrucciones complejas y con matices con alta precisión y relevancia.

Además, la tokenización tiene un papel clave en el modelo de precios de los FM de Amazon Bedrock. El costo de usar estos modelos se determina por el número de tokens de entrada procesados y de tokens de salida generados. Este sistema de cobro por tokens asegura que a los usuarios se les cobre de forma justa según los recursos computacionales que realmente consumen. Por ejemplo, un prompt más largo y detallado requeriría más tokens y, por lo tanto, tendría un costo mayor que uno más corto y simple. En el capítulo 4 presentaremos un ejemplo práctico que muestra cómo generar una imagen a partir de un prompt sencillo usando un FM disponible en Amazon Bedrock, lo que ilustra la eficiencia y la eficacia del procesamiento basado en tokens.

Para dar escala al costo: una regla práctica muy difundida es que un token equivale, en promedio, a unos tres cuartos de palabra en inglés. El español suele requerir algo más de tokens por palabra. La cifra exacta depende del tokenizador de cada modelo, así que tómala solo como orden de magnitud. Una página de texto de unas 500 palabras rondaría entonces los 650 a 700 tokens. Los precios de Bedrock suelen expresarse por cada 1000 tokens (o por millón de tokens), con tarifas distintas para entrada y salida.

> [!warning] Nota de precisión: tokens de los LLM y cobro de Bedrock (verifica en la página de precios vigente)
>
> - Los modelos fundacionales actuales no tokenizan por palabras completas, sino en **subpalabras**, con algoritmos como BPE o SentencePiece. Una palabra común puede ser un solo token, y una palabra rara o en otro idioma puede partirse en varios («fascinating» podría quedar como «fascin» + «ating»). El ejemplo de dividir el prompt en «palabras individuales» es una simplificación.
> - El cobro por tokens de entrada y salida aplica a los modelos **de texto** en modalidad bajo demanda. Los modelos que **generan imágenes**, como el del ejemplo del capítulo 4, se cobran típicamente **por imagen generada** (según resolución y calidad), no por tokens. Además existen otras modalidades de precio, como el _throughput_ aprovisionado (capacidad reservada que se cobra por hora) y la inferencia por lotes.

##### Eliminación de palabras vacías (_Stop-Words Removal_)

Las **palabras vacías** (_stop words_) son palabras comunes (p. ej., «the», «is», «and») que a menudo no aportan información significativa. Eliminarlas puede reducir el ruido y mejorar el rendimiento del modelo.

Por ejemplo, eliminar «is» y «the» de «The cat is on the mat» daría `["cat", "on", "mat"]`.

El «a menudo» del texto importa. En análisis de sentimiento, algunas listas estándar de stop words incluyen negaciones como «not» o «no», y eliminarlas convierte «not good» en «good», con el significado contrario. Conviene revisar la lista según la tarea.

##### Stemming y lematización (_Stemming and Lemmatization_)

Estas técnicas reducen las palabras a su forma base o raíz. El _stemming_ recorta prefijos y sufijos, mientras que la lematización usa reglas lingüísticas.

Por ejemplo, «running» se convierte en «run».

La diferencia se nota en otros casos. El stemming aplica reglas mecánicas y puede producir raíces que no son palabras: el algoritmo de Porter convierte «studies» en «studi». La lematización consulta un diccionario y la categoría gramatical, así que convierte «studies» en «study» y el adjetivo «better» en «good», algo que ningún recorte de sufijos lograría. A cambio, la lematización es más lenta.

##### N-gramas (_N-grams_)

Los n-gramas son secuencias contiguas de _n_ elementos de un texto dado. Capturan el contexto y las relaciones entre palabras.

Por ejemplo, con bigramas (es decir, secuencias contiguas de dos palabras), la oración «Machine learning is fun» se transforma en `[("Machine", "learning"), ("learning", "is"), ("is", "fun")]`.

##### Word embeddings

Los _word embeddings_, como Word2Vec y GloVe, convierten las palabras en vectores continuos que capturan su significado semántico y preservan las relaciones entre palabras. Esto hace que los datos sean más adecuados para los modelos de ML. Word2Vec aprende los vectores entrenando una red pequeña para predecir una palabra a partir de sus vecinas, o las vecinas a partir de la palabra. GloVe los obtiene factorizando la matriz de coocurrencias de palabras de un corpus grande. Ambos se basan en la idea de que las palabras que aparecen en contextos parecidos tienen significados parecidos. A diferencia de one-hot, donde cada palabra es un vector de ceros con un solo 1 y todas quedan igual de distantes entre sí, aquí la distancia entre vectores refleja la similitud de significado.

Por ejemplo, «King» y «Queen» podrían quedar cerca en el espacio vectorial, lo que refleja su similitud semántica.

## Etiquetado de datos (_Data Labeling_)

En el ámbito del ML, el etiquetado de datos y la ingeniería de características son dos procesos entrelazados y fundamentales para construir modelos eficaces. Ambos transforman los datos crudos en una forma estructurada que los algoritmos de ML pueden entender y de la que pueden aprender.

La ingeniería de características se centra en seleccionar, modificar y crear características nuevas a partir de los datos crudos para mejorar el rendimiento del modelo. El etiquetado de datos, en cambio, consiste en enriquecer los datos crudos con metadatos en forma de etiquetas significativas que denotan las predicciones reales o atributos específicos. Para el aprendizaje supervisado, este paso es fundamental, porque proporciona la **verdad de referencia** (_ground truth_) de la que aprenden los modelos. _Ground truth_ es el valor correcto y verificado de la variable objetivo, contra el que se entrena y se evalúa el modelo. Al etiquetar datos, le das al modelo las «respuestas correctas» de las que aprender y contra las que predecir.

Las etiquetas actúan como punto de referencia y guían el proceso de aprendizaje de los algoritmos. Definen lo que el modelo debe predecir o clasificar. Por ejemplo, en un dataset de imágenes de animales, etiquetar cada imagen como «cat» o «dog» permite al modelo aprender a distinguir entre estos animales.

### Amazon SageMaker Ground Truth

> [!warning] Estado del servicio (verificado el 23-09-2026)
> Según la documentación de AWS, **Amazon SageMaker Ground Truth ya no admite clientes nuevos**. Los clientes existentes pueden seguir usándolo con normalidad y AWS sigue invirtiendo en su seguridad y disponibilidad, pero no planea añadirle funciones. El contenido del libro sigue siendo útil para el examen y para entender cómo se organiza un proceso de etiquetado.

En AWS, el etiquetado de datos se hace con **Amazon SageMaker Ground Truth**, un servicio diseñado específicamente para agilizar y mejorar el proceso de etiquetado. Este servicio combina automatización avanzada con capacidades de intervención humana (**_human-in-the-loop_**) para entregar datasets etiquetados de alta calidad de forma eficiente. _Human-in-the-loop_ significa que en el proceso automático hay un punto donde las personas revisan, corrigen o deciden, sobre todo en los casos en que la máquina tiene poca confianza. Este es el proceso a grandes rasgos:

- **Almacenar los datos.** Sube tus datos crudos (imágenes, texto, etc.) a Amazon S3. Crea carpetas o buckets separados para organizar tus datos según el tipo o el proyecto. En S3 las «carpetas» son en realidad prefijos de la clave del objeto, como `proyecto-a/imagenes/`, que la consola muestra como si fueran carpetas.
- **Crear un trabajo de etiquetado.** En Amazon SageMaker Ground Truth, define la tarea y las instrucciones.
- **Etiquetado automatizado.** Elige el tipo de datos que vas a etiquetar (p. ej., clasificación de imágenes, clasificación de texto, detección de objetos) y especifica el dataset de entrada almacenado en S3. Selecciona _Automated Data Labeling_ para usar modelos de ML preentrenados que etiqueten un subconjunto de tus datos, y configura los ajustes y parámetros del algoritmo de etiquetado.
- **Revisión humana.** Los anotadores revisan y corrigen las etiquetas para asegurar su precisión. Este paso puede hacerlo Amazon Mechanical Turk, una fuerza de trabajo privada o proveedores externos. **Amazon Mechanical Turk (MTurk)** es un mercado de _crowdsourcing_ que conecta a las empresas con una fuerza de trabajo global de personas que completan tareas difíciles para las computadoras; el etiquetado de datos es una de ellas. _Crowdsourcing_ significa repartir una tarea entre una multitud de personas externas, cada una de las cuales hace una pequeña parte a cambio de un pago por tarea. Una **fuerza de trabajo privada** son tus propios empleados o contratistas, que acceden a un portal de etiquetado; es la opción cuando los datos no pueden salir de la organización. Los **proveedores externos** son empresas especializadas en etiquetado que se contratan a través de AWS Marketplace, la tienda de software y servicios de terceros de AWS.
- **Etiquetado y validación de datos.** Amazon SageMaker Ground Truth ofrece una interfaz fácil de usar para que los etiquetadores anoten los datos. Los anotadores revisan y corrigen las etiquetas automáticas, y agregan etiquetas, cuadros delimitadores u otras anotaciones relevantes según sea necesario. Incluso puedes implementar medidas de control de calidad, como el etiquetado por consenso (varios anotadores etiquetan el mismo dato) y el muestreo de auditoría (expertos revisan un subconjunto de las etiquetas).
- **Almacenar los datos etiquetados.** Puedes conservar los datos etiquetados en tu bucket de S3, o guardarlos automáticamente en Amazon SageMaker Feature Store para tenerlos a mano durante el entrenamiento y la inferencia. En cualquier caso, asegúrate de mantener los datos etiquetados versionados para rastrear los cambios. En S3, la forma más directa de lograrlo es activar el **versionado del bucket** (_S3 Versioning_): cada vez que se sobrescribe o borra un objeto, S3 conserva la versión anterior y se puede recuperar cualquier versión pasada.
- **Entrenar el modelo.** Usa los datos etiquetados para entrenar tus modelos en Amazon SageMaker.

> [!warning] Nota de precisión: cómo funciona realmente el etiquetado automatizado (según la documentación de AWS)
>
> - _Automated data labeling_ no parte de «modelos preentrenados» que etiquetan primero. Usa **aprendizaje activo** (_active learning_). Primero, **personas** etiquetan una muestra aleatoria. Con esas etiquetas, Ground Truth entrena un modelo y calcula un umbral de confianza. El modelo etiqueta automáticamente solo los objetos que superan ese umbral, y los de baja confianza vuelven a las personas. El ciclo se repite hasta etiquetar todo el dataset. El orden real es, entonces, humanos → modelo → humanos para los casos dudosos, y no automático → revisión humana como sugiere la lista.
> - Solo está disponible para cuatro tipos de tarea integrados: clasificación de imágenes (una etiqueta), segmentación semántica, detección de objetos con cuadros delimitadores y clasificación de texto (una etiqueta). Exige un mínimo de **1250 objetos**, y AWS recomienda al menos **5000**. Verifica estos valores en la documentación vigente.
> - La salida documentada de Ground Truth es un **archivo de manifiesto de salida** en S3: un archivo con una línea JSON por objeto, que contiene la ubicación del dato y su etiqueta. No encontré documentada una opción para guardar los resultados «automáticamente» en Feature Store; tómalo como algo que harías con un paso adicional.
> - AWS advierte no compartir información confidencial, datos personales ni datos de salud protegidos con la fuerza de trabajo de Mechanical Turk. De hecho, Ground Truth **exige** declarar que los datos están libres de información personal identificable (PII) para usarla; si no se declara, el trabajo falla. Para esos datos corresponde una fuerza de trabajo privada o un proveedor.
> - **Amazon Mechanical Turk cierra definitivamente el 30 de septiembre de 2026**, según el aviso de la documentación de AWS. A partir de esa fecha, las opciones de fuerza de trabajo que quedan son la privada y la de proveedores.

Con la combinación adecuada de automatización e intervención humana, Amazon SageMaker Ground Truth ofrece una solución robusta para crear datasets etiquetados precisos y confiables, que son cruciales para construir modelos de ML de alto rendimiento.

## Gestión del desbalance de clases (_Managing Class Imbalance_)

Hasta aquí aprendiste a transformar tu dataset limpiando primero los datos crudos, luego haciendo la ingeniería de características y, por último, etiquetando los datos. Este último paso es un aspecto fundamental del ML, porque se centra en proporcionar la predicción correcta para cada punto de datos, de modo que tu modelo pueda aprender rápidamente durante el entrenamiento.

Con tu problema de ML planteado y tus datos cuidadosamente procesados, pensarías que ya estás listo para elegir un algoritmo de ML y comenzar el entrenamiento, ¿verdad? ¡Todavía no!

Tu dataset puede verse bien sintácticamente como resultado de seleccionar, extraer y crear características, pero no semánticamente, por una distribución desigual de las etiquetas. Por ejemplo, en un caso de uso de salud, considera un dataset donde la clase mayoritaria representa a personas sanas y la clase minoritaria a pacientes con una enfermedad rara. El objetivo de tu problema de ML es predecir la enfermedad con precisión. Cuando tu dataset presenta una proporción desigual de etiquetas, como en este escenario donde la clase mayoritaria es la de las personas sanas, estás ante un **desbalance de clases** (_class imbalance_).

El desbalance de clases produce un modelo sesgado hacia la clase mayoritaria. Esto puede afectar significativamente el rendimiento y la equidad del modelo, sobre todo cuando la clase minoritaria es de importancia crítica, como en nuestro caso de salud. El razonamiento implícito es el siguiente: si el 99 % de las personas está sana, un modelo que responda siempre «sano» tiene 99 % de exactitud (_accuracy_) y no detecta a ningún enfermo. El algoritmo, al minimizar el error promedio, encuentra rentable ignorar la clase minoritaria.

La buena noticia es que AWS ofrece un conjunto completo de productos y servicios para abordar el desbalance de clases, de modo que tus modelos no solo sean precisos, sino también éticos e imparciales.

Antes de ver cómo aborda AWS este desafío, repasemos qué enfoques puedes seguir para tratar el desbalance de clases y los sesgos que produce.

### Técnicas de mitigación del desbalance de clases (_Class-Imbalance Mitigation Techniques_)

El **aumento de datos** (_data augmentation_) se considera en general una buena práctica para generar más datos etiquetados de la clase minoritaria, y así reducir la brecha entre las clases mayoritaria y minoritaria. Técnicas como el aumento de imágenes (rotaciones, volteos y ajustes de color) para datos de imagen, o el aumento de texto (reemplazo de sinónimos, inserción aleatoria) para datos textuales, pueden crear muestras diversas y representativas.

Sin embargo, el aumento de datos no siempre es viable por el costo, la disponibilidad de recursos de cómputo y las restricciones de tiempo. En esos casos, considera las siguientes opciones:

- **Sobremuestreo (_oversampling_).** Esta técnica consiste en aumentar el número de puntos de la clase minoritaria duplicando ejemplos existentes o generando ejemplos sintéticos. La técnica de sobremuestreo sintético de la minoría (**SMOTE**, _synthetic minority over-sampling technique_) es un método popular que crea muestras sintéticas interpolando entre ejemplos existentes de la clase minoritaria. SMOTE elige un ejemplo minoritario y uno de sus _k_ vecinos más cercanos de la misma clase, y crea un punto nuevo en algún lugar del segmento que los une. Esto ayuda a balancear el dataset sin simplemente duplicar registros. El sobremuestreo se aplica solo al conjunto de entrenamiento, después de dividir los datos. Si se aplica antes, puntos sintéticos derivados de ejemplos de prueba acaban en entrenamiento, lo que es fuga de datos.
- **Submuestreo (_undersampling_).** Esta técnica consiste en eliminar aleatoriamente puntos de la clase mayoritaria, de modo que la distribución de clases quede más pareja con la clase minoritaria. Aunque puede ser eficaz, el submuestreo puede provocar la pérdida de información valiosa, lo que podría afectar el rendimiento del modelo.
- **Ponderación de clases (_class weighting_).** Al calcular la pérdida durante el entrenamiento, esta técnica asigna al algoritmo pesos más altos por clasificar mal la clase minoritaria, lo que obliga al modelo a prestarle más atención. En scikit-learn corresponde al parámetro `class_weight="balanced"`, y en XGBoost a `scale_pos_weight`.

> [!note] Recuadro del libro: aumento de datos frente a datos sintéticos
> El aumento de datos es distinto de los datos sintéticos. El aumento crea artificialmente datos nuevos a partir de los datos de entrenamiento existentes, mientras que los datos sintéticos son datos generados que no usan el dataset original.

### Amazon SageMaker Clarify

> [!warning] Estado del servicio (verificado el 23-09-2026)
> Según la documentación de AWS, **Amazon SageMaker Clarify ya no admite clientes nuevos**. Los clientes existentes pueden seguir usándolo, pero AWS no planea añadirle funciones. Como reemplazo, AWS recomienda calcular las mismas métricas de sesgo (CI, DPL, etc.), que son fórmulas estándar, con pandas y scikit-learn dentro de tus propios pipelines; usar directamente la biblioteca **SHAP**, que es el motor que Clarify usa internamente, para la explicabilidad; y usar **Amazon Bedrock Evaluations** para evaluar modelos fundacionales. Las fórmulas siguen documentadas y el examen puede preguntarlas.

Amazon SageMaker Clarify ayuda a abordar el desbalance de clases proporcionando métricas de sesgo previas al entrenamiento (_pre-training bias metrics_) para detectar y mitigar el sesgo en tus datasets. **Clarify** es la herramienta de SageMaker para medir sesgos en los datos y en las predicciones y para explicar qué características pesan en cada predicción. Se ejecuta como un trabajo de procesamiento que lee tus datos de S3 y escribe un informe. Cada métrica corresponde a una noción distinta de equidad (_fairness_).

> [!warning] Nota de precisión: Clarify mide, tú mitigas
> Clarify **detecta y cuantifica** el sesgo. La mitigación (remuestrear, ponderar, recolectar más datos) la decides y aplicas tú con las técnicas de la sección anterior.

Estas métricas son críticas para asegurar que tus modelos sean justos e imparciales desde el principio. La Tabla 3.1 muestra dos de las métricas más usadas: el desbalance de clases (CI, _Class Imbalance_) y la diferencia en proporciones de etiquetas (DPL, _Difference in Proportions of Labels_). Sus fórmulas, que el libro no incluye, son, según la documentación de AWS:

$$\text{CI} = \frac{n_a - n_d}{n_a + n_d} \qquad\qquad \text{DPL} = q_a - q_d$$

donde $n_a$ y $n_d$ son el número de registros de las facetas _a_ y _d_ (definidas más abajo), y $q_a = n_a^{(1)}/n_a$ y $q_d = n_d^{(1)}/n_d$ son la proporción de etiquetas positivas dentro de cada faceta. CI solo cuenta cuántos registros hay de cada grupo. DPL compara qué fracción de cada grupo recibió el resultado favorable, como una diferencia de proporciones entre dos grupos.

**Tabla 3.1** Ejemplos de métricas de sesgo previas al entrenamiento.

| Métrica de sesgo                                                                     | Interpretación                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Desbalance de clases (_Class Imbalance_, CI)                                         | Rango normalizado: [−1, +1]<br>0: sin desbalance de clases<br>+1: desbalance completo hacia la clase mayoritaria<br>−1: desbalance completo hacia la clase minoritaria                                                                                                                                                                           |
| Diferencia en proporciones de etiquetas (_Difference in Proportions of Labels_, DPL) | Rango para etiquetas de faceta normalizadas binarias y multicategoría: [−1, +1]<br>Rango para etiquetas de faceta continuas: [−∞, +∞]<br>0: igual proporción de resultados positivos entre facetas<br>+1: la faceta _a_ tiene la mayor proporción de resultados positivos<br>−1: la faceta _d_ tiene la mayor proporción de resultados positivos |
| Diferencia de igualdad de oportunidades (_Equal opportunity difference_, EOD)        | _(sin interpretación en el original)_                                                                                                                                                                                                                                                                                                            |
| Sesgo en detección de objetos (_Object detection bias_)                              | _(sin interpretación en el original)_                                                                                                                                                                                                                                                                                                            |
| Diferencia de paridad predictiva (_Predictive parity difference_, PPD)               | _(sin interpretación en el original)_                                                                                                                                                                                                                                                                                                            |
| Sesgo en clasificación de imágenes (_Image classification bias_)                     | _(sin interpretación en el original)_                                                                                                                                                                                                                                                                                                            |

> [!warning] Nota de precisión: la Tabla 3.1
>
> - **Filas finales.** Las cuatro últimas filas aparecen en el `.txt` sin interpretación. Ninguna figura entre las métricas previas al entrenamiento de Clarify. Según la documentación, estas son ocho: CI, DPL, KL (divergencia de Kullback-Leibler), JS (Jensen-Shannon), LP (norma Lp), TVD (distancia de variación total), KS (Kolmogórov-Smirnov) y CDD (disparidad demográfica condicional). EOD y PPD son nombres de métricas de equidad **posteriores** al entrenamiento en la literatura, porque requieren predicciones del modelo. Las filas de imágenes parecen restos de otra tabla. Las conservo por fidelidad al original, pero no las tomes como métricas de Clarify.
> - **Interpretación de CI.** La documentación no habla de «clase mayoritaria o minoritaria», sino de **facetas**: +1 significa que el dataset solo contiene miembros de la faceta _a_, y −1 que solo contiene miembros de la faceta _d_. CI mide cuánta representación tiene cada **grupo** en los datos, no la proporción de etiquetas.

En el contexto de los sesgos por desbalance de clases con Amazon SageMaker Clarify, una **faceta** (_facet_) denota una característica específica de tu dataset que quieres analizar en busca de posibles sesgos.

Las facetas se usan para identificar y medir cómo el desbalance de clases puede afectar a distintos subgrupos de tus datos. En Amazon SageMaker Clarify, la faceta _a_ denota el valor de la característica que define al grupo demográfico que el sesgo favorece, y la faceta _d_ el valor que define al grupo demográfico que el sesgo desfavorece. En términos estadísticos, una faceta es una variable de agrupación y cada valor define un estrato. Los ejemplos típicos de la documentación son atributos sensibles como la edad o el género, por ejemplo «mediana edad» (_a_) frente a «jóvenes y mayores» (_d_).

Por ejemplo, si analizas un dataset de transacciones con tarjeta de crédito, las facetas podrían incluir características como `is_fraudulent`, cuyo valor puede ser 1 (verdadero) o 0 (falso). Supongamos que el 99.9 % de los valores de esta característica son 0, lo que indica que el 99.9 % de las transacciones del dataset no son fraudulentas. Al examinar los valores de esta faceta (1 o 0), Amazon SageMaker Clarify puede ayudarte a entender si ciertos subgrupos de alguno de estos valores están subrepresentados o sobrerrepresentados en tus datos de entrenamiento, lo que podría llevar a predicciones sesgadas del modelo.

Puedes encontrar otras métricas de sesgo previas al entrenamiento en https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-measure-data-bias.html.

Aunque el rango de cada métrica varía, todas tienen en común que un valor 0 (o cercano a 0) indica ausencia de desbalance de clases.

Al calcular estas métricas de sesgo previas al entrenamiento, puedes decidir los siguientes pasos para preparar tu dataset para el entrenamiento. En nuestro ejemplo, suponiendo que la faceta _a_ es 0, probablemente obtendríamos un CI cercano a 1. Por lo tanto, necesitaríamos hacer submuestreo. Con la fórmula: si hay 1 000 000 de transacciones, $n_a$ = 999 000 (valor 0) y $n_d$ = 1000 (valor 1), así que CI = (999 000 − 1000)/1 000 000 = 0.998.

> [!warning] Nota de precisión: el ejemplo del fraude
>
> - `is_fraudulent` es la **variable objetivo**, no un atributo demográfico. Usarla como faceta hace que CI mida simplemente el desbalance de la etiqueta, que se calcula igual con `value_counts()`. El uso previsto de las facetas es otro: comprobar si un **grupo** (por ejemplo, clientes de cierta edad o región) está subrepresentado.
> - «Necesitaríamos hacer submuestreo» es **una** opción, no una consecuencia obligada. Con un 0.1 % de positivos, igualar las clases por submuestreo descartaría el 99.8 % de los datos. Muchas veces se prefieren la ponderación de clases, el sobremuestreo (SMOTE) o un submuestreo parcial combinado con métricas adecuadas para clases raras, como el _recall_, la precisión o el área bajo la curva precisión-recall, en lugar de la exactitud.

La clase `BiasConfig` forma parte de la biblioteca `sagemaker.clarify`. Esta biblioteca te ayuda a detectar y mitigar el sesgo en modelos de ML. La clase `BiasConfig` se usa para configurar los ajustes del análisis de sesgo de tu dataset. `sagemaker.clarify` es un módulo del **SDK de SageMaker para Python**. Un SDK (_software development kit_) es la biblioteca que permite manejar un servicio desde código en vez de hacerlo desde la consola web. En `BiasConfig` se declara, en esencia, qué valor de la etiqueta cuenta como resultado positivo, qué columna es la faceta y qué valor o valores de esa columna definen el grupo a examinar. Los parámetros se llaman, según la documentación del SDK, `label_values_or_threshold`, `facet_name` y `facet_values_or_threshold`; verifica la firma en tu versión. Después, la configuración se pasa a un procesador de Clarify, que ejecuta el análisis como un trabajo en AWS.

## División de datos (_Data Splitting_)

Después de gestionar el desbalance de clases, tus datos por fin están listos para dividirse en los datasets de entrenamiento, validación y prueba.

La división de datos consiste en separar tu dataset en subconjuntos distintos para entrenamiento, validación y prueba. Es un paso crítico en ML para que tu modelo se entrene eficazmente y se evalúe con precisión. Veamos qué debe incluir cada dataset y, sobre todo, cuál es su propósito:

- **Dataset de entrenamiento.** El dataset de entrenamiento se usa para enseñar al modelo. Es donde el modelo aprende los patrones y las relaciones de los datos. Durante el entrenamiento, el modelo usa estos datos para ajustar sus parámetros y aprende iterativamente a minimizar errores y mejorar sus predicciones. Por ejemplo, podrías darle al modelo numerosos ejemplos etiquetados de correos (spam y no spam) para enseñarle a clasificar los correos entrantes.
- **Dataset de validación.** El dataset de validación se usa para evaluar el rendimiento del modelo mientras se ajustan finamente sus hiperparámetros. Los _hiperparámetros_ son las perillas que fijas antes de entrenar, como la profundidad de un árbol o la tasa de aprendizaje, a diferencia de los parámetros que el modelo aprende de los datos. Uno de los beneficios de usar un dataset de validación es evitar que el modelo se sobreajuste, asegurando que generalice bien a datos no vistos.
- **Dataset de prueba.** El dataset de prueba está (y debe estar) formado por datos que el modelo nunca ha visto. Después del entrenamiento y la validación, el rendimiento del modelo se mide sobre el dataset de prueba para obtener una evaluación imparcial de qué tan bien generaliza a datos nuevos. Por ejemplo, podrías crear un dataset de prueba con un conjunto separado de correos que el modelo no ha visto, para probar su precisión al clasificar spam y no spam.

En resumen, el dataset de entrenamiento se usa para construir el modelo, el de validación ayuda a ajustarlo y previene el sobreajuste, y el de prueba proporciona una evaluación imparcial de su rendimiento.

> [!note] Recuadro del libro: la analogía del examen
> Piensa en los datasets de entrenamiento, validación y prueba como etapas de la preparación de un examen importante. El dataset de entrenamiento es como el material de estudio que usas para aprender a fondo la materia, que te permite entender e interiorizar la información. El dataset de validación se parece a los exámenes de práctica que haces para evaluar tu preparación, identificar áreas de mejora y hacer los ajustes necesarios sin que eso influya en tu proceso central de aprendizaje. Por último, el dataset de prueba es como el examen real que presentas en un centro de pruebas con un supervisor: evalúa objetivamente tu conocimiento y desempeño con preguntas que no has visto, para asegurar que estás bien preparado para escenarios del mundo real. Esta metáfora resalta los papeles distintos, pero interconectados, que cada dataset cumple en el desarrollo de un modelo de ML exitoso y confiable.

Para dividir eficazmente los datos para ML (según las buenas prácticas), considera asignar aproximadamente entre el 60 y el 80 % al entrenamiento, para que el modelo aprenda y adquiera una comprensión completa de los datos. Usa entre el 10 y el 20 % para la validación, para ajustar finamente los parámetros del modelo sin influir en el proceso de entrenamiento y así asegurar una evaluación imparcial del rendimiento. El 10–20 % final se reserva para la prueba, que sirve como medida definitiva de la capacidad del modelo para generalizar a datos nuevos y no vistos. Una partición adecuada es un aspecto clave de la ingeniería de características: previene la fuga de datos, evita el sobreajuste y da una medida precisa de qué tan bien funcionará tu modelo de ML en escenarios del mundo real, lo que afecta su confiabilidad y eficacia generales.

> [!note] Recuadro del libro: fuga de datos
> La **fuga de datos** (_data leakage_) ocurre cuando parte de los datos de prueba «se filtra» al dataset de entrenamiento. Es un error común que puede comprometer gravemente la integridad de los modelos de ML. Cuando ocurre, el modelo aprende información a la que no tendría acceso en un escenario real, lo que infla artificialmente las métricas de rendimiento y hace que generalice mal a datos nuevos. Evitar la fuga de datos es crucial para que el rendimiento del modelo se evalúe con precisión y para que pueda generalizar bien en situaciones reales. Entre las técnicas eficaces para prevenirla están mantener una separación clara entre los datasets de entrenamiento y de prueba, usar validación cruzada y vigilar con cuidado los pasos de preprocesamiento. Como buena práctica, normaliza o estandariza siempre tus datos **después** de haberlos dividido en datasets de entrenamiento y de prueba. Si se detecta una fuga de datos, es esencial volver a evaluar el modelo con un dataset correctamente separado para obtener una valoración justa de su rendimiento.

El recuadro da la regla de normalizar después de dividir sin explicar el porqué. Si calculas la media y la desviación estándar (o el mínimo y el máximo) con **todos** los datos antes de dividir, esos números ya contienen información del conjunto de prueba, y el modelo la «ve» indirectamente a través de la escala. Lo correcto es hacer `fit` del escalador **solo con entrenamiento** y aplicar `transform`, con esos mismos parámetros, a validación y prueba, exactamente como ocurrirá en producción con datos que todavía no existen. La misma lógica vale para la media de imputación, los límites del IQR, el vocabulario de un codificador y el sobremuestreo. Las listas de la sección de limpieza aparecen _antes_ que la de división porque presentan los conceptos, no porque el ajuste deba hacerse con todo el dataset. En scikit-learn, un `Pipeline` evaluado con validación cruzada garantiza este orden automáticamente.

AWS ofrece Amazon SageMaker Data Wrangler para ayudarte a dividir tu dataset. Veamos cómo con más detalle.

### Amazon SageMaker Data Wrangler

Además de ser tu herramienta integral (_one-stop shop_, literalmente una tienda donde encuentras todo) para la ingeniería de características, Amazon SageMaker Data Wrangler puede ayudarte a dividir tu dataset en datasets de entrenamiento, validación y prueba con poco o nada de código. Específicamente, puedes usar la transformación _Split data_ basada en estas cuatro técnicas de uso común:

- **División aleatoria (_random split_).** Esta técnica asegura que cada subconjunto (entrenamiento, validación, prueba) tenga una distribución similar de categorías, lo que previene el sesgo hacia alguna clase en particular. Este método es especialmente útil cuando no necesitas preservar el orden de tus datos de entrada.
- **División ordenada (_order split_).** Esta técnica divide los datos preservando el orden, lo que previene la fuga de datos al asegurar que la información pasada o futura no se superponga entre los datasets de entrenamiento, validación y prueba. Es especialmente útil para series de tiempo o cualquier escenario donde el orden de los datos sea crítico. Según la documentación, en una división 80/20 las primeras observaciones (el 80 %) van a entrenamiento y las últimas (el 20 %) a prueba. Con series de tiempo, esto significa entrenar con el pasado y evaluar con el futuro, igual que ocurrirá en producción.
- **División estratificada (_stratified split_).** Esta técnica asegura que cada subconjunto mantenga la misma proporción de categorías que el dataset original. Es especialmente útil en problemas de clasificación con datos desbalanceados, porque ayuda a mantener la distribución de clases en todos los subconjuntos, lo que da lugar a una evaluación y un entrenamiento del modelo más confiables. Con un 0.1 % de fraudes, una división aleatoria podría dejar el conjunto de prueba con muy pocos casos positivos, o ninguno. La estratificación garantiza que cada subconjunto tenga su parte.
- **División por clave (_split by key_).** Esta técnica asegura que ninguna combinación de valores de las columnas de entrada aparezca en más de una de las particiones. Es especialmente útil para evitar la fuga de datos en datos no ordenados. También permite agrupar de forma consistente, manteniendo juntos los datos relacionados, con lo que se preserva la integridad de las particiones. Por ejemplo, si la clave es `customer_id`, todas las transacciones de un mismo cliente quedan en la misma partición. Si no, el modelo podría aprender los hábitos de un cliente en entrenamiento y ser evaluado con ese mismo cliente en prueba, lo que infla las métricas. Es el equivalente de `GroupShuffleSplit` o `GroupKFold` de scikit-learn.

> [!warning] Nota de precisión: la división aleatoria no «asegura» proporciones
> Una división aleatoria produce proporciones de clases **similares en promedio** (en valor esperado), pero no las garantiza, y con clases raras o datasets pequeños pueden desviarse bastante. La documentación de Data Wrangler la describe simplemente como «una muestra aleatoria, sin solapamiento, del dataset original». La técnica que **sí** garantiza las proporciones es la división estratificada.

Después de elegir la técnica de división, puedes especificar los porcentajes para los datasets de entrenamiento, validación y prueba. Por ejemplo, podrías asignar un 70 % al entrenamiento, un 20 % a la validación y un 10 % a la prueba. Por último, aplicas la transformación de división, y Amazon SageMaker Data Wrangler divide automáticamente tu dataset en los subconjuntos especificados. Según la documentación vigente, Data Wrangler usa por defecto una semilla aleatoria para que las divisiones sean reproducibles. También permite fijar un umbral de error que cambia exactitud de las proporciones por velocidad en datasets grandes.

## Escenarios donde este servicio es la opción obligada

Este capítulo no trata de un solo servicio, sino de varios. Por eso cada escenario se centra en uno distinto de los que aparecen en el capítulo, elegido porque, dadas las restricciones del caso, es el único razonable dentro de AWS. Ground Truth y Clarify quedaron fuera a propósito: como ya no admiten clientes nuevos, no pueden ser la opción obligada de un proyecto que empieza hoy. Las empresas son ficticias; las restricciones son las que aparecen en proyectos reales.

### Escenario 1: detección de fraude en tiempo real en una fintech → Amazon SageMaker Feature Store

**Contexto.** Una fintech que emite tarjetas de crédito procesa, en hora pico, unos miles de autorizaciones de pago por segundo. Su equipo de ciencia de datos entrenó un modelo de fraude con dos años de historia, que usa características como «número de compras de esta tarjeta en la última hora», «monto promedio de los últimos 30 días» y «comercios distintos en las últimas 24 horas». El modelo debe puntuar cada pago **antes** de aprobarlo. Los equipos de riesgo crediticio y de marketing quieren reutilizar esas mismas características en sus propios modelos.

**Restricciones clave.**

1. Latencia: el sistema de autorización le da al modelo un presupuesto de unas decenas de milisegundos, lo que incluye leer las características de la tarjeta.
2. Consistencia: las características deben calcularse con la misma lógica para entrenar y para servir. Un desfase ya les costó un modelo que rendía bien en la evaluación y mal en producción.
3. Historia exacta: para reentrenar sin fuga de información, necesitan saber qué valor tenía cada característica **en el instante** de cada transacción pasada. Auditoría interna, además, exige poder reproducirlo.
4. El equipo es de ciencia de datos, no de infraestructura, y la dirección prohibió construir y mantener a mano un sistema de sincronización entre dos bases de datos.
5. Otros equipos deben poder descubrir y reutilizar las características.

**Por qué este servicio.**

- (1) → El **almacén online** conserva el último valor por identificador (aquí, el número de tarjeta) y sirve lecturas con latencia de pocos milisegundos.
- (2) y (4) → Las características se **ingieren una sola vez** y Feature Store las mantiene en el almacén online y en el offline. No hay dos caminos de código que puedan divergir ni sincronización que mantener.
- (3) → El **almacén offline** solo agrega registros, nunca los sobrescribe. Guarda la marca de tiempo del evento en Parquet dentro de S3 y se consulta con Athena, lo que permite reconstruir el valor de cada característica en una fecha pasada (consulta _point-in-time_).
- (5) → Los **grupos de características** se pueden buscar por nombre, descripción y etiquetas, y compartir con otros usuarios autorizados.
- Además es _fully managed_ y se distribuye en varias zonas de disponibilidad, así que el equipo no opera servidores.

**Por qué no las alternativas.**

- **Amazon S3**, incluso su clase de baja latencia S3 Express One Zone, guarda archivos completos. Mantener el último valor de millones de tarjetas actualizado en cada pago y, a la vez, un historial consistente sería justamente la construcción propia que prohíbe la restricción 4.
- **Amazon DynamoDB** es una base de datos NoSQL de clave-valor, totalmente administrada, con lecturas de milisegundos. Resolvería la restricción 1, pero no guarda el historial ni alimenta un almacén de entrenamiento: habría que construir y mantener el flujo hacia S3, y eso viola la restricción 4.
- **Amazon ElastiCache** es una caché en memoria (Redis, Valkey o Memcached) con latencias por debajo del milisegundo. Tiene el mismo problema que DynamoDB: sirve el presente, pero no conserva el pasado ni resuelve la consistencia con el entrenamiento.
- **Amazon Athena** y **Amazon Redshift** (el _data warehouse_ de AWS) sirven para consultas analíticas que tardan segundos, no milisegundos por registro. Quedan descartados para el camino online.
- **SageMaker Data Wrangler** y **AWS Glue DataBrew** **calculan** características, pero no las **almacenan ni sirven**. Pueden alimentar a Feature Store, pero no reemplazarlo.

### Escenario 2: red hospitalaria de investigación con datos clínicos → AWS Lake Formation

**Contexto.** Un consorcio de cinco hospitales reúne historias clínicas seudonimizadas y resultados de laboratorio en un data lake en S3, en formato Parquet y catalogado con AWS Glue. Unos 40 investigadores, cada uno con cuentas de AWS de su propia institución, entrenan modelos de riesgo de reingreso a 30 días desde notebooks de SageMaker, consultando los datos con Athena.

**Restricciones clave.**

1. **Filas**: cada investigador solo puede ver a los pacientes de su hospital, salvo en proyectos multicéntricos aprobados por el comité de ética.
2. **Columnas**: nadie fuera del equipo de gobierno de datos puede ver identificadores indirectos, como el código postal completo o la fecha de nacimiento exacta.
3. **Una sola copia**: el comité prohíbe crear copias filtradas por hospital o por proyecto. Cada copia es otra superficie de riesgo y acaba desincronizándose de la original.
4. Los permisos deben estar declarados en un solo lugar, en un formato que el comité pueda revisar, y cada acceso debe quedar registrado para auditoría.
5. Los investigadores pertenecen a cuentas de AWS distintas.

**Por qué este servicio.**

- (1) y (2) → Lake Formation permite conceder permisos a nivel de **columna, fila y celda**, esto último mediante filtros de datos, sobre las tablas del catálogo de Glue.
- (3) → Los permisos se aplican **sobre la misma tabla**. Cada investigador ve un subconjunto distinto de la única copia de los datos.
- (4) → Los permisos se conceden y revocan de forma centralizada, en un estilo parecido al `GRANT` de SQL. Los accesos quedan registrados en **AWS CloudTrail**, el servicio que guarda el historial de las llamadas a la API de AWS en la cuenta.
- (5) → Lake Formation permite compartir tablas, o partes de ellas, con otras cuentas de AWS.
- Los investigadores no reciben permisos directos sobre el bucket. Acceden a través de motores integrados como Athena, a los que Lake Formation entrega credenciales temporales válidas solo para los datos autorizados. Consulta en la documentación la lista vigente de servicios integrados.

**Por qué no las alternativas.**

- **Políticas de IAM y de bucket de S3** controlan el acceso por bucket, prefijo u objeto. Un archivo Parquet contiene todas las columnas y filas, así que no se puede ocultar una columna dentro de él sin crear archivos separados, lo que viola la restricción 3.
- **S3 Access Points** y **S3 Access Grants** facilitan dar accesos por aplicación o por prefijo, pero tienen la misma granularidad de objeto: no filtran columnas ni filas.
- **AWS Glue DataBrew** o un **Glue job** que genere versiones enmascaradas por hospital crean exactamente las copias que prohíbe la restricción 3.
- **Amazon Macie** descubre y clasifica datos sensibles en S3, pero no controla quién puede leerlos.
- **Amazon Redshift** tiene seguridad a nivel de fila y de columna, pero exige cargar los datos en el _warehouse_, lo que crea otra copia. Además, los accesos directos al lago en S3 quedarían fuera de su control.

### Escenario 3: agricultura de precisión con imágenes de drones → Amazon S3 como base del data lake

**Contexto.** Una empresa de agricultura de precisión opera drones que, en temporada, generan alrededor de 2 TB diarios de imágenes multiespectrales (GeoTIFF). A eso se suman lecturas de sensores de suelo en JSON cada cinco minutos y datos climáticos en CSV descargados de servicios externos. Con esos datos entrena modelos de detección de plagas (CNN) en SageMaker, cataloga todo con crawlers de Glue y consulta metadatos con Athena. El volumen va a pasar de decenas a cientos de terabytes en pocos años. Por contrato con las aseguradoras agrícolas, las imágenes de cada temporada deben conservarse siete años, aunque después de la primera temporada casi no se consultan.

**Restricciones clave.**

1. Formatos heterogéneos (imágenes, JSON, CSV, Parquet) que deben guardarse **sin definir un esquema antes**.
2. Crecimiento sin planificar capacidad: nadie en la empresa sabe administrar ni ampliar discos.
3. Los mismos datos deben ser legibles por Athena, los crawlers de Glue, el entrenamiento de SageMaker y el almacén offline de Feature Store, **sin copiarlos** entre sistemas.
4. Costo bajo para datos fríos, idealmente con transición automática.
5. Durabilidad máxima: una temporada de imágenes perdida es irrecuperable, porque no se puede volver a volar sobre el campo del año pasado. Aquí la opción «recolectar de nuevo» de la sección de valores faltantes simplemente no existe.

**Por qué este servicio.**

- (1) → S3 es **almacenamiento de objetos**: guarda cualquier archivo tal como llega, que es el _schema-on-read_ del data lake.
- (2) → La capacidad es, a efectos prácticos, ilimitada y no se aprovisiona: se paga por lo que se guarda.
- (3) → Athena consulta archivos en S3, los crawlers de Glue catalogan S3, el entrenamiento de SageMaker lee sus datos de S3 y el almacén offline de Feature Store vive en S3. Es la **fuente única de verdad** del capítulo.
- (4) → Las clases de almacenamiento, Intelligent-Tiering o reglas de ciclo de vida que mueven objetos a Glacier por antigüedad, permiten conservar siete años de imágenes a un costo por GB mucho menor.
- (5) → Su objetivo de diseño de durabilidad es de once nueves, con réplicas en varias zonas de disponibilidad en las clases estándar.

**Por qué no las alternativas.**

- **Amazon EBS** es almacenamiento de bloques: un disco virtual unido a una instancia en una sola zona de disponibilidad, con un tamaño que se fija por adelantado (del orden de decenas de TiB como máximo por volumen, según el tipo; verifica los límites vigentes). Athena y Glue no pueden leerlo, y viola las restricciones 2 y 3.
- **Amazon EFS** es un sistema de archivos compartido por red que usa **NFS**, el protocolo estándar de Linux para montar una carpeta remota como si fuera local. Crece solo, pero en su clase estándar cuesta mucho más por GB que S3 Standard (del orden de diez veces en us-east-1; verifica los precios vigentes), y Athena y los crawlers de Glue no trabajan sobre él. Está pensado para aplicaciones que necesitan una carpeta compartida, no para un data lake.
- **Amazon FSx** es una familia de sistemas de archivos administrados. **FSx for Lustre** es un sistema de archivos paralelo de alto rendimiento, usado en supercómputo, que se usa en ML como caché rápida **delante** de un bucket de S3 para acelerar el entrenamiento: complementa al lago, no lo reemplaza. **FSx for Windows File Server** ofrece carpetas compartidas por **SMB**, el protocolo de archivos compartidos de Windows, para aplicaciones de Windows. **FSx for NetApp ONTAP** y **FSx for OpenZFS** replican las capacidades de sistemas de almacenamiento empresarial (NetApp y ZFS) para migrar aplicaciones que dependen de ellos. Ninguno resuelve las restricciones 3 y 4.
- **Amazon Redshift, Amazon RDS** (bases de datos relacionales administradas) o **DynamoDB** exigen esquema al escribir y no están hechas para guardar terabytes de imágenes binarias. Violan la restricción 1.

## Resumen (_Summary_)

En este capítulo aprendiste lo que necesitas hacer para preparar tu dataset crudo como un conjunto significativo de características, que almacena el contenido relevante de tus datos en un formato adecuado para alimentar a un algoritmo de ML, que en última instancia lo entienda y produzca predicciones precisas y confiables.

Se presentaron los distintos tipos de datos (categóricos, numéricos, textuales, de imagen y de series de tiempo) para agrupar lógicamente las distintas técnicas de transformación de datos.

Al hacer ingeniería de tus datos con selección y extracción de características, aprendiste a seguir un enfoque minimalista (reducción de dimensionalidad) para elegir selectivamente las características relevantes que importan a tu modelo de ML. Se cubrieron todos los aspectos de la limpieza de datos, con énfasis en la detección y eliminación de valores atípicos, así como en los métodos de imputación de datos faltantes. Se presentaron técnicas de ingeniería de características para datos numéricos, categóricos, de series de tiempo y no estructurados, con ejemplos en el lenguaje de programación Python.

Aprendiste que Amazon SageMaker Data Wrangler y Amazon SageMaker Feature Store son los más adecuados, respectivamente, para realizar la ingeniería de características y para almacenar, compartir y gestionar las características resultantes.

Por último, aprendiste a desarrollar una estrategia de etiquetado eficaz con Amazon SageMaker Ground Truth, a mitigar el desbalance de clases con Amazon SageMaker Clarify y a dividir tu dataset procesado en datasets de entrenamiento, validación y prueba usando Amazon SageMaker Data Wrangler.

En los próximos capítulos aprenderás a elegir un enfoque de modelado, a entrenar y refinar tu modelo, y a evaluar su rendimiento.

## Puntos esenciales para el examen (_Exam Essentials_)

**Conoce las técnicas de ingeniería de características que manejan valores atípicos.** Para manejar valores atípicos en la ingeniería de características, es esencial aplicar técnicas que mitiguen su impacto en tu modelo de ML. Entre estos métodos están eliminar los valores atípicos por completo (siempre que sea posible) o transformarlos con enfoques como la transformación logarítmica, que puede reducir el sesgo de tus datos. Además, los métodos de imputación, como reemplazar los atípicos por la mediana o la media, pueden ser eficaces para mantener la integridad del dataset y reducir su impacto en el modelo de ML resultante.

**Conoce las técnicas de ingeniería de características que manejan el sesgo.** Para manejar el sesgo (la asimetría de la distribución), técnicas como la transformación logarítmica, la de raíz cuadrada y las transformaciones de Box-Cox o Yeo-Johnson son excelentes opciones. Estas técnicas pueden ayudar a normalizar las distribuciones y hacerlas más adecuadas para el análisis estadístico, sobre todo con datos sesgados, donde la mayoría de los valores se agrupan hacia un lado de la distribución y hay una cola larga del otro.

**Conoce las técnicas de ingeniería de características que no manejan el sesgo.** La estandarización por puntuación Z no corrige directamente el sesgo. Escala los datos para que tengan media 0 y desviación estándar 1, pero si los datos originales están sesgados, siguen sesgados después de la transformación.

El escalado MinMax tampoco corrige directamente el sesgo. Reajusta los datos para que quepan en un rango específico, normalmente de 0 a 1, pero no transforma la distribución de los datos. Si tus datos están sesgados, el sesgo persiste incluso después del escalado. Lo mismo vale para el escalado robusto y MaxAbs: todos son transformaciones lineales y ninguna cambia la forma de la distribución.

**Conoce cuándo hacer ingeniería de características con normalización frente a estandarización.** La normalización es útil cuando importa la escala, mientras que la estandarización se prefiere cuando la clave es la distribución.

**Conoce las técnicas de ingeniería de características que se usan para la estandarización.** Para la estandarización, cuyo propósito es asegurar que cada característica de tu dataset contribuya por igual al rendimiento del modelo de ML, considera usar la puntuación Z con datos de distribución normal.

**Conoce las técnicas de ingeniería de características que se usan para la normalización.** Para la normalización, cuyo propósito es escalar cada característica de tu dataset al mismo rango (normalmente de 0 a 1), considera usar la técnica de escalado MinMax.

**Conoce las técnicas de ingeniería de características que se usan para datos categóricos.** Usa label encoding para datos categóricos ordinales y modelos de ML basados en árboles. Usa one-hot encoding para conjuntos pequeños de datos categóricos nominales y modelos de ML no basados en árboles. Usa binary encoding para reducir la dimensionalidad que causan las técnicas one-hot. Usa feature hashing para datos categóricos de alta cardinalidad, sobre todo en casos donde hay que optimizar los recursos de cómputo y la solución debe ser económica.

> [!warning] Nota de precisión
> Para datos ordinales, recuerda fijar el orden explícitamente (`OrdinalEncoder` con `categories=`), porque `LabelEncoder` ordena alfabéticamente. Ver la nota en «Codificación por etiquetas».

**Conoce las técnicas de ingeniería de características que se usan para datos de imagen.** Para datos de imagen, puedes extraer características de tu dataset crudo con Amazon SageMaker JumpStart. Amazon Rekognition también es una opción válida para el análisis básico de imágenes y la extracción de características, según tu caso de uso. Después de extraer las características relevantes del dataset, puedes aprovechar Amazon SageMaker Data Wrangler para transformar tu dataset con normalización y reducción de dimensionalidad. Por último, puedes almacenar tus características en Amazon SageMaker Feature Store.

**Conoce las técnicas de ingeniería de características que se usan para datos textuales.** Para datos textuales, puedes extraer características de tu dataset crudo con Amazon Comprehend o Amazon Textract. Para una extracción más personalizada, Amazon SageMaker te permite usar tokenización, stemming y lematización para hacer una selección básica de palabras o reducirlas a su forma raíz para el análisis semántico posterior.

**Conoce los servicios de AWS que se usan para el etiquetado de datos.** Para el etiquetado de datos, cuyo propósito es enriquecer tu dataset con las predicciones reales o con metadatos significativos que ayuden a tu modelo a aprender más rápido, considera usar Amazon SageMaker Ground Truth.

> [!warning] Estado del servicio
> Ground Truth ya no admite clientes nuevos y Mechanical Turk cierra el 30 de septiembre de 2026 (ver la sección de Ground Truth).

**Conoce cómo gestionar el desbalance de clases.** Para mitigar los riesgos que introduce el desbalance de clases, selecciona una característica sensible al sesgo (faceta) y considera calcular para ella las métricas de sesgo previas al entrenamiento (que proporciona Amazon SageMaker Clarify), como el desbalance de clases (CI) o la diferencia en proporciones de etiquetas (DPL). Determina la estrategia adecuada en consecuencia: sobremuestreo, submuestreo o ponderación de clases.

> [!warning] Estado del servicio
> Clarify ya no admite clientes nuevos. Las métricas CI y DPL se calculan igual con pandas a partir de sus fórmulas (ver la sección de Clarify).

**Conoce las diferencias entre los datasets de entrenamiento, validación y prueba.** El dataset de entrenamiento es donde el modelo aprende patrones, relaciones y características de los datos. En esencia, es donde el modelo recibe su «educación». Una vez entrenado el modelo, el dataset de validación ayuda a ajustar los hiperparámetros y a afinar el modelo para mejorar su rendimiento y su generalización a datos no vistos. Es como un examen de práctica antes del examen final. El dataset de prueba se usa para evaluar el rendimiento general del modelo. Proporciona una valoración imparcial de qué tan bien generaliza el modelo a datos nuevos y no vistos. Piensa en él como el examen final que determina la calificación del modelo.

## Preguntas de repaso (_Review Questions_)

1. Estás preparando un dataset que contiene características numéricas, categóricas y ordinales. Para entrenar un modelo predictivo y aumentar su precisión, necesitas transformar las características categóricas en valores numéricos. ¿Qué solución de ingeniería de características es la más adecuada para este caso de uso?
   - A. One-hot encoding
   - B. Escalado de características (_feature scaling_)
   - C. Extracción de características (_feature extraction_)
   - D. Formateo de fechas (_date formatting_)

2. Quieres convertir una columna de tus datos de entrenamiento en valores binarios. ¿Qué técnica es la más adecuada para esta transformación?
   - A. One-hot encoding
   - B. Tokenización
   - C. Label encoding
   - D. Feature hashing

3. Durante la preparación de datos, descubres valores faltantes en algunas columnas de un dataset que contiene características categóricas. Necesitas asegurarte de que esto no distorsione los datos ni reduzca la confiabilidad del modelo. ¿Qué solución es la más adecuada para este caso de uso?
   - A. Amazon SageMaker Clarify
   - B. Imputación múltiple (_multiple imputations_)
   - C. Eliminar la característica
   - D. Volver a recolectar los datos

4. Estás entrenando un modelo y, durante el análisis de datos, observas distintas variables de entrada que varían significativamente. Quieres asegurarte de que tu dataset no tenga características con valores más grandes que influyan mucho en la capacidad predictiva del modelo. ¿Qué transformación es la más adecuada en este escenario?
   - A. Normalización
   - B. Estandarización
   - C. Binning
   - D. One-hot encoding

5. Estás preparando un dataset que tiene varias características categóricas, cada una con alta cardinalidad. Quieres hacer la ingeniería de características del dataset de forma eficiente y económica. ¿Qué solución se ajusta mejor a este caso de uso?
   - A. Label encoding
   - B. Lag features
   - C. Binary encoding
   - D. Feature hashing

6. Un modelo entrenado para reconocer autos no está rindiendo bien. Necesitas rediseñar las características del dataset de imágenes para asegurar un mejor rendimiento. ¿Qué solución se ajusta mejor a este escenario?
   - A. Tokenización con Amazon SageMaker Data Wrangler
   - B. Lag feature con Amazon SageMaker Data Wrangler
   - C. Binning con Amazon SageMaker Data Wrangler
   - D. Amazon SageMaker JumpStart

7. ¿Qué técnica avanzada se usa comúnmente en la ingeniería de características para datos textuales con el fin de convertir palabras en vectores numéricos que capturan su significado semántico?
   - A. One-hot encoding
   - B. Tokenización
   - C. Word embeddings
   - D. Normalización

8. ¿Cuál es la técnica más común para manejar el desbalance de clases en datasets de machine learning?
   - A. Codificación de datos (_data encoding_)
   - B. Aumento de datos (_data augmentation_)
   - C. Escalado de características (_feature scaling_)
   - D. División de datos (_data splitting_)

9. ¿Qué servicios de AWS pueden usarse para el etiquetado de datos en AWS?
   - A. Amazon Comprehend
   - B. Amazon SageMaker Ground Truth
   - C. Amazon Rekognition
   - D. Amazon SageMaker Clarify

10. ¿Cuál es el propósito de dividir un dataset en datasets de entrenamiento, validación y prueba en machine learning?
    - A. Mejorar la precisión del modelo
    - B. Asegurar que el modelo tenga datos diversos
    - C. Prevenir el sobreajuste y evaluar el rendimiento del modelo
    - D. Simplificar el procesamiento de datos

## Glosario

- **Active learning (aprendizaje activo).** Método en que un modelo decide qué ejemplos, los de menor confianza, deben etiquetar las personas; es la base del etiquetado automatizado de Ground Truth.
- **Almacén offline (Feature Store).** Historial completo de valores de características, solo de escritura acumulativa, en Parquet dentro de S3; sirve para entrenar e inferir por lotes.
- **Almacén online (Feature Store).** Último valor de cada característica por identificador, con lecturas de pocos milisegundos; sirve para la inferencia en tiempo real.
- **Almacenamiento de archivos.** Carpeta compartida por red que varias máquinas montan a la vez (en AWS: EFS, FSx).
- **Almacenamiento de bloques.** Disco virtual que se conecta a una máquina y el sistema operativo usa como disco propio (en AWS: EBS).
- **Almacenamiento de objetos.** Archivos guardados enteros, con metadatos y una clave, dentro de buckets, y accesibles por API web (en AWS: S3).
- **Alta cardinalidad.** Propiedad de una variable categórica con muchísimos valores distintos, como códigos postales o identificadores de producto.
- **Athena (Amazon Athena).** Motor SQL serverless que consulta archivos directamente en S3 y cobra por datos escaneados.
- **AWS Marketplace.** Tienda de software y servicios de terceros integrada en AWS; ahí se contratan los proveedores de etiquetado.
- **Base de datos operacional.** Base que usa una aplicación en su día a día para registrar transacciones, a diferencia de las bases analíticas.
- **Batch (lote).** Bloque grande de datos que se carga o se procesa de una vez.
- **Bedrock (Amazon Bedrock).** Servicio fully managed para usar modelos fundacionales de varios proveedores mediante API.
- **BiasConfig.** Clase del SDK de SageMaker para Python que declara la etiqueta positiva, la faceta y el grupo de un análisis de sesgo de Clarify.
- **Bounding box (cuadro delimitador).** Rectángulo, dado por coordenadas, que encierra un objeto detectado en una imagen.
- **Bucket.** Contenedor de objetos de S3, con nombre único, dentro del cual cada objeto se identifica por su clave.
- **Canvas (Amazon SageMaker Canvas).** Entorno visual _low-code_ de SageMaker AI; hoy aloja Data Wrangler.
- **Catálogo de datos (AWS Glue Data Catalog).** Registro central de metadatos (esquemas y ubicaciones en S3) que convierte archivos en tablas consultables.
- **Cifrado en reposo.** Protección de los datos mientras están guardados en disco.
- **Cifrado en tránsito.** Protección de los datos mientras viajan por la red, normalmente con TLS.
- **Cifrado en uso.** Protección de los datos mientras se procesan en memoria, por ejemplo con enclaves aislados.
- **Clarify (Amazon SageMaker Clarify).** Herramienta de SageMaker para medir sesgo y explicar predicciones; ya no admite clientes nuevos.
- **Clase de almacenamiento (S3).** Nivel de precio y acceso de S3 (Standard, Standard-IA, Glacier, etc.) según la frecuencia de lectura.
- **CloudTrail (AWS CloudTrail).** Servicio que registra el historial de llamadas a la API de AWS en una cuenta, útil para auditoría.
- **CNN (red neuronal convolucional).** Red neuronal para imágenes que aprende filtros de bordes, texturas y formas.
- **Colisión (hash).** Caso en que dos entradas distintas producen el mismo valor hash.
- **Compliance (cumplimiento normativo).** Capacidad de demostrar ante auditores o reguladores que se cumplen normas como RGPD o HIPAA.
- **Comprehend (Amazon Comprehend).** Servicio de NLP ya entrenado que extrae entidades, frases clave, sentimiento e idioma de un texto.
- **Crawler (AWS Glue).** Proceso que recorre archivos en S3, infiere su esquema y lo registra en el catálogo.
- **Crowdsourcing.** Reparto de una tarea entre una multitud de personas externas que cobran por tarea.
- **Data lake (lago de datos).** Repositorio central que guarda datos crudos en su formato original y aplica el esquema al leer.
- **Data warehouse (almacén de datos).** Base analítica que exige un esquema definido antes de cargar los datos.
- **Data Wrangler (Amazon SageMaker Data Wrangler).** Herramienta visual _low-code_ para importar, transformar, analizar y dividir datos mediante flujos; hoy está dentro de Canvas.
- **DataBrew (AWS Glue DataBrew).** Herramienta visual sin código para limpiar y normalizar datos mediante recetas reutilizables que se ejecutan como trabajos serverless.
- **Datos dispersos (sparse).** Datos en que la gran mayoría de los valores son cero y se almacenan guardando solo los distintos de cero.
- **Disponibilidad.** Probabilidad de poder acceder a un dato o servicio en un momento dado.
- **Durabilidad.** Probabilidad de no perder un dato guardado; para S3 el objetivo de diseño es 99.999999999 %.
- **DynamoDB (Amazon DynamoDB).** Base de datos NoSQL de clave-valor, totalmente administrada, con lecturas de milisegundos.
- **EBS (Amazon Elastic Block Store).** Almacenamiento de bloques: discos virtuales para instancias EC2 dentro de una zona de disponibilidad.
- **EFS (Amazon Elastic File System).** Sistema de archivos compartido por NFS que crece automáticamente.
- **ElastiCache (Amazon ElastiCache).** Caché en memoria administrada (Redis, Valkey o Memcached) con latencias por debajo del milisegundo.
- **Embedding / vector de características.** Vector numérico denso que resume una imagen, una palabra o un texto, de modo que la cercanía entre vectores refleja similitud.
- **Entidad (NLP).** Mención de algo con nombre o tipo reconocible (persona, lugar, organización, fecha) dentro de un texto.
- **Faceta (Clarify).** Característica, normalmente un atributo sensible, cuyos valores definen los grupos que se comparan en un análisis de sesgo.
- **Feature group (grupo de características).** Tabla de Feature Store: columnas de características y filas con identificador y marca de tiempo.
- **Feature Store (Amazon SageMaker Feature Store).** Repositorio fully managed para almacenar, compartir y servir características, con almacenes online y offline.
- **FSx (Amazon FSx).** Familia de sistemas de archivos administrados: Lustre, Windows File Server, NetApp ONTAP y OpenZFS.
- **Fuente única de verdad (single source of truth).** Una sola copia autorizada de los datos a la que apuntan todos los sistemas.
- **Fuerza de trabajo privada.** Empleados o contratistas propios que etiquetan datos en un portal privado de Ground Truth.
- **Fully managed (totalmente administrado).** Servicio cuya infraestructura opera AWS; el usuario solo lo consume y paga por uso.
- **Función hash.** Función determinista que convierte cualquier entrada en un número entero; distintas entradas pueden coincidir.
- **Glue (AWS Glue).** Servicio serverless de integración de datos: catálogo, crawlers y trabajos ETL.
- **Glue job (trabajo de Glue).** Script, normalmente PySpark, que Glue ejecuta en infraestructura serverless cobrando por tiempo de cómputo.
- **Gobernanza de datos.** Reglas y procesos que determinan quién puede usar qué datos, para qué y cómo se audita.
- **Ground Truth (Amazon SageMaker Ground Truth).** Servicio de etiquetado de datos con personas y automatización; ya no admite clientes nuevos.
- **Ground truth (verdad de referencia).** Valor correcto y verificado de la variable objetivo con el que se entrena y evalúa un modelo.
- **Guardrails (barreras de seguridad).** Reglas preventivas o de detección que acotan lo que cualquiera puede hacer en una cuenta.
- **Hiperparámetro.** Ajuste que se fija antes de entrenar (profundidad de un árbol, tasa de aprendizaje), a diferencia de los parámetros que se aprenden de los datos.
- **Human-in-the-loop.** Diseño en el que las personas revisan o deciden en ciertos puntos de un proceso automático.
- **IAM (AWS Identity and Access Management).** Servicio donde se define, mediante políticas, qué identidad puede hacer qué acción sobre qué recurso.
- **Inferencia (en ML).** Uso de un modelo entrenado para producir predicciones sobre datos nuevos; no es la inferencia estadística.
- **Inferencia en tiempo real / por lotes.** Responder una petición individual en milisegundos, frente a puntuar muchos registros de una vez.
- **Intelligent-Tiering (S3).** Clase de S3 que mueve cada objeto entre niveles de precio según su patrón de acceso.
- **JumpStart (Amazon SageMaker JumpStart).** Catálogo de modelos preentrenados que se despliegan o ajustan con pocos pasos.
- **Lake Formation (AWS Lake Formation).** Capa de gobernanza de un data lake con permisos centralizados a nivel de tabla, columna, fila y celda.
- **Línea de negocio (LOB).** Aplicaciones que sostienen los procesos centrales de una empresa, como el ERP o el CRM.
- **Low-code / no-code.** Herramientas en que se trabaja sobre todo configurando pasos visuales, con poco código o ninguno.
- **Lustre.** Sistema de archivos paralelo de alto rendimiento usado en supercómputo; en AWS se ofrece como FSx for Lustre.
- **Macie (Amazon Macie).** Servicio que descubre y clasifica datos sensibles en S3, sin controlar el acceso.
- **Manifiesto de salida (Ground Truth).** Archivo en S3 con una línea JSON por objeto, que contiene la ubicación del dato y su etiqueta.
- **Mechanical Turk (Amazon MTurk).** Mercado de crowdsourcing para tareas humanas; cierra el 30 de septiembre de 2026.
- **Modelo fundacional (FM).** Modelo muy grande, preentrenado con datos generales, que sirve de base para muchas tareas.
- **NFS (Network File System).** Protocolo estándar, sobre todo en Linux, para montar una carpeta remota como si fuera local.
- **Notebook de SageMaker.** Entorno de Jupyter que corre en una instancia administrada por SageMaker.
- **OCR (reconocimiento óptico de caracteres).** Tecnología que convierte la imagen de un texto en texto de computadora.
- **Payload.** Contenido útil de un envío o mensaje individual.
- **Pipeline.** Cadena automatizada de pasos de datos y ML que se ejecuta sin intervención manual.
- **Point-in-time (consulta).** Reconstrucción del valor que tenía una característica en un instante pasado, para entrenar sin fuga de información.
- **Producción.** Entorno donde un modelo sirve predicciones reales a usuarios o sistemas.
- **Prompt.** Instrucción o texto de entrada que se le da a un modelo fundacional.
- **Receta (DataBrew).** Secuencia guardada de transformaciones que se puede reaplicar a otros datasets o programar.
- **Redshift (Amazon Redshift).** Data warehouse administrado de AWS para consultas analíticas SQL.
- **Rekognition (Amazon Rekognition).** Servicio de visión por computadora ya entrenado que detecta objetos, escenas, texto y rostros por API.
- **Rol de IAM.** Identidad con permisos que asumen temporalmente personas o servicios, como un notebook que lee S3.
- **S3 (Amazon Simple Storage Service).** Almacenamiento de objetos de AWS, base habitual de un data lake.
- **S3 Access Points / Access Grants.** Mecanismos para dar acceso a S3 por aplicación o por prefijo, con granularidad de objeto.
- **S3 Express One Zone.** Clase de S3 de muy baja latencia que guarda los datos en una sola zona de disponibilidad.
- **SageMaker (Amazon SageMaker AI).** Plataforma de ML de AWS: notebooks, preparación de datos, entrenamiento, despliegue y monitoreo.
- **Schema-on-read.** Aplicar el esquema al leer los datos, no al guardarlos.
- **SDK (software development kit).** Biblioteca para manejar un servicio desde código, como `boto3` o el SDK de SageMaker para Python.
- **Serverless.** Modelo en que no se aprovisionan servidores: el servicio asigna cómputo al vuelo y cobra por uso.
- **SMB (Server Message Block).** Protocolo de carpetas compartidas de Windows.
- **Tensor.** Arreglo multidimensional; generaliza vectores y matrices a más ejes.
- **Textract (Amazon Textract).** Servicio de OCR que además extrae la estructura de los documentos: formularios (clave-valor) y tablas.
- **Token.** Unidad en que se divide un texto; en los LLM suele ser una subpalabra, y es la unidad de cobro de los modelos de texto en Bedrock.
- **Training-serving skew.** Diferencia entre cómo se calculan las características al entrenar y al servir, que degrada el modelo en producción sin dar error.
- **Versionado de S3 (S3 Versioning).** Opción de un bucket que conserva las versiones anteriores de cada objeto al sobrescribirlo o borrarlo.
- **Zona de disponibilidad.** Uno o más centros de datos físicamente separados dentro de una región de AWS.
