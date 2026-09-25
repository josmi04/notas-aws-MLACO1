---
tema: "Capítulo 3 — Transformación de datos e ingeniería de características (versión explicada: AWS e infraestructura)"
fuente: Guia Oficial/03_data_transformation_and_feature_engineering.txt
guia-mla-c01:
  [
    Dominio 1,
    1.2 Transform data and perform feature engineering,
    1.3 Ensure data integrity and prepare data for modeling,
  ]
perfil-lector: profesional de datos/estadística sin experiencia administrando infraestructura
alcance-explicaciones: solo AWS e infraestructura; los conceptos de ML y estadística se conservan sin explicación añadida
versiones-codigo:
  python: "3.12.6"
  scikit-learn: "1.5.2"
  pandas: "2.2.2"
  numpy: "2.0.2"
  scipy: "1.14.1"
  category_encoders: "2.6.4"
  sagemaker-sdk-del-libro: "2.x"
  sagemaker-sdk-instalado-localmente: "3.22.1"
verificado: 2026-09-25
tags:
  [
    aws,
    mla-c01,
    mla-c02,
    feature-engineering,
    data-lake,
    s3,
    lake-formation,
    glue,
    glue-databrew,
    athena,
    sagemaker,
    feature-store,
    data-wrangler,
    canvas,
    jumpstart,
    rekognition,
    comprehend,
    textract,
    bedrock,
    ground-truth,
    clarify,
  ]
---

> [!info] Cómo leer esta versión
>
> - **Qué es.** Traducción íntegra al español del capítulo 3 de la guía oficial de estudio, con los mismos encabezados, ejemplos, fragmentos de código y afirmaciones. No se eliminó nada. Los encabezados conservan entre paréntesis el título original en inglés, porque el examen se presenta en inglés.
> - **Qué se añadió.** Explicaciones de los términos de AWS y de infraestructura integradas en el texto, aclaraciones de razonamientos que el libro da por sentados, **notas de precisión** (recuadros amarillos) cuando el original es impreciso o está desactualizado, la sección «Escenarios donde este servicio es la opción obligada» y un glosario al final.
> - **Qué no se añadió.** Las explicaciones nuevas se limitan a AWS e infraestructura: qué hace cada servicio, dónde guarda los datos, qué recursos usa, qué permisos necesita, cuánto y cómo cobra, cómo escala y qué administra AWS frente a lo que administras tú. Los conceptos de ML y estadística (tipos de datos, valores atípicos, escalado, codificaciones, desbalance de clases, división de datos, etc.) aparecen tal como los presenta el libro, sin desarrollo adicional. Cuando el libro contiene un error de hecho en esos temas, solo se señala en una nota breve.
> - **Figuras y fórmulas.** El `.txt` no incluye las imágenes, así que solo quedan los pies de figura. Cuando la figura mostraba la salida de texto de un programa que se puede ejecutar sin AWS, aquí va la salida real obtenida con las versiones del encabezado. Varias fórmulas se perdieron en la extracción del PDF y aparecen como huecos; se reconstruyeron en su forma estándar y se marcan como reconstruidas.
> - **Código.** Se corrigieron defectos de la extracción del PDF (comillas tipográficas `‘ ’ “ ”` en lugar de comillas rectas, el signo menos tipográfico `−` y líneas partidas a la mitad) para que el código se pueda ejecutar. La lógica no se modificó.
> - **Cifras y estado de los servicios.** Se verificaron en la documentación de AWS el 25 de septiembre de 2026. Cambian con frecuencia: confírmalos en la documentación oficial vigente antes de usarlos en una decisión real.

> [!warning] Cambios en AWS posteriores a la edición del libro (verificado el 25-09-2026)
>
> - **Amazon SageMaker** pasó a llamarse **Amazon SageMaker AI**. El nombre «Amazon SageMaker» designa ahora una plataforma más amplia que reúne datos, analítica e IA. En este texto se usa el nombre del libro.
> - **SageMaker Data Wrangler** ya no es una aplicación independiente dentro de Studio: en la versión actual de SageMaker Studio se usa desde **Amazon SageMaker Canvas**, el entorno visual de SageMaker AI. La versión de _Studio Classic_ sigue documentada para quien no haya migrado.
> - **Amazon SageMaker Ground Truth** y **Amazon SageMaker Clarify** ya no admiten clientes nuevos. Los clientes existentes pueden seguir usándolos con normalidad; AWS sigue invirtiendo en su seguridad y disponibilidad, pero no planea añadirles funciones. Las preguntas de examen basadas en el libro pueden seguir mencionándolos.
> - **Amazon Mechanical Turk**, la fuerza de trabajo pública que el libro menciona para el etiquetado, cierra definitivamente el **30 de septiembre de 2026**. Ese mismo día desaparece como opción de fuerza de trabajo en Ground Truth.
> - **SDK de SageMaker para Python v3.** El libro usa la versión 2 del SDK (por ejemplo, `sagemaker.clarify.BiasConfig`). En la versión 3, que es la que instala hoy `pip install sagemaker` (en esta máquina, la 3.22.1), ese módulo ya no existe con ese nombre: la clase está en `sagemaker.core.clarify`, y el antiguo `sagemaker.feature_store` se reemplazó por el recurso `FeatureGroup` (en `sagemaker.mlops.feature_store`). Para ejecutar código del libro sin reescribirlo, crea un entorno virtual aparte con `pip install "sagemaker>=2,<3"`.

> [!tip] Relevancia para el examen (MLA-C01 frente a MLA-C02)
> El último día para presentar el MLA-C01 en inglés es el **28-09-2026**; el MLA-C02 entra en beta el 29-09-2026. La preparación de datos de este capítulo sigue siendo materia de ambos. Tres matices para el C02: Mechanical Turk sale de la lista de servicios del examen, las habilidades evaluadas ya no nombran a Ground Truth ni a Clarify (aunque los conceptos de etiquetado y de sesgo siguen dentro), y el examen exige más IA generativa de la que aparece aquí en la sección de tokenización con Bedrock.

**Capítulo 3**

# Transformación de datos e ingeniería de características (_Data Transformation and Feature Engineering_)

> LOS OBJETIVOS DEL EXAMEN AWS CERTIFIED MACHINE LEARNING (ML) ENGINEER – ASSOCIATE QUE CUBRE ESTE CAPÍTULO PUEDEN INCLUIR, ENTRE OTROS, LOS SIGUIENTES:
>
> ✔ Dominio 1: Preparación de datos para machine learning
>
> - 1.2 Transformar datos y realizar ingeniería de características
> - 1.3 Asegurar la integridad de los datos y prepararlos para el modelado

## Introducción (_Introduction_)

En el capítulo 2 aprendiste a ingerir datos desde distintas fuentes y a almacenarlos en AWS. Ahora que ya recolectaste los datos y los resguardaste en AWS, tu trabajo como ingeniero de machine learning (ML) es dejarlos listos para entrenar tu modelo de ML. Este paso corresponde a la fase _Process Data_ (procesar datos) del ciclo de vida de ML, como se muestra en la Figura 3.1.

_Figura 3.1 El ciclo de vida de ML._

Tus datos suelen estar almacenados en su estado crudo en un **_data lake_** (lago de datos). Un data lake es un repositorio central donde los archivos se guardan tal como llegan (CSV, JSON, Parquet, imágenes, registros de aplicaciones) sin exigirles un esquema en el momento de escribirlos; el esquema se aplica después, al leerlos, lo que se conoce como _schema-on-read_. Se contrapone al **_data warehouse_** (almacén de datos), una base de datos analítica que exige tablas con un esquema definido antes de cargar nada (_schema-on-write_). Una arquitectura de data lake puede proporcionar una base sólida sobre la cual construir una solución de ML, porque está diseñada para almacenar cantidades masivas de datos en un repositorio central, de modo que estén disponibles para que distintos grupos de tu organización los categoricen, procesen, enriquezcan y consuman. El razonamiento implícito es doble: un proyecto de ML suele necesitar datos que no son tablas (imágenes, texto), que un warehouse no guarda bien, y conservar el dato crudo permite rehacer la preparación desde cero cada vez que cambie el modelo, sin volver a pedir los datos a los sistemas de origen.

Las siguientes son las características clave de un data lake:

- **Almacenamiento a escala.** Un data lake debe poder acomodar datos que llegan en cualquier momento, ya sea a intervalos predefinidos o en tiempo real, en cargas pequeñas o en grandes lotes. En la jerga, cada envío pequeño es un **_payload_** (la carga útil de un mensaje, como el evento JSON de pocos kilobytes que manda un sensor), mientras que un lote puede ser el volcado nocturno de una base de datos de varios gigabytes. El almacenamiento tiene que absorber ambos patrones sin que haya que rediseñarlo ni ampliar discos a mano.
- **Movimiento de datos.** Igual que un lago real tiene un río de entrada que trae el agua y un río de salida que se la lleva, un data lake debe permitir que los datos entren y salgan de él. En AWS, los «ríos de entrada» son los servicios de ingesta del capítulo 2 (por ejemplo, Amazon Data Firehose para flujos continuos o AWS DataSync para copiar archivos desde un centro de datos), y los de salida son los servicios que leen del lago para procesar o entrenar.
- **Datos crudos.** Los datos que llegan al data lake pueden provenir de fuentes distintas y ser estructurados, semiestructurados o no estructurados. El formato de los datos es irrelevante durante las fases de ingesta y almacenamiento de la recolección de datos. Es irrelevante precisamente por el _schema-on-read_: la decisión sobre cómo interpretar cada archivo se pospone a la fase de procesamiento, que es el tema de este capítulo.
- **Seguridad.** Como al data lake pueden llegar datos sensibles, es crítico que ningún usuario no autorizado tenga acceso a ellos. Por eso el data lake debe protegerse con controles robustos de gestión de identidades y accesos (**IAM**, _identity and access management_). En AWS, IAM es el servicio donde se define qué identidad puede hacer qué acción sobre qué recurso. Las identidades son usuarios, grupos y **roles**; un rol es una identidad sin contraseña propia que asumen temporalmente personas o servicios (por ejemplo, un notebook de SageMaker asume un rol para poder leer un bucket). Los permisos se escriben en **políticas**, documentos JSON del tipo «permitir `s3:GetObject` sobre `arn:aws:s3:::mi-lago/ventas/*`». Se parece a los permisos de una base de datos (`GRANT SELECT ON tabla TO usuario`), pero se aplica a todos los servicios de la cuenta. El cifrado en uso y en tránsito no es menos importante que el cifrado en reposo:
  - El **cifrado en reposo** protege los datos mientras están guardados: quien obtenga el disco o una copia del archivo no puede leerlos sin la clave. En S3 se activa por bucket, con claves administradas por S3 o con claves de **AWS KMS** (_Key Management Service_), el servicio de AWS que crea y custodia claves de cifrado y registra cada uso que se hace de ellas.
  - El **cifrado en tránsito** protege los datos mientras viajan por la red, con **TLS**, el mismo protocolo que pone el candado de HTTPS en el navegador. Evita que alguien que intercepte el tráfico lo lea; en S3 se puede exigir con una política de bucket que rechace toda petición que no llegue por HTTPS.
  - El **cifrado en uso** protege los datos mientras se procesan en la memoria de una máquina. Es el más difícil de lograr y requiere hardware o entornos especiales, como los _enclaves_ (en AWS, AWS Nitro Enclaves: una porción aislada de una instancia EC2 a la que ni el administrador de la propia instancia puede entrar).

  Por lo tanto, deben establecerse y hacerse cumplir barreras de seguridad (**_guardrails_**) que protejan los datos mientras se producen y se consumen. Un guardrail es una regla que acota lo que cualquiera puede hacer en la cuenta, sin importar qué permisos tenga cada persona. En AWS se implementan, por ejemplo, con _S3 Block Public Access_ (que impide que un bucket se haga público aunque alguien lo intente), con políticas de control de servicios (**SCP**) de AWS Organizations (que prohíben acciones en todas las cuentas de una organización) o con reglas de AWS Config (que detectan y reportan recursos que incumplen una norma, como un bucket sin cifrado). Como las barandillas de una carretera, no dirigen el trabajo diario: impiden salirse del camino.
- **Catálogo.** Los data lakes deben permitirte almacenar datos relacionales (p. ej., bases de datos operacionales o datos de aplicaciones de línea de negocio) y no relacionales (p. ej., datos de aplicaciones móviles, dispositivos IoT y redes sociales). Una **base de datos operacional** es la que usa una aplicación en su día a día para registrar transacciones (pedidos, pagos, altas de clientes); las **aplicaciones de línea de negocio** (_line-of-business_, LOB) son las que sostienen los procesos centrales de la empresa, como el ERP, el CRM o la nómina. Los data lakes también te dan la capacidad de entender qué datos hay en el lago mediante el rastreo (_crawling_), la catalogación y la indexación de los datos. Sin un catálogo, un lago con millones de archivos se convierte en un «pantano de datos» donde nadie sabe qué hay ni dónde. Rastrear significa que un proceso automático (en AWS, un **_crawler_** de AWS Glue) recorre los archivos, infiere su esquema (columnas, tipos de dato y particiones, es decir, la organización de los archivos en carpetas como `anio=2025/mes=01/`) y lo registra como una tabla en un catálogo de metadatos. Indexar es organizar esos metadatos para que se puedan buscar.

Para crear un data lake en AWS, normalmente usarías una combinación de los siguientes servicios: Amazon S3 (para almacenamiento), AWS Glue (para catalogación), Amazon Athena (para consultas) y AWS Lake Formation (para la gestión central de IAM, la seguridad de los datos y la gobernanza).

Para entender por qué S3 ocupa el lugar del almacenamiento, conviene distinguir las tres familias de almacenamiento en la nube, porque el resto del libro las usa sin definirlas:

- **Almacenamiento de bloques.** Es un disco virtual que se conecta a una sola máquina, como el SSD de tu laptop. El sistema operativo lo formatea y lo usa como disco propio. En AWS es **Amazon EBS** (_Elastic Block Store_).
- **Almacenamiento de archivos.** Es una carpeta compartida por red que varias máquinas «montan» a la vez, como la unidad de red de una oficina. En AWS son **Amazon EFS** y la familia **Amazon FSx**.
- **Almacenamiento de objetos.** Cada archivo se guarda entero como un **objeto**, junto con sus metadatos, identificado por una **clave** (algo como `ventas/2025/enero.parquet`) dentro de un contenedor llamado **_bucket_**. No se monta como un disco: se lee y se escribe mediante una API web (peticiones HTTPS del tipo `GET` y `PUT`). Tampoco se modifica un fragmento de un objeto: se reemplaza el objeto completo. A cambio, escala prácticamente sin límite y es la opción más barata por gigabyte de las tres.

**Amazon S3** (_Simple Storage Service_) es el almacenamiento de objetos de AWS, y por eso es la base natural de un data lake. **AWS Glue** es un servicio de integración de datos; la pieza que importa aquí es su **Data Catalog**, donde se registran las definiciones de tabla: el esquema de los datos y la ubicación de los archivos en S3. **Amazon Athena** es un motor SQL que consulta directamente los archivos de S3 usando esas definiciones del catálogo. Athena es **_serverless_**: no hay servidor ni base de datos que encender, dimensionar o apagar; el servicio asigna cómputo al vuelo para cada consulta y cobra por la cantidad de datos que esa consulta escanea. Por eso guardar el lago en un formato columnar y comprimido como Parquet abarata directamente las consultas: Athena lee solo las columnas que la consulta pide. **AWS Lake Formation** es una capa de gobernanza sobre el catálogo de Glue. La **gobernanza de datos** es el conjunto de reglas y procesos que determinan quién puede usar qué datos, para qué y cómo se audita. Lake Formation permite conceder permisos a nivel de base de datos, tabla, columna, fila y celda desde un único lugar, en vez de combinar a mano políticas de IAM y políticas de bucket.

Con excepción de AWS Lake Formation, ya cubrimos estos servicios en el capítulo anterior. Para el examen, necesitas saber que Amazon S3 es la opción de almacenamiento preferida para un data lake en AWS, porque ofrece un almacenamiento altamente durable (99.999999999 %), altamente disponible y seguro, y una integración transparente con varios servicios de procesamiento de datos y plataformas de ML de AWS.

La **durabilidad** es la probabilidad de no perder un dato guardado; la **disponibilidad**, la de poder acceder a él en un momento dado. Son cosas distintas: un servicio puede estar momentáneamente inaccesible sin haber perdido nada. Los «once nueves» de durabilidad son difíciles de imaginar, y AWS los ilustra así: si guardas 10 millones de objetos en S3, en promedio esperarías perder uno cada 10 000 años. Como punto de comparación, las estadísticas públicas de grandes flotas de discos duros muestran tasas de falla anual del orden del 1 % por disco: guardar datos valiosos en un único disco de laptop o de servidor es apostar a que ese 1 % no te toque. S3 alcanza su cifra replicando cada objeto en varios dispositivos repartidos entre varias **zonas de disponibilidad** (_Availability Zones_, AZ), que son uno o más centros de datos físicamente separados dentro de una misma región de AWS, con energía y red independientes.

Amazon S3 puede usarse como almacenamiento de **fuente única de verdad** (_single source of truth_) para la mayoría de los servicios de ML de AWS. Esto significa que existe una sola copia autorizada de los datos a la que apuntan todos los servicios (Athena, Glue, el entrenamiento de SageMaker, el almacén offline de Feature Store), en lugar de copias que divergen en cada herramienta, y así se evita que un equipo entrene con una versión de los datos y otro equipo con otra. Además, con la clase de almacenamiento S3 Intelligent-Tiering puedes reducir el costo de almacenamiento dejando que AWS determine automáticamente cuándo mover los datos a la clase de almacenamiento más adecuada.

Las **clases de almacenamiento** de S3 son niveles de precio según la frecuencia con que se leen los datos: S3 Standard para acceso frecuente (el más caro por GB guardado y sin cargo por leer), S3 Standard-IA para acceso infrecuente (más barato por GB, pero con cargo por cada GB recuperado) y las clases Glacier para archivo (muy baratas, con recuperación más lenta o más cara). **S3 Intelligent-Tiering** observa el patrón de acceso de cada objeto y lo mueve solo entre niveles internos, sin que cambie la forma de leerlo: pasa al nivel de acceso infrecuente tras 30 días seguidos sin leerse y al de archivo con acceso instantáneo tras 90 días, y vuelve al nivel frecuente en cuanto alguien lo lee. Encaja con ML porque los datos crudos se leen intensamente al principio de un proyecto y después casi nunca, pero nadie sabe de antemano cuándo habrá que reentrenar con ellos.

> [!warning] Nota de precisión: S3 (verifica las cifras en la documentación vigente)
>
> - El 99.999999999 % es el **objetivo de diseño** de durabilidad que AWS publica para S3, no una garantía contractual: el acuerdo de nivel de servicio (SLA) de S3 cubre la disponibilidad, no la durabilidad. Las clases de una sola zona (por ejemplo, S3 One Zone-IA) guardan los datos en una única zona de disponibilidad y no resisten la pérdida de esa zona.
> - Intelligent-Tiering cobra una pequeña tarifa mensual de monitoreo por objeto, y los objetos de menos de 128 KB no se monitorean ni se mueven: se quedan siempre en el nivel frecuente. En datasets formados por millones de archivos diminutos, el ahorro puede ser nulo.

AWS Lake Formation complementa muy bien los servicios mencionados: centraliza los permisos sobre los datos, simplifica la gestión de la seguridad y la gobernanza a escala, monitorea el acceso a los datos y ayuda a asegurar el cumplimiento normativo (**_compliance_**). Compliance significa poder demostrar ante auditores o reguladores que se cumplen normas como el RGPD europeo, la HIPAA estadounidense para datos de salud o la regulación bancaria. Lake Formation ayuda de dos formas concretas. Primero, los permisos quedan declarados en un solo lugar, con una sintaxis de concesiones parecida al `GRANT` de SQL que un auditor puede revisar. Segundo, los usuarios no reciben permisos directos sobre el bucket: consultan a través de motores integrados (Athena, Amazon EMR, Amazon Redshift Spectrum, entre otros), a los que Lake Formation entrega credenciales temporales válidas solo para los datos autorizados, y cada acceso queda registrado en **AWS CloudTrail**, el servicio que guarda el historial de llamadas a la API de AWS de la cuenta.

En las próximas secciones aprenderás las técnicas que te permiten procesar los datos de tu data lake y dejarlos aptos para entrenar tu modelo de ML, de modo que produzca inferencias significativas.

El objetivo último de la fase de procesamiento de datos es producir datos de calidad para entrenar eficazmente tu modelo de ML, de modo que aprenda más rápido de tus datos y produzca predicciones precisas. Cuanto mayor sea la calidad de tu dataset de entrenamiento, más precisas serán las inferencias que produzca tu modelo de ML.

## Comprender la ingeniería de características (_Understanding Feature Engineering_)

Antes de profundizar en la ingeniería de características, centrémonos en entender los distintos tipos de datos:

- **Datos categóricos.** Los datos categóricos contienen un número finito de categorías distintas, cada una representada con una cadena de texto. Los datos categóricos pueden tener un orden lógico, como las tallas de una camisa: Small, Medium, Large, X-Large. A este tipo de dato categórico se le llama **ordinal**. En cambio, a los datos categóricos sin un orden lógico se les llama **nominales**. Por ejemplo, los 50 estados de EE. UU. son datos categóricos nominales.
- **Datos numéricos.** En ML, los datos numéricos son cualquier tipo de dato que pueda representarse con números. Esto incluye datos discretos y datos continuos.
  - Los **datos discretos** tienen un número contable de valores entre dos valores cualesquiera. Una variable discreta siempre es numérica, como el número de quejas de clientes o el número de fallas o defectos.
  - Los **datos continuos** tienen un número infinito de valores entre dos valores cualesquiera. Una variable continua puede ser numérica o de fecha y hora, como la longitud de una pieza o la fecha y hora en que se recibe un pago.
- **Datos textuales.** Los datos textuales son contenido escrito que puede procesarse y analizarse. Pueden incluir desde oraciones y párrafos de artículos o libros hasta publicaciones en redes sociales, reseñas y más. Se usan en diversas tareas de procesamiento de lenguaje natural (NLP), como el análisis de sentimiento, la traducción de idiomas y la clasificación de textos.
- **Datos de imagen.** Los datos de imagen consisten en valores de píxeles que pueden analizarse para extraer características como bordes, formas, colores y patrones. Se usan a menudo en tareas de visión por computadora, como la detección de objetos, el reconocimiento facial y la clasificación de imágenes.
- **Datos de series de tiempo.** Los datos de series de tiempo son una colección de observaciones o mediciones registradas a intervalos regulares de tiempo. En este tipo de datos, cada observación está asociada a una marca de tiempo (_timestamp_) o periodo específico, lo que crea una secuencia de puntos ordenada cronológicamente. El orden de los puntos es crucial para entender las tendencias, los patrones y las variaciones que ocurren a lo largo de ese periodo.

La capacidad de entender los distintos tipos de datos es un primer paso hacia nuestro objetivo de entrenar un modelo de ML. Es un paso necesario, considerando que los datos de tu data lake son la combinación de múltiples datasets ingeridos desde distintas fuentes. Cada dataset puede estar compuesto por datos estructurados, semiestructurados o no estructurados. E incluso si todos tus datasets estuvieran formados por datos estructurados, no está garantizado que el esquema sea el mismo para todos. Por eso estos datos necesitan transformarse adecuadamente para quedar listos para alimentar a un algoritmo de ML.

Esta última afirmación tiene una traducción directa a la infraestructura del capítulo anterior. Si el crawler de Glue encuentra en la misma carpeta de S3 archivos CSV con columnas distintas (porque un sistema de origen agregó un campo en marzo, por ejemplo), puede registrar varias tablas o una tabla con particiones de esquemas incompatibles, y Athena devolverá errores o columnas vacías. Por eso la transformación empieza por homogeneizar esquemas y formatos antes de cualquier otra cosa.

Aquí es donde entra en juego la ingeniería de características (**_feature engineering_**). La ingeniería de características es la ciencia (y el arte) de extraer más información de los datos existentes para mejorar el poder predictivo de tu modelo de ML y ayudarlo a aprender más rápido. Durante la ingeniería de características no agregas datos nuevos, sino que haces más útiles los datos que ya tienes. Al reorganizar tus datos en un conjunto de características que pueda alimentar directamente a un algoritmo de ML, tu modelo de ML producirá mejores inferencias. La ingeniería de características suele apoyarse en el conocimiento del dominio de los datos para que tu modelo de ML produzca resultados más eficaces.

Los tipos de datos que acabas de conocer (categóricos, numéricos, textuales, de imagen y de series de tiempo) determinarán el enfoque de ingeniería de características más apropiado.

### Definir las características (_Defining Features_)

En ML, las **características** (_features_) son propiedades o rasgos individuales y medibles de los datos que analizas. Cada atributo único de los datos se considera una característica (también llamada _atributo_). Piensa en ellas como las entradas que tu modelo usa para hacer predicciones. Por ejemplo, en un dataset de datos de ventas, las características podrían incluir la fecha, el número de visitantes y el número de pedidos (ventas).

La Figura 3.2 ilustra un dataset simplificado de datos de ventas.

_Figura 3.2 Ejemplo de un dataset._

Supón que tu modelo de ML usará estos datos para predecir las ventas de un día dado. A primera vista, habrás notado que las dos primeras filas tienen un número de ventas notablemente mayor que las tres filas restantes. Un análisis posterior indicó que los clientes tienden más a comprar los fines de semana. Por lo tanto, es el día de la semana lo que influye en los hábitos de compra. Así que podemos construir una característica que indique el día de la semana y luego escribir un script sencillo que complete ese dato automáticamente, como se muestra en la Figura 3.3. (El original usa aquí el verbo _impute_ en el sentido de «calcular y rellenar la columna nueva a partir de la fecha», no en el de rellenar valores faltantes, que aparece más adelante en el capítulo.)

_Figura 3.3 Agregar una característica a un dataset._

Este ejemplo sencillo muestra cómo la información «oculta» de tu dataset puede ayudar a tu modelo de ML a aprender más rápido. La ingeniería de características consiste en descubrir dónde está esa información y cómo transformar tu dataset para aprovecharla al máximo.

### Seleccionar características para el entrenamiento del modelo (_Selecting Features for Model Training_)

Las siguientes son las principales áreas de enfoque de la ingeniería de características:

- Extracción de características (_feature extraction_)
- Selección de características (_feature selection_)
- Creación y transformación de características (_feature creation and transformation_)

El objetivo de la extracción y la selección de características es reducir la **dimensionalidad** de tu dataset. El término dimensionalidad indica el número de características (o entradas) de tu dataset. Cuanto mayor es la dimensionalidad de un dataset, más difícil es entrenar eficazmente tu modelo de ML. A los modelos les cuesta encontrar los patrones que quieres que reconozcan cuando hay muchas dimensiones distintas de datos (muchas características) que revisar.

Por eso es importante realizar la extracción y la selección de características.

La dimensionalidad también tiene un costo de infraestructura que el libro no menciona: cada columna adicional ocupa espacio en S3, aumenta los datos que Athena escanea (y cobra) en cada consulta, alarga el tiempo que el trabajo de entrenamiento pasa leyendo datos y exige más memoria en la instancia que los procesa.

#### Extracción de características (_Feature Extraction_)

La extracción de características es el proceso de reducir automáticamente la dimensionalidad de tu dataset creando características nuevas a partir de las existentes. Es un proceso común con datasets que tienen un gran número de características, y aparece sobre todo cuando se trabaja con datos de imagen, audio o texto.

La Figura 3.4 muestra un ejemplo de reconocimiento de imágenes.

_Figura 3.4 Ejemplo de reconocimiento de imágenes._

Antes de la llegada de las redes neuronales, una de las formas de analizar datasets de imágenes era extraer características de cada imagen. Si la imagen es un auto, extraes algunos de sus aspectos útiles, como el parabrisas, los faros, las direccionales y las llantas, como características independientes. Así, en lugar de que tu dataset esté formado por píxeles crudos, tienes características o columnas como `windshield_present` y `headlight_present`, como se muestra en la Figura 3.5. Estas características facilitarán que el algoritmo de ML aprenda de los datos de imagen y, con el tiempo, empiece a reconocer rostros.

_Figura 3.5 Extracción de características de una imagen._

> [!warning] Nota de precisión: ¿autos o rostros?
> Todo el ejemplo trata de reconocer **autos** (parabrisas, faros, llantas), pero el original cierra diciendo que el algoritmo «empezará a reconocer rostros». Parece un resto de otro ejemplo; lo coherente es «empezará a reconocer autos».

En la mayoría de los casos, los propios datos te ayudarán a determinar qué técnica específica de extracción de características usar.

Para datos de imagen, podría ser extraer rasgos clave, como vimos antes. En NLP, podría ser extraer características útiles como las palabras más frecuentes del texto, sin contar artículos ni preposiciones.

#### Selección de características (_Feature Selection_)

La selección de características es otra técnica para reducir la dimensionalidad de tu dataset y se usa con frecuencia junto con la extracción de características.

La selección de características ordena las características existentes del dataset según su importancia predictiva y se queda solo con las más relevantes según ese orden.

Como tu dataset contiene datos crudos almacenados en un data lake, es probable que algunas características sean más importantes que otras para la precisión del modelo. Algunas también serán redundantes por estar correlacionadas con otras características. En el ejemplo de la Figura 3.6, los ingresos (_revenue_) se mueven en paralelo a las ventas, así que es poco probable que aporten al modelo mucha más información de la que ya obtiene de los datos de ventas.

_Figura 3.6 Selección de características._

Hay que eliminar las características irrelevantes para el problema. La selección de características resuelve estos problemas filtrando del dataset las características irrelevantes o redundantes. Como resultado, el algoritmo de ML solo recibe un subconjunto de las características más útiles para el problema.

Los algoritmos de filtrado pueden usar una medida estadística para identificar las características que tienen una relación fuerte con la variable objetivo. En el ejemplo, la variable objetivo que queremos que prediga nuestro modelo de ML son las ventas de un día dado, así que las utilidades netas de las ventas probablemente no son relevantes para esa predicción. Por lo tanto, podemos eliminar esa característica.

Recuerda que los algoritmos de ML no se usan solo con los datasets estructurados típicos. A menudo trabajamos con datasets no estructurados, en forma de imágenes, audio o video, por ejemplo. Estos formatos de datos requieren técnicas de filtrado para reducir la dimensionalidad del dataset.

#### Creación y transformación de características (_Feature Creation and Transformation_)

A diferencia de la extracción y la selección de características, la creación y transformación de características no es una técnica de reducción de dimensionalidad. Es el proceso de generar características nuevas a partir de las existentes. Por ejemplo, supongamos que tenemos «fecha» como característica, con formato de día, mes y año de dos dígitos cada uno (dd-mm-aa).

Podrías descubrir que combinar día, mes y año en una sola característica no ayuda mucho a tus predicciones. En su lugar, podrías generar tres características distintas, una para el día, otra para el mes y otra para el año, y así descubrir potencialmente una relación significativa entre una de ellas y la variable objetivo que quieres que prediga tu modelo de ML.

### Uso de Amazon SageMaker Feature Store (_Using Amazon SageMaker Feature Store_)

Como las características son las entradas (o variables) de los modelos de ML durante el entrenamiento, ¿no sería útil tener un lugar central donde seleccionar, refinar, almacenar y gestionar todas las características de tu dataset?

Aquí es donde entra en juego Amazon SageMaker Feature Store. Antes, dos términos que el libro da por conocidos. **Amazon SageMaker** es la plataforma de ML de AWS: reúne entornos de notebooks, herramientas de preparación de datos, entrenamiento, despliegue y monitoreo de modelos (hoy se llama Amazon SageMaker AI; ver el recuadro del inicio). Un servicio **_fully managed_** (totalmente administrado) es uno cuya infraestructura opera AWS: los servidores, el almacenamiento, la replicación entre zonas de disponibilidad, los parches de seguridad y el escalado. Tú lo usas desde la consola web, la API o un SDK y pagas por uso, sin acceso a las máquinas subyacentes. Es la diferencia entre instalar y mantener PostgreSQL en tu propio servidor y contratar una base de datos que otro mantiene.

Amazon SageMaker Feature Store es un repositorio fully managed para almacenar, compartir y gestionar características de modelos de ML. Por ejemplo, en una aplicación que recomienda libros sobre un tema dado, las características podrían incluir la afinidad del libro con el tema, las calificaciones del libro y su fecha de publicación.

Las características las usan constantemente varios equipos (científicos de datos, así como ingenieros de datos y de ML), y su calidad es crítica para la precisión de tu modelo de ML. Además, cuando las características que se usan para entrenar modelos offline, por lotes, se ponen a disposición de la inferencia en tiempo real, es difícil mantener sincronizados los dos almacenes de características. Amazon SageMaker Feature Store proporciona un almacén seguro y unificado para procesar, estandarizar y usar características a escala a lo largo del ciclo de vida de ML.

La frase sobre «los dos almacenes» es el centro de la sección y el libro no explica de dónde salen. Son dos necesidades de infraestructura opuestas:

- Para **entrenar** se necesita el **historial**: millones de filas, con el valor que tenía cada característica en cada momento pasado, leídas de una sola vez. Eso encaja con archivos en S3 consultados con Athena, donde una consulta puede tardar segundos sin problema.
- Para responder **en tiempo real** (por ejemplo, un endpoint que decide si aprueba un pago) se necesita el **valor actual** de las características de un solo cliente, leído en milisegundos. Eso exige una base de datos de clave-valor de baja latencia, no una consulta analítica sobre S3.

Construido a mano, esto son dos sistemas distintos (una base de datos rápida y un almacén histórico) y dos procesos que escriben en ellos. Si uno falla o se retrasa, los valores divergen, y el modelo recibe en producción datos distintos de los que vio al entrenar sin que nada dé error.

Feature Store resuelve esa dualidad con estos elementos:

- **Grupo de características (_feature group_).** Es el recurso principal: una especie de tabla con una definición de columnas (nombre y tipo: entero, fraccionario o cadena). Cada grupo declara qué columna es el **identificador de registro** (_record identifier_, por ejemplo `customer_id`) y qué columna es la **hora del evento** (_event time_), es decir, el momento en que esos valores eran válidos.
- **Almacén online (_online store_).** Guarda solo el registro más reciente de cada identificador y responde lecturas individuales con baja latencia mediante la API `GetRecord`. Se elige entre varios tipos de almacenamiento: `Standard` (el predeterminado), `Standard_V2` (permite actualizar una sola característica sin reescribir el registro completo) e `InMemory`, que está respaldado por Amazon ElastiCache (una caché en memoria) y ofrece latencia todavía menor para aplicaciones de muy alto tráfico.
- **Almacén offline (_offline store_).** Vive en un bucket de S3 **tuyo** y conserva **todos** los registros históricos, en archivos Parquet registrados como tabla en el Data Catalog de Glue (en formato de tabla Glue o Apache Iceberg; AWS recomienda Iceberg porque permite compactar los muchos archivos pequeños en pocos grandes y acelera las consultas). Se consulta con Athena para construir datasets de entrenamiento o para inferencia por lotes.
- **Ingesta única.** Tu código escribe cada registro una sola vez (API `PutRecord`, o exportando desde Data Wrangler), y si el grupo tiene ambos almacenes activados, Feature Store los mantiene sincronizados. Esa es la respuesta concreta a la dificultad que plantea el libro: ya no hay dos procesos que puedan divergir.
- **Consultas _point-in-time_.** Como el almacén offline conserva la hora del evento de cada registro, se puede reconstruir qué valor tenía una característica en un instante pasado concreto y armar el dataset de entrenamiento con los valores vigentes en la fecha de cada ejemplo, no con los actuales.
- **Descubrimiento y acceso.** Los grupos se pueden buscar por nombre, descripción y etiquetas desde Studio, de modo que otro equipo encuentre y reutilice características ya construidas. El acceso se controla con IAM, los datos se cifran en reposo con claves de KMS y, para no salir a internet, las aplicaciones dentro de una VPC (la red privada de tu cuenta en AWS) pueden llegar a Feature Store por un endpoint privado de AWS PrivateLink.

> [!warning] Nota de precisión: límites del nivel en memoria (verificado el 25-09-2026)
> Según la documentación vigente, un grupo con almacén online `InMemory` **no** puede tener almacén offline asociado (no hay replicación entre ambos), no admite claves de KMS administradas por el cliente y tiene un tamaño máximo predeterminado de 50 GiB. Si necesitas historial para entrenar y lectura en tiempo real sincronizados, el tipo que corresponde es `Standard` o `Standard_V2` con el almacén offline activado.

En la próxima sección cubriremos el trabajo de preprocesamiento necesario antes de la ingeniería de características. Este trabajo empieza por transformar tu dataset para abordar problemas de inconsistencia de los datos, como gestionar los valores atípicos, imputar los datos faltantes y eliminar duplicados.

## Limpieza y transformación de datos (_Data Cleaning and Transformation_)

La limpieza de datos es un paso fundamental de preprocesamiento que asegura que tu dataset esté en la mejor forma posible para la ingeniería de características.

Con un enfoque eficaz de limpieza de datos, preparas el terreno para la ingeniería de características transformando datos «sucios» en una base estructurada, precisa y confiable. Esto te permite concentrarte en crear y seleccionar las características más relevantes y potentes para tu modelo de ML.

Así es como la limpieza proporciona la base necesaria:

- **Gestión de valores faltantes.** Este paso aborda cualquier hueco de tu dataset, ya sea imputando los valores faltantes, eliminando los registros incompletos o gestionándolos de otro modo. Unos datos limpios aseguran que las características que construyas no sufran sesgos ni imprecisiones causados por huecos en los datos.
- **Detección y tratamiento de valores atípicos.** Al identificar y gestionar los valores atípicos, puedes evitar que distorsionen la distribución de las características y afecten el rendimiento del modelo. Unos datos limpios ofrecen una representación más realista de la distribución de los datos, lo cual es crucial para una ingeniería de características eficaz.
- **Deduplicación.** Eliminar registros duplicados y puntos de datos redundantes ayuda a depurar el dataset, lo que hace la ingeniería de características más eficiente y menos propensa al sobreajuste.
- **Reformateo y estandarización.** Asegurar tipos de datos, unidades y formatos uniformes facilita la creación de características. Por ejemplo, estandarizar los formatos de fecha y las unidades de medida asegura la consistencia y la comparabilidad entre características.
- **Eliminación de ruido y errores.** La limpieza de datos implica identificar y corregir errores o inconsistencias en los datos. Esto asegura que las características que crees se basen en datos precisos y confiables, lo que produce modelos más robustos.

### Gestión de valores faltantes (_Managing Missing Values_)

No es raro que tu dataset tenga datos faltantes. Por ejemplo, a algunas columnas de tu dataset pueden faltarles valores por un error en la recolección de datos, o quizá el dato simplemente no estaba disponible en tu fuente de datos.

Los datos faltantes pueden dificultar que el algoritmo de ML elegido interprete con precisión la relación entre la característica afectada y la variable objetivo. Por eso es importante abordar el problema.

Por desgracia, la mayoría de los algoritmos de ML no pueden manejar automáticamente los valores faltantes. Se requiere conocimiento humano para reemplazar los valores faltantes con algo significativo y relevante para el problema.

Los enfoques para abordar los datos faltantes varían según la cantidad de datos faltantes. Las principales técnicas son las siguientes:

- **Recolectar.** Si la cantidad de datos faltantes es considerable y afecta a varias características, deberías intentar volver a recolectar los datos. Sin embargo, el proceso de ingesta de datos puede ser caro y lento. En ese caso, necesitas determinar cuáles son los compromisos entre el costo de volver a recolectar los datos, el costo de tener un modelo de ML de bajo rendimiento y alternativas como la imputación o la eliminación. En AWS, «volver a recolectar» tiene costos concretos: volver a ejecutar los trabajos de ingesta del capítulo 2 (horas de cómputo de Glue, transferencia de datos desde el centro de datos de origen, tarifas por petición de S3) y, en el peor caso, pedir a los dueños de los sistemas de origen una nueva extracción. Si el dato crudo original sigue en el lago, a veces basta con reprocesarlo; por eso conviene no borrar nunca la zona cruda.
- **Imputar.** Si tienes una cantidad pequeña de valores faltantes, distribuidos aleatoriamente entre las características de tu dataset, podría deberse a una falla en el proceso de ingesta de datos. En ese caso, la imputación probablemente sea una buena opción. Con la imputación, rellenas los datos faltantes con la media, la mediana o el valor más frecuente observado para tu característica. Si la distribución de tu característica es normal, usa la media; si no, usa la mediana o el valor más frecuente.
- **Eliminar.** Si la cantidad de datos faltantes es considerable pero se limita a la misma característica, podrías considerar eliminar la característica completa. Sin embargo, hay que tener cuidado: si eliminas demasiadas características, quizá no tengas datos suficientes para alimentar el modelo de ML.

Con Amazon SageMaker puedes usar bibliotecas de Python como `SimpleImputer` de `sklearn` para rellenar valores faltantes. La frase merece una aclaración: `SimpleImputer` no es una función de AWS, sino de scikit-learn. «Con SageMaker» significa que el código corre en un **notebook de Jupyter alojado por SageMaker**, en una de dos formas: un espacio de JupyterLab dentro de SageMaker Studio o una _notebook instance_ independiente. En ambos casos tú eliges el **tipo de instancia**, es decir, la máquina virtual con cierta cantidad de CPU, memoria y, si hace falta, GPU (por ejemplo, `ml.t3.medium` para explorar o `ml.m5.4xlarge` para trabajar con datos más grandes), y SageMaker la aprovisiona con Python y las bibliotecas habituales ya instaladas. Se cobra por hora mientras está encendida, así que conviene apagarla al terminar. Hay una consecuencia práctica que el libro no menciona: pandas y scikit-learn cargan todo el dataset en la **memoria RAM de esa instancia**; si el dataset no cabe, hay que elegir una instancia más grande o pasar a un motor distribuido, como un trabajo de Glue con Spark (ver «Estandarización y reformateo»).

El siguiente fragmento muestra cómo funciona esta función:

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

Salida real al ejecutar el fragmento (el resultado es el mismo, impreso en punto flotante):

```text
[[1. 2.]
 [3. 4.]
 [5. 3.]]
```

AWS Glue DataBrew también puede usarse para rellenar valores faltantes mediante varias transformaciones integradas. **AWS Glue DataBrew** es una herramienta visual de preparación de datos, sin código, de la familia AWS Glue. Funciona así:

1. Creas un **proyecto** que se conecta a tus datos (en S3, en el Data Catalog de Glue, en Amazon Redshift o en bases de datos accesibles por red). DataBrew carga una **muestra** del dataset y la muestra en una cuadrícula parecida a una hoja de cálculo, con la distribución de valores de cada columna.
2. Eliges transformaciones con clics (AWS habla de más de 250 transformaciones prediseñadas; verifica la cifra vigente) y ves de inmediato el antes y el después sobre la muestra. Cada transformación queda guardada como un paso de una **receta** (_recipe_), reutilizable con otros datasets.
3. Ejecutas la receta sobre el **dataset completo** como un **trabajo** (_job_) de DataBrew, que puede programarse (por ejemplo, cada noche) y escribe el resultado en S3.

DataBrew es **serverless**: no hay clúster que crear ni máquinas que administrar. Se cobra por las sesiones interactivas del proyecto y por el tiempo de cómputo de los trabajos (consulta la página de precios vigente). Para valores faltantes ofrece, entre otros, pasos de receta que rellenan con la media, la mediana, la moda, un valor fijo o el último valor válido (`FILL_WITH_AVERAGE`, `FILL_WITH_MEDIAN`, `FILL_WITH_MODE`, `FILL_WITH_CUSTOM`, `FILL_WITH_LAST_VALID`) o que eliminan las filas afectadas (`REMOVE_MISSING`).

### Detección y tratamiento de valores atípicos (_Detecting and Treating Outliers_)

Un valor atípico (_outlier_) es un punto de tu dataset que se distingue de todos los demás por tener una desviación significativa respecto de la media.

¿Qué significa exactamente «significativamente»? Aunque la respuesta depende del tipo de datos y de su distribución de probabilidad, el consenso general es que un punto se considera atípico si se encuentra a más de tres desviaciones estándar de la media. Esto se basa en las propiedades de la distribución normal, en la que aproximadamente el 99.7 % de los datos se encuentra a menos de tres desviaciones estándar de la media.

Los algoritmos de ML son muy sensibles a la distribución y al rango de los valores de tus características. Como los valores atípicos se apartan del patrón de todos los demás puntos, tienden a engañar al algoritmo de ML durante el entrenamiento.

Por ejemplo, considera el siguiente dataset:

```text
[x,y] = [[4, 11], [3.8, 12], [4.5, 12.5], [8, 8], [9, 8.5], [9.5, 7.5], [13, 5], [14, 4.7], [13.7, 6], [25, 23]]
```

En este ejemplo, supón que x indica tu tasa de consumo diario de agua, mientras que y indica tu tasa de consumo diario de energía. Estos números son solo ilustrativos.

Como puedes ver en la Figura 3.7, este dataset se distribuye en tres grupos.

_Figura 3.7 Ejemplo de un valor atípico._

Un punto destaca claramente y actúa como valor atípico.

Aunque algunos valores atípicos se deben a errores artificiales, otros simplemente aparecen en tu dataset como resultado de fenómenos naturales. Un atípico natural no es el resultado de algún error artificial, sino que refleja alguna verdad presente en los datos.

Tu trabajo como ingeniero de ML es determinar si un valor atípico debe quedarse en tu dataset o si hay que cambiarlo o eliminarlo. Tomas esa decisión apoyándote en técnicas estadísticas que te ayudan a entender qué tan relevante es el valor atípico respecto a los demás puntos de tu dataset.

Es importante distinguir entre ruido y valores atípicos. El primero denota un grupo de puntos erróneos, mientras que los segundos son puntos que se desvían significativamente de la media de tu dataset. En el resto de los capítulos aprenderás a aprovechar el amplio ecosistema de servicios de ML de AWS para determinar qué significa «significativamente».

El tratamiento de los valores atípicos se apoya en tres enfoques principales:

- **Eliminar.** Si los valores atípicos se deben a ruido o a errores artificiales, puedes simplemente eliminarlos de tu dataset sin afectar la calidad ni la precisión de tu modelo de ML.
- **Transformación logarítmica.** Al reemplazar el valor atípico por su logaritmo en una base dada (p. ej., base _e_, base 2, base 10, etc.), comprimes el rango de valores de una característica, lo que reduce la variación extrema entre los valores. Como resultado, el valor atípico no quedará tan lejos de los demás valores de esa característica. Por ejemplo, el logaritmo en base 10 ($\log_{10}$) de 1000 es 3. Eso se debe a que $10^3 = 1000$, lo que lo acerca a otros valores más pequeños sin que pierda su significado. _(Fórmulas y base e reconstruidas; el `.txt` las perdió.)_
- **Imputar.** Igual que con los valores faltantes, podrías usar, por ejemplo, la media de la característica e imputar ese valor en lugar del valor atípico. Este sería un enfoque excelente si el valor atípico se hubiera debido a errores artificiales.

Los valores atípicos suelen crear una distribución sesgada, como se muestra en la Figura 3.7. La transformación logarítmica puede ayudar a normalizar esa distribución, haciéndola más simétrica y mejorando el rendimiento de tu modelo de ML.

La Figura 3.8 ilustra el mismo dataset tras representar el eje de ordenadas _y_ con una escala logarítmica. El punto atípico [25, 23] tiene x = 25 e y = 23. Como $\log_{10}(23) \approx 1.36$, puedes ver que este punto ya no se desvía significativamente de la media. _(Fórmula reconstruida; se asume base 10 por coherencia con el ejemplo anterior.)_

_Figura 3.8 Tratamiento de valores atípicos con transformación logarítmica._

Para el examen, necesitas conocer los servicios de AWS que te ayudan a detectar y tratar valores atípicos, que son Amazon SageMaker Data Wrangler y AWS Glue DataBrew.

**Amazon SageMaker Data Wrangler** es la herramienta visual de SageMaker para importar, explorar, transformar y analizar datos con poco o nada de código (**_low-code_**: la mayor parte del trabajo consiste en elegir y configurar pasos en una interfaz, y solo se escribe código donde hace falta). Hoy se usa desde SageMaker Canvas. Construyes un **flujo de datos** (_data flow_): una secuencia de pasos (importar, unir, transformar, analizar) que se diseña sobre una muestra y después se aplica al dataset completo, ya sea exportando el resultado a S3 o a Feature Store, convirtiéndolo en un paso de SageMaker Pipelines (el servicio de SageMaker para encadenar automáticamente preparación de datos, entrenamiento y despliegue) o generando un script de Python. Su grupo de transformaciones _Handle outliers_ detecta atípicos por desviación estándar, por desviación estándar robusta, por cuantiles o por umbrales mínimo y máximo, y los corrige recortándolos al límite, eliminando la fila o marcándolos como inválidos.

> [!note] Un detalle de funcionamiento de Data Wrangler (según la documentación vigente)
> Los estadísticos que usa _Handle outliers_ (media, desviación estándar, cuantiles) se calculan con los datos cargados en Data Wrangler **cuando defines el paso**, que suelen ser una muestra, y esos mismos valores se reutilizan cuando el flujo se ejecuta después como trabajo sobre el dataset completo. Si la muestra no es representativa, los límites tampoco lo serán.

Puedes usar los notebooks de Jupyter de Amazon SageMaker para cargar tu dataset con bibliotecas de Python como pandas.

Para determinar qué algoritmo de detección de atípicos usar, necesitas analizar el histograma de cada característica. La Figura 3.9 muestra los histogramas de nuestro dataset, generados con el siguiente fragmento de Python:

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

En la Figura 3.9, los histogramas muestran una distribución de datos sesgada a la derecha en ambas características. Una distribución sesgada a la derecha significa que la mayoría de nuestros puntos se concentran hacia el extremo inferior del rango, con una cola larga que se extiende hacia la derecha. Esa cola representa un pequeño número de puntos con valores mucho más altos que el resto del dataset. En esencia, la mayoría de los valores son bajos y solo unos pocos son excepcionalmente altos, que es exactamente nuestro escenario, con el punto [25, 23] como el par de valores más alto.

Para datos sesgados, el método del rango intercuartílico (IQR, _interquartile range_) suele considerarse uno de los mejores enfoques para detectar valores atípicos. El IQR se ve menos afectado por el sesgo y proporciona una medida robusta para identificar valores extremos.

Si $Q_1$ y $Q_3$ denotan el primer y el tercer cuartil de nuestro dataset, esta es una descripción rápida de cómo funciona _(fórmulas reconstruidas en su forma estándar)_:

1. Calcula $Q_1$ y $Q_3$.
2. Calcula $\text{IQR} = Q_3 - Q_1$.
3. Todo punto cuyo valor sea menor que $Q_1 - 1.5 \cdot \text{IQR}$ o mayor que $Q_3 + 1.5 \cdot \text{IQR}$ se considera atípico.

En nuestro ejemplo, podemos aprovechar el método integrado `quantile`, disponible en la clase `DataFrame` de las bibliotecas numpy y pandas, para detectar los atípicos del dataset, como se ilustra en el siguiente fragmento:

> [!warning] Nota de precisión: `DataFrame` es de pandas
> NumPy no tiene una clase `DataFrame`. El método `DataFrame.quantile` pertenece a pandas; la función equivalente de NumPy es `np.quantile`, que trabaja sobre arreglos.

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

Salida real:

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

Cleaned Data without Outliers:
   Feature1  Feature2
0       4.0      11.0
1       3.8      12.0
2       4.5      12.5
3       8.0       8.0
4       9.0       8.5
5       9.5       7.5
6      13.0       5.0
7      14.0       4.7
8      13.7       6.0
```

Para datos con distribución normal, el método de la puntuación Z (_Z-score_) es muy eficaz para detectar valores atípicos. Este es un resumen rápido _(fórmulas reconstruidas en su forma estándar)_:

1. Calcula la media $\mu$ de tu dataset.
2. Calcula la desviación estándar $\sigma$ de tu dataset.
3. Para cada punto $x_i$, la puntuación Z es $z_i = \dfrac{x_i - \mu}{\sigma}$.

Como una distribución normal se caracteriza por lo siguiente:

- Una desviación estándar cubre alrededor del 68 % de los datos.
- Dos desviaciones estándar cubren alrededor del 95 % de los datos.
- Tres desviaciones estándar cubren alrededor del 99.7 % de los datos.

los puntos con puntuaciones Z mayores que 3 o menores que −3 se consideran atípicos. En otras palabras, es estadísticamente seguro suponer que el 0.3 % de los puntos cuya puntuación Z es mayor que 3 o menor que −3 están lo bastante lejos de la media como para considerarse atípicos.

El método de detección de atípicos por puntuación Z también puede calcularse con código. El siguiente fragmento muestra cómo:

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

Salida real (la segunda tabla, _Cleaned Data_, repite las diez filas sin la columna `Outlier`):

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
9      25.0      23.0    False
```

Como puedes ver, el método de la puntuación Z no detectó el atípico [25, 23]. Esto se debe a que la distribución de los puntos de nuestro dataset no es normal, sino sesgada a la derecha.

> [!warning] Nota de precisión: con 10 datos, ninguna puntuación puede superar 3
> Con la desviación estándar que usa `scipy.stats.zscore` por defecto (`ddof=0`), la mayor puntuación |z| posible en una muestra de tamaño _n_ es $(n-1)/\sqrt{n}$, que para _n_ = 10 vale 2.85. Ningún punto de este dataset podía superar el umbral de 3, fuera cual fuera su valor, así que el fallo no se explica solo por el sesgo de la distribución.

Con AWS Glue DataBrew también puedes detectar valores atípicos en tus datos y manejarlos con varias transformaciones. Este enfoque incluye reemplazar, eliminar, reescalar o marcar los valores atípicos usando los métodos que acabas de aprender, es decir, la puntuación Z y el IQR. En la documentación vigente de DataBrew, estos pasos de receta aparecen como `REMOVE_OUTLIERS`, `REPLACE_OUTLIERS`, `FLAG_OUTLIERS`, `RESCALE_OUTLIERS_WITH_Z_SCORE` y `RESCALE_OUTLIERS_WITH_SKEW`, y admiten tres estrategias de detección: `Z_SCORE`, `MODIFIED_Z_SCORE` (basada en la mediana) e `IQR`.

¿Cuándo usar uno u otro servicio? El libro los presenta como equivalentes para esta tarea. La diferencia práctica está en quién los usa y dónde viven: DataBrew es un servicio de la familia Glue, pensado también para analistas de datos que no trabajan en SageMaker, y cobra por sesión y por trabajo; Data Wrangler vive dentro de SageMaker (Canvas), exige un dominio de SageMaker configurado y se integra directamente con Feature Store y con Pipelines.

### Deduplicación (_Performing Deduplication_)

La deduplicación es el proceso de eliminar los puntos duplicados de tu dataset. No se trata solo de ordenar tus datos: se trata de hacer que tus modelos sean más precisos, tengan mejor rendimiento y sean más confiables.

Desde el punto de vista de la calidad de los datos, los datos duplicados pueden distorsionar los resultados y llevar a predicciones inexactas. Eliminar los duplicados asegura que los datos usados para entrenar los modelos estén limpios y sean precisos.

El rendimiento y la confiabilidad del modelo son factores clave que hay que tener en cuenta en las fases de evaluación y despliegue del ciclo de vida de ML. Los duplicados pueden causar sobreajuste, en el que el modelo aprende el ruido en lugar de la señal. Los datos limpios y deduplicados ayudan a crear modelos más generalizables.

Desde el punto de vista de la confiabilidad, los duplicados pueden introducir sesgo, en particular si ciertas entradas se repiten más que otras, lo que produce datos de entrenamiento desbalanceados.

Por todo ello, una estrategia de deduplicación eficaz es crítica para asegurar la calidad de los datos de entrenamiento de tu modelo de ML.

¿De dónde salen los duplicados en un data lake? El libro no lo dice, y la causa suele ser de infraestructura. Muchos servicios de ingesta garantizan entrega «al menos una vez» (_at-least-once_): ante un fallo de red, reenvían el mensaje en lugar de arriesgarse a perderlo, así que el mismo evento puede llegar dos veces. También es común que un trabajo de carga nocturno falle a la mitad y se relance completo, o que dos sistemas de origen exporten el mismo cliente. Por eso la deduplicación es un paso habitual en cualquier canalización que lee del lago.

Por simplicidad y facilidad de uso, AWS Glue DataBrew es probablemente tu mejor opción para eliminar duplicados de tu dataset. AWS Glue DataBrew ofrece una interfaz visual y directa para limpiar y preparar tus datos sin necesidad de código complejo. Es fácil de usar y eficiente, lo que hace que tareas de limpieza como eliminar duplicados sean rápidas y sin complicaciones. En la documentación vigente, los pasos de receta correspondientes son `DELETE_DUPLICATE_ROWS` y `REMOVE_DUPLICATES`, y hay variantes que solo marcan los duplicados sin borrarlos (`FLAG_DUPLICATE_ROWS`), útiles para revisarlos antes de decidir. El «probablemente» del libro se explica porque hay otras vías igual de válidas en AWS: Data Wrangler ofrece un fragmento de código listo para eliminar filas duplicadas en su transformación personalizada, y un trabajo de Glue con Spark lo hace con una sola instrucción (`dropDuplicates`). DataBrew gana cuando quien limpia los datos no programa.

### Estandarización y reformateo (_Standardizing and Reformatting_)

Al aplicar la estandarización, evitas que las características con valores grandes influyan en el modelo de forma desproporcionada en comparación con las características con valores pequeños. Esta técnica es particularmente importante para los algoritmos de ML sensibles a la escala de las características, como la regresión lineal y las máquinas de vectores de soporte, que se cubrirán en el próximo capítulo.

Reformatear, en ML, significa reestructurar tus datos en un formato consistente y adecuado para el análisis. Incluye convertir tipos de datos, armonizar formatos y asegurar que todas las entradas sigan la misma estructura. Un reformateo muy común en AWS es convertir los CSV o JSON crudos en **Parquet**, un formato de archivo por columnas y comprimido: ocupa menos en S3, y Athena y Spark pueden leer solo las columnas que necesitan, lo que abarata y acelera cada consulta posterior.

Para realizar la estandarización y el reformateo con AWS, puedes usar Amazon SageMaker y AWS Glue. Los pasos principales son los siguientes:

- **Cargar los datos.** Puedes usar notebooks de SageMaker para cargar tu dataset desde un bucket de S3, como se muestra en el siguiente fragmento:

  ```python
  import pandas as pd
  data = pd.read_csv('your_dataset.csv')
  ```

  Así escrito, el fragmento lee un archivo del disco local de la instancia, no de S3. Para leer directamente desde S3 se usa una ruta del tipo `'s3://nombre-del-bucket/ruta/your_dataset.csv'`. pandas la resuelve si se cumplen dos condiciones: que esté instalada la biblioteca `s3fs` (la que traduce la ruta `s3://` a llamadas a la API de S3) y que el **rol de ejecución** del notebook, el rol de IAM con el que el notebook actúa ante AWS, tenga permiso `s3:GetObject` sobre ese bucket. Si falta el permiso, el error será un `AccessDenied`, no un problema de pandas.

- **Estandarizar los datos.** Puedes usar la clase `StandardScaler` del módulo `sklearn.preprocessing` para estandarizar tus características, como se muestra en el siguiente fragmento:

  ```python
  from sklearn.preprocessing import StandardScaler
  scaler = StandardScaler()
  data_scaled = scaler.fit_transform(data)
  ```

  Más técnicas de normalización y estandarización se cubrirán en las próximas secciones.

- **Exportar los datos.** Puedes guardar el dataset estandarizado en S3, por ejemplo con `df.to_csv('s3://...')` (mismos requisitos que para leer, pero con permiso `s3:PutObject`) o con **boto3**, el SDK de AWS para Python. Un **SDK** (_software development kit_) es la biblioteca que permite manejar los servicios de AWS desde código en lugar de hacerlo desde la consola web.
- **Crear un trabajo de AWS Glue.** Crea un trabajo (_job_) de AWS Glue para reformatear los datos según sea necesario. Un **Glue job** es un script, normalmente en PySpark (la interfaz de Python de Apache Spark, un motor de procesamiento distribuido que reparte los datos entre varias máquinas y las procesa en paralelo), que Glue ejecuta en infraestructura serverless. Tú defines el script y la capacidad en **DPU** (_Data Processing Units_; según la documentación, una DPU equivale a 4 vCPU y 16 GB de memoria), y Glue levanta las máquinas, corre el trabajo, las apaga y cobra por DPU-hora consumida. Como ejemplo de escala: un notebook con 16 GB de RAM no puede cargar en pandas un dataset de 200 GB, mientras que un trabajo de Glue con suficientes DPU lo procesa por partes sin que nadie administre un clúster.
- **Cargar los datos limpios.** Usa los datos limpios y estandarizados para entrenar tu modelo de ML.

El libro combina las dos herramientas sin explicar el reparto. Lo habitual es que el notebook sirva para explorar y probar la transformación sobre una muestra, y que el Glue job ejecute esa misma lógica de forma repetible sobre el volumen completo, por ejemplo cada noche, sin que nadie tenga un notebook abierto. Para que el trabajo pueda leer y escribir en S3, necesita su propio rol de IAM con esos permisos.

### Eliminación de ruido y errores (_Removing Noise and Errors_)

El ruido natural y los errores artificiales se presentaron brevemente antes. Como el objetivo principal de la ingeniería de características es producir la mejor cantidad posible de datos de entrenamiento para tu modelo de ML, uno de los aspectos clave de este proceso es la capacidad de manejar el ruido y los errores.

Al crear características nuevas o transformar las existentes, haces que los datos de entrenamiento sean más informativos y relevantes para tu modelo de ML. Este proceso ayuda a mejorar la precisión, la eficiencia y la capacidad de generalización del modelo, lo que aumenta su poder predictivo.

Un modelo entrenado con datos ruidosos a menudo aprende peculiaridades o errores específicos del dataset de entrenamiento, en lugar de los patrones subyacentes. Esto lleva al sobreajuste (_overfitting_), en el que el modelo rinde muy bien con los datos de entrenamiento, pero mal con datos nuevos, no vistos. Al manejar adecuadamente el ruido y los errores, ayudas a asegurar que el modelo generalice mejor en escenarios del mundo real. En el capítulo 5 cubriremos en detalle los métodos para abordar el sobreajuste.

## Técnicas de ingeniería de características (_Feature Engineering Techniques_)

En la sección anterior se presentaron los métodos para limpiar tu dataset. La limpieza de datos es un paso preliminar hacia la preparación de los datos de entrenamiento de tu modelo.

El objetivo del siguiente paso es transformar eficazmente los datos crudos en características significativas que mejoren el rendimiento de tu modelo de ML. Este es el foco principal de la ingeniería de características. En última instancia, después de la ingeniería de características, tu modelo de ML estará listo para el entrenamiento.

Dominar la ingeniería de características es crucial para demostrar tu experiencia en la construcción y el despliegue de soluciones de ML escalables y eficientes en AWS. Las próximas secciones cubren las técnicas de ingeniería de características que necesitas conocer para el examen.

Empezaremos con las técnicas para datos estructurados y después seguiremos con las técnicas para datos no estructurados, en forma de imágenes, texto y series de tiempo.

### Técnicas para datos estructurados (_Techniques for Structured Data_)

Con datos estructurados, puedes usar varias técnicas de extracción de características para reducir la dimensionalidad de tu dataset. Algo con lo que debes tener cuidado al usar la extracción de características es que, cuando pongas este modelo en producción o automatices la canalización, estas características puedan replicarse fácilmente y aun así reduzcan las altas dimensiones de los datos.

La advertencia de «replicarse fácilmente» es un problema de infraestructura, y el libro no lo desarrolla. En producción, la aplicación envía datos crudos a un **endpoint** (el punto de acceso HTTPS donde SageMaker sirve el modelo), y alguien tiene que aplicarles exactamente las mismas transformaciones que se aplicaron al entrenar, con los mismos parámetros. En AWS hay tres formas habituales de lograrlo: exportar el flujo de Data Wrangler como un **pipeline de inferencia en serie** (_serial inference pipeline_: un único endpoint que ejecuta en cadena un contenedor de preprocesamiento y el contenedor del modelo), guardar las características ya calculadas en Feature Store para leerlas desde ambos lados, o empaquetar el código de transformación dentro del propio contenedor del modelo. Si la transformación solo existe en un notebook, no es replicable.

#### Ingeniería de características para datos numéricos (_Feature Engineering for Numerical Data_)

Los datos numéricos son, en última instancia, el tipo de datos con el que quieres alimentar tu modelo de ML. Tanto si tus datos son categóricos como textuales o de imagen, el resultado de tu ingeniería de características tendrá que tener la forma de un conjunto distinto y bien organizado de números que puedan usarse para entrenar tu algoritmo de ML.

La forma en que conviertes este conjunto de números en características significativas puede tener un impacto considerable en el rendimiento y la precisión de tu modelo.

En las próximas secciones profundizaremos en los conceptos de ingeniería de características presentados antes.

##### Normalización (_Normalization_)

La normalización es un enfoque de ingeniería de características que lleva los datos numéricos de todas tus características a una escala consistente, normalmente entre 0 y 1. Esto tiene varios beneficios clave:

- **Ponderación equitativa.** La normalización asegura que ninguna característica domine el modelo por su escala. Esto es particularmente importante para los algoritmos de ML sensibles a la escala de tus datos, como los k vecinos más cercanos (k-NN) y las redes neuronales, que calculan distancias entre puntos de datos.
- **Convergencia más rápida.** En la optimización por descenso de gradiente, los datos normalizados ayudan al algoritmo de ML a converger más rápido, al asegurar que todas las características contribuyan por igual al gradiente. Esto produce un entrenamiento más eficiente y un mejor rendimiento de tu modelo de ML.
- **Interpretabilidad mejorada.** Los datos normalizados facilitan la interpretación de la importancia de las características y de los coeficientes del modelo, ya que todas las características están en una escala comparable.

Esta técnica es útil cuando quieres que tus datos estén en una escala común, pero no necesitan estar centrados en cero.

Como resultado de la normalización, los modelos de ML rinden mejor cuando todas las características de tu dataset están en la misma escala. La normalización puede producir mayor precisión y estabilidad, especialmente en los algoritmos basados en distancias.

La normalización se implementa típicamente con la función de escalado MinMax, que se cubrirá en detalle en la próxima sección, «Escalado».

La «convergencia más rápida» tiene una lectura directa en la factura: menos iteraciones significan menos tiempo de un trabajo de entrenamiento de SageMaker, que se cobra por segundo de instancia mientras corre.

##### Estandarización (_Standardization_)

La estandarización transforma las características para que tengan una media de 0 y una desviación estándar de 1. Se usa cuando quieres asegurar que los datos de tu característica estén centrados alrededor de la media ($\mu$) y escalados según la desviación estándar ($\sigma$).

Esta técnica es ideal para los algoritmos de ML que suponen datos con distribución normal, como la regresión lineal, la regresión logística o los algoritmos que usan descenso de gradiente.

La estandarización se implementa con la función de puntuación Z.

- **Puntuación Z (_Z-score_).** Esta solución transforma el dataset de la característica para que tenga una media de 0 y una desviación estándar de 1 con la siguiente fórmula _(reconstruida en su forma estándar; el `.txt` la perdió)_:

$$z = \frac{x - \mu}{\sigma}$$

donde

- $x$ es un punto de datos de tu característica.
- $\mu$ es la media del dataset de tu característica.
- $\sigma$ es la desviación estándar del dataset de tu característica.

La desviación estándar resultante es 1 por la naturaleza de la transformación.

Cuando restas la media ($\mu$) de cada punto ($x$), centras los datos alrededor de 0. Cada punto queda ajustado respecto del valor promedio del dataset.

Dividir entre la desviación estándar ($\sigma$) escala los puntos respecto de la dispersión de los datos. Este paso asegura que la varianza (que es el cuadrado de la desviación estándar) de los datos estandarizados sea 1.

##### Escalado (_Scaling_)

El escalado es el término general para ajustar el rango o la distribución de las características. Como la principal preocupación de la normalización es el rango del dataset transformado (escala de 0 a 1 para todas las características), mientras que la estandarización se centra en la distribución (0 como media $\mu$ y 1 como desviación estándar $\sigma$ del dataset transformado), estas dos técnicas que acabas de aprender forman parte del escalado.

Si los datos de tu característica no tienen una distribución normal, necesitas técnicas que manejen el sesgo o los valores atípicos de forma distinta a la estandarización por puntuación Z. Las que necesitas conocer para el examen son las siguientes:

- **Escalado robusto (_Robust scaling_).** El escalado robusto centra el dataset de la característica restando la mediana y lo escala según el IQR. Esto lo hace particularmente útil para datasets con valores atípicos, porque asegura que la tendencia central y la dispersión sean robustas frente a ellos. Como resultado, esta técnica de estandarización es menos sensible a los valores atípicos y funciona bien con datos sesgados.

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

Salida real (solo la tabla escalada; la original es el dataset de siempre):

```text
Scaled Data:
   Feature1  Feature2
0 -0.644172  0.511628
1 -0.668712  0.697674
2 -0.582822  0.790698
3 -0.153374 -0.046512
4 -0.030675  0.046512
5  0.030675 -0.139535
6  0.460123 -0.604651
7  0.582822 -0.660465
8  0.546012 -0.418605
9  1.932515  2.744186
```

Como puedes ver en la Figura 3.12, con el escalado robusto el data frame escalado es menos sensible a los valores atípicos que el data frame original. Como resultado de la transformación, los valores atípicos como [25, 23] no sesgan el nuevo dataset de tu característica.

- **Escalado MinMax (_MinMax scaling_).** Esta técnica de estandarización transforma las características a un rango fijo, normalmente entre 0 y 1. Esta técnica preserva las relaciones entre las características, pero no aborda el sesgo.

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

Salida real (tabla escalada):

```text
Scaled Data:
    Feature1  Feature2
0       0.4  0.500000
1       0.6  0.666667
2       0.8  0.833333
3       1.0  1.000000
4       0.2  0.166667
5       0.0  0.000000
```

- **Escalado MaxAbs (_MaxAbs scaling_).** El escalado MaxAbs es una técnica que se usa para estandarizar datos escalando cada punto de la característica respecto de su valor absoluto máximo. Como resultado, cada punto queda dentro del rango [−1, 1].

Esta técnica es la más adecuada para estandarizar datasets dispersos (_sparse_) o datasets con muchos puntos en 0, porque preserva las entradas en 0 manteniendo la distribución original de los datos.

Aquí hay una consecuencia de infraestructura que explica el «más adecuada». Un dataset disperso se guarda en memoria y en disco en un formato que solo almacena los valores distintos de cero y su posición. Una transformación que convierte los ceros en otro número (como restar la media) obliga a guardar todos los valores y puede multiplicar el tamaño en memoria hasta no caber en la instancia. MaxAbs no toca los ceros, así que el dataset sigue siendo disperso.

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

Salida real (tabla escalada):

```text
Scaled Data:
    Feature1  Feature2
0  0.142857      0.25
1  0.428571      0.50
2  0.714286      0.75
3  1.000000      1.00
4 -0.142857     -0.25
5 -0.428571     -0.50
```

- **Transformaciones de potencia (Box-Cox, Yeo-Johnson).** La varianza de un dataset es el cuadrado de la desviación estándar ($\sigma^2$). Por definición, la varianza siempre es un número positivo y mide la dispersión de los puntos de un dataset, indicando qué tan lejos está cada punto de la media ($\mu$). Matemáticamente, es el promedio de las diferencias al cuadrado respecto de la media. Una varianza alta significa que los puntos están dispersos lejos de la media; una varianza baja, que están cerca de ella. _(Símbolos reconstruidos; el `.txt` los perdió.)_

Las transformaciones de potencia ayudan a estabilizar la varianza y a hacer los datos más parecidos a una distribución normal. Dos tipos comunes son la transformación de Box-Cox y la de Yeo-Johnson. La primera funciona solo con datos positivos, mientras que la segunda maneja tanto datos positivos como negativos.

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

Salida real (tabla transformada):

```text
Scaled Data:
    Feature1  Feature2
0 -0.231746 -0.098131
1  0.335720  0.382620
2  0.878582  0.868027
3  1.406058  1.356739
4 -0.853764 -1.029260
5 -1.534851 -1.479995
```

Las transformaciones de potencia hacen que tus datos sean más adecuados para el modelado, al reducir el sesgo y estabilizar la varianza, lo que produce modelos más robustos y precisos.

##### Transformación logarítmica (_Logarithmic Transformation_)

Como aprendiste antes, la transformación logarítmica es un método para detectar y gestionar valores atípicos. Ayuda como técnica de ingeniería de características para datos numéricos al reducir el sesgo, limitar el impacto de los valores atípicos y reforzar las relaciones lineales. Así es como lo hace:

- **Reducir el sesgo.** Muchos datasets del mundo real están sesgados a la derecha, lo que significa que la mayoría de los puntos se concentran en valores bajos y unos pocos atípicos extienden la «cola» hacia valores más altos. Ejemplos comunes son la distribución del ingreso, donde un número pequeño de personas con ingresos altos crea una cola derecha larga, o las calificaciones de un examen difícil, donde la mayoría de los estudiantes obtiene puntajes bajos y unos pocos logran los más altos. Aplicar una transformación logarítmica comprime esta cola, lo que hace la distribución más simétrica y más cercana a una distribución normal, que muchos algoritmos de ML manejan mejor.
- **Limitar el impacto de los valores atípicos.** Los valores atípicos altos pueden dominar y distorsionar la escala de los datos numéricos. La transformación logarítmica reduce la magnitud de estos atípicos y los acerca al grueso de los datos sin eliminarlos por completo.
- **Reforzar las relaciones lineales.** En algunos datos, la relación entre las características y la variable objetivo puede ser multiplicativa o exponencial en lugar de lineal. La transformación logarítmica puede linealizar estas relaciones, lo que facilita que los modelos lineales capturen los patrones subyacentes.

Por ejemplo, considera el siguiente dataset:

```text
[x,y] = [[3, 1], [4, 10], [5, 100], [6, 1000]]
```

Al transformar $y$ en $\log_{10}(y)$, el nuevo dataset queda así _(expresión reconstruida; el `.txt` la perdió)_:

```text
[x,y] = [[3, 0], [4, 1], [5, 2], [6, 3]]
```

Al convertir los datos a una escala logarítmica, las variaciones se vuelven más manejables y la estructura general de los datos es más clara. Esta transformación puede mejorar significativamente el rendimiento y la precisión de muchos modelos.

> [!note] Recuadro del libro: ceros y negativos
> La transformación logarítmica no puede aplicarse directamente a características con valores 0 o negativos, porque el logaritmo de 0 y de los números negativos no está definido, sea cual sea su base. En esos casos, considera desplazar tus datos para asegurar que todos los valores sean positivos, o usa transformaciones alternativas (p. ej., la raíz cúbica), que podrían ser más adecuadas para tus datos.

##### Raíz cuadrada o cúbica (_Square or Cube Root_)

Para manejar una varianza alta en datasets numéricos (además de la transformación de potencia vista antes), considera usar transformaciones de raíz cuadrada o de raíz cúbica. La raíz cuadrada y la raíz cúbica de una característica tienen un efecto sobre su distribución. Sin embargo, ese impacto no es tan significativo como el de la transformación logarítmica. La raíz cúbica tiene una ventaja propia: puede aplicarse a valores negativos, incluido el 0. La raíz cuadrada solo puede aplicarse a valores positivos y al 0.

##### Discretización (_Binning_)

La discretización (_binning_, también llamada _bucketing_) es el proceso de dividir datos numéricos continuos en intervalos discretos o «contenedores» (_bins_). Esta técnica puede simplificar el desarrollo de modelos de ML y hacerlos más robustos frente a los valores atípicos y el ruido. Con el binning, transformas datos numéricos en datos categóricos.

Por ejemplo, si trabajas con datos de automóviles, puedes convertir el peso del vehículo en cinco columnas: `is_minicompact`, `is_subcompact`, `is_compact`, `is_midsize` e `is_large`.

#### Ingeniería de características para datos categóricos (_Feature Engineering for Categorical Data_)

Los datos categóricos pueden capturar información importante sobre las relaciones y características de tus datos que los datos numéricos podrían pasar por alto. Unas características categóricas bien construidas pueden mejorar significativamente el rendimiento de tus modelos de ML.

La ingeniería de características para datos categóricos empieza con la codificación (_encoding_), que es el proceso de transformar los datos de tu característica de formato de cadena a formato numérico. El formato numérico puede ser un entero, un arreglo de enteros, una matriz o incluso un tensor de enteros. La codificación se realiza para que los algoritmos de ML puedan interpretar y usar estos datos como entrada, ya que la mayoría de los algoritmos solo entienden valores numéricos.

Por ejemplo, si tus categorías son White, Black y Red, podrías codificar estos datos en tres vectores: [1, 0, 0] para representar White, [0, 1, 0] para representar Black y [0, 0, 1] para representar Red.

##### Codificación por etiquetas (_Label Encoding_)

La codificación por etiquetas (_label encoding_), como su nombre indica, asigna a cada categoría de tu característica categórica un entero único. Por ejemplo, la característica categórica «Color», con valores como estos

```text
["White", "Black", "Red"]
```

se convierte en esto:

```text
[0, 1, 2]
```

Aunque es sencilla, esta técnica puede ser problemática para los algoritmos que suponen relaciones ordenadas en los datos, porque impone una estructura ordinal a los valores categóricos.

Esta técnica de codificación es la más adecuada para los algoritmos de ML basados en árboles, porque estos algoritmos pueden manejar la relación ordinal implícita en la codificación por etiquetas.

##### Codificación one-hot (_One-Hot Encoding_)

La codificación one-hot convierte una característica categórica en un conjunto de características binarias, donde cada característica binaria representa una categoría posible, con valor 1 si la categoría está presente y 0 si no lo está.

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

La Figura 3.13 muestra el resultado que produce este programa.

_Figura 3.13 Codificación one-hot._

Salida real de la tabla codificada, con las ocho columnas visibles (en una terminal de 80 caracteres pandas oculta las del medio con `...`):

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

Como puedes ver, el dataset original tenía una característica categórica, «Color», con 10 puntos de datos (categorías): White, Black, Red, Blue, Green, Yellow, Pink, Brown, White y Black.

El dataset codificado con one-hot se transformó en ocho características numéricas, igual al número de colores distintos: `Color_Black`, `Color_Blue`, `Color_Brown`, `Color_Green`, `Color_Pink`, `Color_Red`, `Color_White` y `Color_Yellow`. Cada una tiene 10 puntos de datos.

La codificación one-hot es una buena opción para características con un conjunto pequeño de categorías y para algoritmos de ML no basados en árboles, como la regresión lineal, k-NN y las redes neuronales. Estos algoritmos se cubrirán en el próximo capítulo.

> [!note] Recuadro del libro: dimensionalidad y cardinalidad
> La codificación one-hot aumenta la dimensionalidad del dataset de tus características. Esto puede convertirse en un problema, sobre todo con características de alta cardinalidad (las que tienen muchas categorías únicas). Sin embargo, el impacto depende en gran medida del algoritmo de ML que uses y de la naturaleza de tus datos.

Para dar escala al «problema» en términos de infraestructura: una columna de códigos postales de EE. UU. tiene del orden de 40 000 valores distintos. Codificada con one-hot sobre 10 millones de filas y guardada como una tabla densa de números de 8 bytes, ocuparía unos 3.2 billones de bytes, es decir, alrededor de 3 TB, frente a unos 50 MB de la columna original. Nada de eso cabe en la memoria de una instancia; por eso las herramientas guardan estas columnas en formato disperso o se recurre a las técnicas siguientes.

Puedes mitigar el riesgo de una alta dimensionalidad usando técnicas como la codificación binaria, como se explica en la próxima sección. Otro enfoque es usar el algoritmo de análisis de componentes principales (PCA) después de la codificación one-hot. Consulta el capítulo 4 para más información.

##### Codificación binaria (_Binary Encoding_)

La codificación binaria es una técnica eficaz para manejar datos categóricos sin «hacer explotar» la dimensionalidad del dataset de tus características, como podría hacerlo la codificación one-hot. Funciona convirtiendo cada valor de categoría en su número binario correspondiente y luego dividiendo el número binario en bits individuales como características (columnas) separadas.

Piensa en la codificación binaria como una forma de compactar el número de características que resulta de la técnica one-hot.

En el siguiente ejemplo, usamos la clase `BinaryEncoder` del módulo `category_encoders` para codificar en binario el mismo dataset del ejemplo anterior:

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

La Figura 3.14 muestra la salida de esta transformación.

_Figura 3.14 Codificación binaria._

Salida real (tabla codificada):

```text
Binary Encoded Data:
    Color_0  Color_1  Color_2  Color_3
0        0        0        0        1
1        0        0        1        0
2        0        0        1        1
3        0        1        0        0
4        0        1        0        1
5        0        1        1        0
6        0        1        1        1
7        1        0        0        0
8        0        0        0        1
9        0        0        1        0
```

Como resultado de codificar en binario nuestras 10 categorías no distintas, el nuevo dataset tiene solo cuatro características numéricas (en lugar de ocho).

¿Por qué cuatro? Porque el dataset original tenía ocho categorías distintas, que pueden representarse en formato binario con cuatro bits. Observa que el número de puntos de datos (10) no cambia al transformar los datos.

> [!warning] Nota de precisión: por qué salen cuatro columnas
> Ocho categorías caben en tres bits (0 a 7). Salen cuatro porque `BinaryEncoder` numera primero las categorías a partir de 1, y la octava (Brown = 8 = `1000`) necesita un cuarto bit, como se ve en la fila 7 de la salida.

La codificación binaria es la más adecuada para resolver las limitaciones de la codificación one-hot, porque mitiga la «explosión» de dimensionalidad que resulta de crear una característica nueva por cada categoría única.

Sin embargo, comparada con la codificación one-hot, la codificación binaria puede producir una pérdida de información dentro de los datos categóricos, lo que podría afectar negativamente el rendimiento de tu modelo. Esto se debe a que la codificación binaria comprime los datos categóricos en menos bits, lo que hace que las distinciones entre categorías se difuminen.

##### Hashing de características (_Feature Hashing_)

Esta técnica usa una función hash para convertir datos categóricos de alta cardinalidad en un número fijo de características numéricas que tú, como ingeniero de ML, eliges.

El hashing de características es eficiente y altamente escalable, porque el resultado de la transformación es un vector de tamaño fijo, lo que mantiene bajo control el uso de memoria. Así funciona a grandes rasgos:

- **Característica de entrada.** La función hash recibe un valor de la característica, que podría ser una palabra, una categoría o cualquier otro atributo identificable de tus datos.
- **Aplicación de la función hash.** La característica se procesa con la función hash, que devuelve un valor entero único (valor hash) basado en el valor de la característica.
- **Operación módulo (opcional).** Para asegurar que el valor hash caiga dentro del rango deseado (normalmente el tamaño del vector de características), a menudo se toma el valor hash módulo un número predefinido.
- **Actualización del vector.** El valor hash calculado se usa como índice para actualizar el elemento correspondiente del vector de características.

Al aprovechar una función hash, este enfoque transforma los datos rápidamente, lo que lo hace adecuado para datasets grandes. Aunque pueden ocurrir colisiones, la agregación de valores suele mitigar su impacto, especialmente con _hash buckets_ lo bastante grandes.

El «uso de memoria bajo control» es la ventaja de infraestructura que el libro resume en una línea. Con one-hot, la transformación necesita guardar la lista completa de categorías vistas (el vocabulario) y el número de columnas crece cada vez que aparece una categoría nueva, lo que rompe una canalización ya desplegada. Con hashing no se guarda vocabulario y el número de columnas lo fijas tú de antemano: el mismo código funciona igual en el notebook, en un trabajo de Spark repartido entre muchas máquinas o en el endpoint, sin compartir ningún estado entre ellos.

Por lo tanto, el hashing de características logra un equilibrio entre preservar la información y mantener la eficiencia computacional. Es una solución excelente para hacer la ingeniería de características de datos categóricos de alta cardinalidad de forma eficiente y a la vez económica.

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

La Figura 3.15 muestra las características hash que genera este programa.

_Figura 3.15 Hashing de características._

Salida real (tabla con hash):

```text
Hashed Data:
   feature_0  feature_1  feature_2
0        0.0       -1.0        0.0
1        0.0        1.0        0.0
2       -1.0        0.0        0.0
3        0.0       -1.0        0.0
4        0.0        0.0       -1.0
5        0.0        0.0        1.0
6        0.0        0.0       -1.0
7        0.0        0.0       -1.0
8        0.0       -1.0        0.0
9        0.0        1.0        0.0
```

Observa cómo la clase `FeatureHasher` aplica el hashing de características para convertir los datos categóricos (10 categorías) en un vector de tamaño fijo para cada categoría. En este ejemplo, fijamos el número de características en el constructor de la clase `FeatureHasher`. Ese número se fijó en 3.

Como resultado, las 10 categorías se convirtieron mediante hash en 10 vectores. Cada vector está formado por tres enteros, uno por cada una de las tres características.

> [!warning] Nota de precisión: el valor hash no es único
> La lista del libro dice que la función hash devuelve un «valor entero único». No lo es, y la salida lo muestra: White y Blue producen el mismo vector (filas 0 y 3), y lo mismo Green, Pink y Brown (filas 4, 6 y 7). Esas son las colisiones que el propio libro menciona después. `FeatureHasher` asigna además un signo (+1 o −1) a cada valor, por eso aparecen los −1.

> [!tip] Dónde están estas técnicas en AWS sin escribir código (según la documentación vigente)
> El libro muestra las técnicas con scikit-learn. Para el examen conviene saber que también existen como transformaciones integradas en los dos servicios visuales del capítulo:
>
> | Técnica del libro                         | SageMaker Data Wrangler (en Canvas)                                   | AWS Glue DataBrew (paso de receta)                               |
> | ----------------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------- |
> | Estandarización, robusto, MinMax, MaxAbs  | _Process numeric_: Standard, Robust, Min Max y Max Absolute Scaler    | `SCALE` con estrategia `Z_SCORE`, `MIN_MAX`, `MEAN_NORMALIZATION` |
> | Logaritmo, raíces                         | Transformación personalizada (pandas, PySpark o fórmula)              | `SKEWNESS` con función `LOG`, `ROOT` o `SQUARE`                  |
> | Binning                                   | Transformación personalizada o fórmula                                | `BUCKETIZATION`                                                  |
> | Label encoding / ordinal                  | _Encode categorical_ → _Ordinal encode_                               | `CATEGORICAL_MAPPING`                                            |
> | One-hot                                   | _Encode categorical_ → _One-hot encode_                               | `ONE_HOT_ENCODING`                                               |
> | Alta cardinalidad                         | _Encode categorical_ → _Similarity encode_                            | —                                                                |
>
> Data Wrangler ejecuta estas transformaciones con Apache Spark, lo que le permite aplicar el mismo flujo a datasets que no caben en la memoria de una sola máquina.

> [!warning] Un comportamiento operativo de los codificadores de Data Wrangler
> Los codificadores categóricos de Data Wrangler fijan la lista de categorías **en el momento en que defines el paso**. Si el flujo se ejecuta después como trabajo sobre datos nuevos (por ejemplo, cada mes) y aparecen categorías que no existían, o que no estaban en la muestra con la que se definió el paso, se tratan como valores faltantes según la _estrategia de manejo de inválidos_ que elijas (omitir la fila, conservarla como categoría extra, dar error o reemplazar por NaN). Es la misma razón por la que el hashing es cómodo en canalizaciones que no se detienen.

#### Ingeniería de características para series de tiempo (_Feature Engineering for Time-Series Data_)

Los datos de series de tiempo se presentaron al comienzo de este capítulo y son distintos de los datos tabulares estándar porque se capturan repetidamente a lo largo del tiempo, y cada punto sucesivo depende de sus valores pasados. Piensa en una serie de tiempo como en una película: cada escena depende de las anteriores. La dimensión temporal desempeña un papel clave en los datos de series de tiempo.

Amazon SageMaker Data Wrangler ofrece una solución _low-code_ para el procesamiento de series de tiempo, con la capacidad de limpiar, transformar y preparar datos más rápido. También permite a los científicos de datos construir las características de las series de tiempo respetando los requisitos de formato de entrada de su modelo de pronóstico. En la documentación vigente, Data Wrangler agrupa estas herramientas en el grupo de transformaciones _Time series_: agrupar por serie, remuestrear a otra frecuencia, rellenar huecos temporales, validar las marcas de tiempo, igualar la longitud de las series, extraer características, crear rezagos, generar rangos de fechas y calcular ventanas móviles.

Para hacer ingeniería de características sobre un dataset de series de tiempo, primero necesitamos entender los patrones presentes en nuestro dataset. Amazon SageMaker Data Wrangler ofrece varias visualizaciones que dan a científicos y analistas de datos pistas valiosas sobre los patrones existentes y que pueden ayudarte a elegir una estrategia de modelado. Una vez que entendemos los patrones presentes en nuestro dataset, podemos empezar a construir características nuevas orientadas a aumentar la precisión de los modelos de pronóstico.

##### Descomponer la fecha y hora (_Featurize Datetime_)

Es buena práctica empezar el proceso de ingeniería de características para series de tiempo desambiguando las características de fecha y hora. Las características de fecha y hora se crean a partir de la columna de marca de tiempo y son una forma óptima para que los ingenieros de ML empiecen el proceso de ingeniería de características. La transformación de series de tiempo _Featurize datetime_ nos permite descomponer una característica de fecha y hora agregando a nuestro dataset las nuevas características `date_month`, `date_day`, `date_week_of_year`, `date_day_of_year` y `date_quarter`. Como proporcionamos los componentes de la fecha y hora como características separadas, permitimos que los algoritmos de ML detecten patrones que mejoran la precisión de las predicciones.

En la documentación vigente, esta transformación acepta fechas escritas como texto o como marca de tiempo Unix (el número de segundos, o de milisegundos, microsegundos o nanosegundos, transcurridos desde el 1 de enero de 1970). Permite que Data Wrangler infiera el formato o que tú lo indiques con los códigos de `strftime` de Python: indicarlo es lo más rápido; no indicarlo ni inferirlo es lo más robusto, pero puede ser un orden de magnitud más lento. La salida puede ser un solo vector o una columna por componente, y hay que elegir un modo de representación (_embedding mode_): AWS recomienda `cyclic` para modelos lineales y redes profundas, y `ordinal` para algoritmos basados en árboles.

##### Codificar como categórica (_Encode Categorical_)

Las características de fecha y hora no se limitan a valores enteros. También puedes decidir considerar ciertas características de fecha y hora extraídas como variables categóricas y representarlas como características codificadas con one-hot, con cada columna conteniendo valores binarios. Para ello, usa la transformación _One-hot encode_. La característica recién creada `date_quarter` contiene valores de 0 a 3 y puede codificarse con one-hot en cuatro columnas binarias, una por cada uno de los cuatro trimestres.

> [!warning] Nota de precisión: rango de `date_quarter`
> No pude confirmar en la documentación vigente de Data Wrangler si los trimestres se numeran de 0 a 3. pandas, por ejemplo, usa 1 a 4 (`Series.dt.quarter`). Para la codificación one-hot da igual: son cuatro columnas en ambos casos.

##### Característica de rezago (_Lag Feature_)

Para aumentar la precisión del modelo, es buena práctica crear características de rezago (_lag features_) para la variable objetivo (o etiqueta). Las características de rezago son valores en marcas de tiempo anteriores que ayudan a predecir valores futuros. También ayudan a identificar patrones de autocorrelación (también llamada correlación serial) en la serie de residuos, al cuantificar la relación de la observación con las observaciones de pasos de tiempo anteriores. La autocorrelación es similar a la correlación normal, pero entre los valores de una serie y sus valores pasados. Amazon SageMaker Data Wrangler ofrece la transformación _Lag features_, que ayuda a crear múltiples características de rezago sobre un tamaño de ventana especificado.

En la consola, la transformación pide la columna de la que se generan los rezagos, la columna de marca de tiempo y la duración del rezago, y ofrece tres opciones de salida: incluir toda la ventana de rezagos, aplanar la salida en columnas separadas y descartar las filas que no tienen suficiente historia previa.

##### Características de ventana móvil (_Rolling Window Features_)

Amazon SageMaker Data Wrangler implementa capacidades automáticas de extracción de características de series de tiempo usando el paquete de código abierto `tsfresh`. Con la transformación de extracción de características de series de tiempo, puedes automatizar el proceso de extracción. Esto elimina el tiempo y el esfuerzo que de otro modo dedicarías a implementar manualmente bibliotecas de procesamiento de señales. Para el examen, necesitas saber que las características pueden extraerse con la transformación _Rolling window features_, que calcula propiedades estadísticas sobre un conjunto de observaciones definido por el tamaño de la ventana.

La transformación _Extract features_ ofrece cuatro estrategias que equilibran cantidad de características y costo de cómputo: _Minimal subset_ (8 características, la más rápida), _Efficient subset_ (todas las que no son computacionalmente costosas), _All features_ y _Manual subset_ (eliges tú la lista). _Rolling window features_ usa esas mismas estrategias, pero calculadas sobre una ventana: con una ventana de 3, a la fila del instante _t_ se le agregan las características extraídas de _t_ − 3, _t_ − 2 y _t_ − 1. La elección importa en la factura, porque la opción _All features_ sobre millones de filas multiplica el tiempo de procesamiento del trabajo.

> [!warning] Nota de precisión: `tsfresh`
> La página vigente de transformaciones de Data Wrangler describe _Extract features_ y _Rolling window features_, pero no menciona `tsfresh` por nombre. La afirmación del libro puede provenir de material anterior de AWS; no pude confirmarla en la documentación actual.

Después de hacer la ingeniería de características del dataset de series de tiempo, estamos listos para usar el dataset transformado como entrada de un algoritmo de ML de pronóstico.

### Técnicas para datos no estructurados (_Techniques for Unstructured Data_)

Los datos no estructurados, como imágenes, texto, audio y video, no caben fácilmente en tablas y a menudo carecen de un formato predefinido. AWS ofrece un ecosistema completo de productos y servicios que hacen más manejable y eficiente la ingeniería de características para datos no estructurados.

En la práctica, en un data lake de AWS estos datos viven como objetos en S3 (un objeto por imagen, por documento o por archivo de audio), y la tabla que describe el dataset suele ser un archivo aparte con una fila por objeto: su ruta en S3 y sus metadatos. Esa separación explica por qué casi todos los servicios de esta sección reciben como entrada una **ruta de S3** y devuelven resultados en JSON, que después se convierten en columnas.

En las próximas secciones cubriremos las técnicas que necesitas conocer para el examen y que se aplican a datos de imagen y de texto.

#### Ingeniería de características para datos de imagen (_Feature Engineering for Image Data_)

Igual que con los datos estructurados, la ingeniería de características para datos de imagen es un paso crítico en el desarrollo de modelos de ML eficaces. Este proceso consiste en extraer de las imágenes características significativas, como bordes, texturas y colores, para mejorar el rendimiento de las tareas de reconocimiento y clasificación de imágenes, proporcionando al modelo información más relevante sobre los datos.

Por ejemplo, al comienzo del capítulo usamos la imagen de un auto (Figura 3.4) para mostrarte qué características podían extraerse de estos datos no estructurados.

Para extraer características de imágenes, lo principal es aprovechar los modelos de visión preentrenados de Amazon SageMaker JumpStart, a menudo procesándolas con una red neuronal convolucional (CNN) y recuperando los vectores de características resultantes. Estos modelos de visión preentrenados incluyen modelos de detección de objetos, de clasificación de imágenes y muchos otros.

**Amazon SageMaker JumpStart** es un catálogo, dentro de SageMaker, de modelos ya entrenados por AWS y por terceros que puedes desplegar o ajustar (_fine-tune_) con pocos clics o pocas líneas de código, sin entrenarlos desde cero. En términos de infraestructura, «usar un modelo de JumpStart para extraer características» significa una de dos cosas:

- **Desplegarlo en un endpoint en tiempo real.** SageMaker crea una o más instancias (con GPU, si el modelo lo requiere) que alojan el contenedor del modelo detrás de una URL HTTPS; tu código le envía imágenes y recibe los vectores. Las instancias se cobran por hora mientras el endpoint exista, aunque no reciba peticiones, así que conviene borrarlo al terminar.
- **Usarlo en un trabajo de transformación por lotes** (_batch transform_). SageMaker levanta instancias, lee todas las imágenes de un prefijo de S3, escribe los vectores resultantes en otro prefijo de S3 y apaga las instancias al terminar. Para procesar una sola vez un archivo histórico de imágenes, es la opción más barata.

En ambos casos, los permisos los da el rol de ejecución de SageMaker, que necesita leer el bucket de las imágenes y escribir en el de salida.

También puedes usar servicios como Amazon Rekognition para el análisis básico de imágenes y la extracción de características, según tu caso de uso. Amazon Rekognition puede detectar objetos, escenas, texto y rostros en imágenes, y proporciona características de alto nivel sin necesidad de un gran esfuerzo manual. Por ejemplo, Amazon Rekognition puede identificar y extraer características como etiquetas y cuadros delimitadores (_bounding boxes_) alrededor de los objetos, que son esenciales para tareas como la detección de objetos y la clasificación de imágenes.

**Amazon Rekognition** es un servicio de visión por computadora ya entrenado que se usa por API: le envías una imagen (en la propia petición o como referencia a un objeto de S3) y te devuelve un JSON con lo que detectó, sin que entrenes, despliegues ni administres ningún modelo; se paga por imagen analizada. Por ejemplo, la operación `DetectLabels` devuelve cada etiqueta con su nivel de confianza y, para los objetos que puede localizar, un `BoundingBox`: el rectángulo que encierra el objeto, descrito por cuatro números (`Left`, `Top`, `Width`, `Height`) expresados como fracciones del ancho y el alto de la imagen, de modo que no dependen de su resolución. Una respuesta del tipo «Car, confianza 98.7, con `Left` = 0.12, `Top` = 0.40, `Width` = 0.35, `Height` = 0.20» se convierte directamente en columnas de tu dataset. La diferencia con JumpStart es de control frente a comodidad: Rekognition no tiene infraestructura que administrar, pero solo devuelve las etiquetas y atributos que AWS definió; JumpStart te da el vector completo del modelo, a cambio de operar el endpoint o el trabajo.

Una vez extraídas las características, transformarlas a un formato adecuado es esencial. Amazon SageMaker Data Wrangler te permite hacer análisis exploratorio de datos (EDA) y aplicar transformaciones como la normalización y la reducción de dimensionalidad. Este paso asegura que tus características estén optimizadas para el entrenamiento del modelo. Por ejemplo, puedes normalizar los valores de los píxeles a una escala común o aplicar PCA para reducir la dimensionalidad de tu conjunto de características. En la documentación vigente, Data Wrangler importa las imágenes directamente desde una carpeta (prefijo) de S3 y ofrece transformaciones integradas para ellas, basadas en las bibliotecas de código abierto OpenCV e imgaug: cambiar el tamaño, ajustar el brillo, pasar a escala de grises, rotar y eliminar imágenes corruptas o duplicadas, entre otras. La reducción de dimensionalidad con PCA está en su grupo de transformaciones _Reduce dimensionality_.

Almacenar y gestionar de forma eficiente tus características construidas es crucial para tener flujos de trabajo ágiles. Como aprendiste al comienzo del capítulo, Amazon SageMaker Feature Store proporciona un repositorio centralizado para almacenar características, lo que facilita compartirlas y reutilizarlas entre distintos modelos y proyectos. Esto asegura la consistencia y ahorra tiempo, porque no tienes que volver a extraer las características para cada modelo nuevo. El ahorro es literal: volver a pasar un archivo de millones de imágenes por un modelo con GPU cuesta horas de instancia cada vez, mientras que leer los vectores ya guardados cuesta una consulta.

#### Ingeniería de características para datos textuales (_Feature Engineering for Textual Data_)

En el mundo actual, impulsado por los datos, el texto está en todas partes, desde las publicaciones en redes sociales hasta las reseñas de clientes y más. Extraer información significativa de estos datos no estructurados requiere una ingeniería de características eficaz. AWS ofrece un ecosistema potente de productos y servicios para agilizar este proceso y convertir el texto crudo en características valiosas para los modelos de ML.

Amazon Comprehend es una herramienta potente para extraer características de alto nivel a partir de texto. Puede detectar entidades, frases clave, sentimiento e idioma, lo que proporciona información rica sin un gran esfuerzo manual. Además, Amazon Textract puede extraer texto de documentos, lo que convierte datos no estructurados en información estructurada.

- **Amazon Comprehend** es un servicio de NLP ya entrenado que se usa por API, igual que Rekognition, pero para texto. Cada capacidad es una operación distinta (`DetectEntities`, `DetectKeyPhrases`, `DetectSentiment`, `DetectDominantLanguage`) y se puede usar de dos maneras: **sincrónica**, enviando un documento en la petición y recibiendo la respuesta al instante, o **asincrónica**, lanzando un trabajo que lee miles de documentos de un prefijo de S3 y escribe los resultados en otro. Para convertir un archivo histórico de reseñas en columnas conviene la segunda. Se cobra por volumen de texto procesado (consulta la página de precios vigente).
- **Amazon Textract** es un servicio de **reconocimiento óptico de caracteres** (OCR, _optical character recognition_), la tecnología que convierte la imagen de un texto (un escaneo, una foto, un PDF sin texto seleccionable) en texto de computadora. Además del texto, Textract reconoce la estructura del documento: pares clave-valor de formularios («Nombre: Ana»), tablas y respuestas a preguntas en lenguaje natural (_queries_), cada elemento con su nivel de confianza y su posición en la página. Una imagen de una página se procesa de forma sincrónica; un PDF de varias páginas guardado en S3 se procesa de forma asincrónica, con un trabajo que avisa al terminar. Se cobra por página.

El orden habitual de uso es Textract primero y Comprehend después: Textract convierte el documento escaneado en texto y Comprehend extrae de ese texto entidades y sentimiento.

Para una extracción de características más personalizada, Amazon SageMaker ofrece algoritmos y frameworks integrados. Estos algoritmos pueden usarse para entrenar modelos y extraer características específicas, como _word embeddings_ (p. ej., Word2Vec), o para hacer modelado de temas (p. ej., _Latent Dirichlet Allocation_). Estos algoritmos se verán en detalle en el capítulo 4. En SageMaker, Word2Vec se ofrece dentro del algoritmo integrado **BlazingText**, y LDA es un algoritmo integrado con ese mismo nombre. «Algoritmo integrado» significa que AWS ya empaquetó el algoritmo en un contenedor: tú solo creas un **trabajo de entrenamiento** que indica el algoritmo, la ruta de los datos en S3, el tipo y número de instancias y el rol de IAM, y el trabajo deja el modelo resultante como un archivo en S3.

Las siguientes son algunas técnicas esenciales que necesitas conocer para el examen.

##### Tokenización (_Tokenization_)

La tokenización es el proceso de dividir el texto en unidades más pequeñas, como palabras o frases, llamadas **tokens**. Es el primer paso para preparar datos de texto para el modelado.

Por ejemplo, la oración «Machine learning is fascinating» se tokeniza como `["Machine", "learning", "is", "fascinating"]`.

La tokenización es un componente clave que usan los modelos fundacionales (FM) de Amazon Bedrock para entender y procesar los _prompts_ de los usuarios. Al descomponer el texto de entrada en tokens más pequeños, estos modelos pueden interpretar eficazmente el significado y el contexto de la petición del usuario. Por ejemplo, un prompt como «Generate an image of a sunset» se dividiría en palabras y símbolos individuales, lo que permite al modelo analizar cada elemento y entender la instrucción en su conjunto. Este enfoque basado en tokens asegura que el modelo pueda manejar instrucciones complejas y con matices con alta precisión y relevancia.

**Amazon Bedrock** es el servicio fully managed de AWS que da acceso por API a **modelos fundacionales** (_foundation models_, FM) de varios proveedores: modelos muy grandes, ya entrenados por AWS o por terceros, que se usan tal cual para generar texto o imágenes. No despliegas ni administras servidores: tu código envía una petición firmada con credenciales de IAM a un endpoint regional de Bedrock y recibe la respuesta. El **prompt** es el texto de entrada de esa petición, es decir, la instrucción que le das al modelo.

Además, la tokenización tiene un papel clave en el modelo de precios de uso de los FM de Amazon Bedrock. El costo de utilizar estos modelos se determina por el número de tokens de entrada procesados y de tokens de salida generados. Este sistema de cobro por tokens asegura que a los usuarios se les cobre de forma justa según los recursos computacionales realmente consumidos. Por ejemplo, un prompt más largo y detallado requeriría más tokens y, por lo tanto, tendría un costo mayor que un prompt más corto y simple. En el capítulo 4 presentaremos un ejemplo práctico que muestra cómo generar una imagen a partir de un prompt sencillo usando un FM disponible en Amazon Bedrock, lo que ilustra la eficiencia y la eficacia del procesamiento basado en tokens.

Tres detalles prácticos del cobro por tokens que el libro no menciona:

- **Cada modelo cuenta los tokens a su manera**, porque cada uno usa su propio tokenizador, así que el mismo prompt puede costar distinto en dos modelos aunque la tarifa por token fuera igual. Bedrock ofrece la operación `CountTokens`, sin costo, que devuelve cuántos tokens consumiría una petición en un modelo concreto antes de enviarla; además, cada respuesta incluye cuántos tokens de entrada y de salida se facturaron.
- **La salida suele costar más por token que la entrada**, así que limitar la longitud de la respuesta (con el parámetro de máximo de tokens) es la forma más directa de acotar el costo.
- Para dar escala, una regla práctica muy difundida es que un token equivale, en promedio, a unos tres cuartos de palabra en inglés; en español suele hacer falta algo más de un token por palabra. Tómalo solo como orden de magnitud.

> [!warning] Nota de precisión: modalidades de cobro de Bedrock (verifica en la página de precios vigente)
>
> - El cobro por tokens de entrada y salida aplica a los modelos **de texto** en modalidad bajo demanda. Los modelos que **generan imágenes**, como el del ejemplo del capítulo 4, se cobran por **imagen generada** (según resolución y calidad), no por tokens.
> - Además de la modalidad bajo demanda existen otras, como la inferencia por lotes y el _throughput_ aprovisionado (capacidad reservada que se cobra por hora).
> - El modelo del ejemplo del capítulo 4, Nova Canvas, no admite clientes nuevos desde el 30-03-2026 y deja de funcionar el 30-09-2026 (ver las notas del capítulo 4).

##### Eliminación de palabras vacías (_Stop-Words Removal_)

Las palabras vacías (_stop words_) son palabras comunes (p. ej., «the», «is», «and») que a menudo no aportan información significativa. Eliminarlas puede reducir el ruido y mejorar el rendimiento del modelo.

Por ejemplo, eliminar «is» y «the» de «The cat is on the mat» daría como resultado `["cat", "on", "mat"]`.

##### Stemming y lematización (_Stemming and Lemmatization_)

Estas técnicas reducen las palabras a su forma base o raíz. El _stemming_ recorta prefijos y sufijos, mientras que la lematización usa reglas lingüísticas.

Por ejemplo, «running» se convierte en «run».

> [!tip] Tokenización, stop words y stemming sin código en AWS
> El paso de receta `TOKENIZATION` de AWS Glue DataBrew hace estas tres cosas en una sola transformación: divide el texto en tokens, elimina palabras vacías (con la lista predeterminada o una lista propia) y aplica stemming con los algoritmos `PORTER` o `LANCASTER`; también puede expandir contracciones («don't» → «do not»). En Data Wrangler, la transformación _Featurize text_ → _Vectorize_ incluye un tokenizador configurable. Ninguno de los dos ofrece lematización como transformación integrada según la documentación consultada; para eso hace falta código propio (por ejemplo, en una transformación personalizada de Data Wrangler).

##### N-gramas (_N-grams_)

Los n-gramas son secuencias contiguas de _n_ elementos de un texto dado. Capturan el contexto y las relaciones entre palabras.

Por ejemplo, con un bigrama (es decir, una secuencia contigua de dos palabras), la oración «Machine learning is fun» se transforma en `[("Machine", "learning"), ("learning", "is"), ("is", "fun")]`.

##### Word embeddings

Los _word embeddings_, como Word2Vec y GloVe, convierten las palabras en vectores continuos que capturan su significado semántico y preservan las relaciones entre palabras. Esto hace que los datos sean más adecuados para los modelos de ML.

Por ejemplo, «King» y «Queen» podrían quedar cerca en el espacio vectorial, lo que refleja su similitud semántica.

## Etiquetado de datos (_Data Labeling_)

En el ámbito del ML, el etiquetado de datos y la ingeniería de características son dos procesos entrelazados y fundamentales para construir modelos eficaces. Ambos transforman los datos crudos en una forma estructurada que los algoritmos de ML pueden entender y de la que pueden aprender.

La ingeniería de características se centra en seleccionar, modificar y crear características nuevas a partir de los datos crudos para mejorar el rendimiento del modelo, mientras que el etiquetado de datos consiste en enriquecer los datos crudos con metadatos en forma de etiquetas significativas que denotan las predicciones reales o atributos específicos. Para el aprendizaje supervisado, este paso es fundamental, porque proporciona la verdad de referencia (_ground truth_) de la que aprenden los modelos. Al etiquetar los datos, le das al modelo las «respuestas correctas» de las que aprender y contra las que predecir.

Las etiquetas actúan como punto de referencia y guían el proceso de aprendizaje de los algoritmos. Definen lo que el modelo debe predecir o clasificar. Por ejemplo, en un dataset de imágenes de animales, etiquetar cada imagen como «cat» o «dog» permite al modelo aprender a distinguir entre estos animales.

### Amazon SageMaker Ground Truth

> [!warning] Estado del servicio (verificado el 25-09-2026)
> Según la documentación de AWS, **Amazon SageMaker Ground Truth ya no admite clientes nuevos**. Los clientes existentes pueden seguir usándolo con normalidad; AWS sigue invirtiendo en su seguridad y disponibilidad, pero no planea añadirle funciones. El contenido del libro sigue siendo útil para el examen y para entender cómo se organiza un proceso de etiquetado en AWS.

Con AWS, el etiquetado de datos se hace con Amazon SageMaker Ground Truth, un servicio diseñado específicamente para agilizar y mejorar el proceso de etiquetado de datos. Este servicio combina automatización avanzada con capacidades de intervención humana (**_human-in-the-loop_**) para entregar datasets etiquetados de alta calidad de forma eficiente. _Human-in-the-loop_ significa que el proceso automático tiene puntos en los que las personas revisan, corrigen o deciden, sobre todo en los casos en que la máquina tiene poca confianza. Este es el proceso a grandes rasgos:

- **Almacenar los datos.** Sube tus datos crudos (imágenes, texto, etc.) a Amazon S3. Crea carpetas o buckets separados para organizar tus datos según el tipo o el proyecto. En S3 las «carpetas» son en realidad **prefijos** de la clave del objeto, como `proyecto-a/imagenes/`, que la consola muestra como si fueran carpetas. Separar por bucket tiene sentido cuando cambian los permisos o el cifrado (por ejemplo, datos de un cliente que exige su propia clave de KMS); separar por prefijo basta para organizar. Ground Truth necesita además un **archivo de manifiesto de entrada**: un archivo JSON Lines (un objeto JSON por línea) en S3 que enumera los objetos que se van a etiquetar.
- **Crear un trabajo de etiquetado.** En Amazon SageMaker Ground Truth, define la tarea y las instrucciones. Un **trabajo de etiquetado** (_labeling job_) es el recurso de AWS que reúne todo lo necesario: la ubicación de los datos en S3, el tipo de tarea, la plantilla de la interfaz que verán los anotadores, la fuerza de trabajo, el precio por tarea si aplica y el rol de IAM con el que Ground Truth lee y escribe en tus buckets.
- **Etiquetado automatizado.** Elige el tipo de datos que vas a etiquetar (p. ej., clasificación de imágenes, clasificación de texto, detección de objetos) y especifica el dataset de entrada almacenado en S3. Selecciona _Automated Data Labeling_ para usar modelos de ML preentrenados que etiqueten un subconjunto de tus datos, y configura los ajustes y parámetros del algoritmo de etiquetado.
- **Revisión humana.** Los anotadores revisan y corrigen las etiquetas para asegurar su precisión. Este paso puede realizarlo Amazon Mechanical Turk, una fuerza de trabajo privada o proveedores externos. Amazon Mechanical Turk (MTurk) es un mercado de _crowdsourcing_ que conecta a las empresas con una fuerza de trabajo global de personas que completan tareas difíciles para las computadoras, y el etiquetado de datos es una de ellas. _Crowdsourcing_ significa repartir una tarea entre una multitud de personas externas, cada una de las cuales hace una parte pequeña a cambio de un pago por tarea. Las otras dos opciones son una **fuerza de trabajo privada** (tus propios empleados o contratistas, que entran a un portal de etiquetado con usuarios que tú administras; es la opción cuando los datos no pueden salir de la organización) y los **proveedores externos**, empresas especializadas en etiquetado que se contratan a través de **AWS Marketplace**, la tienda de software y servicios de terceros integrada en AWS.
- **Etiquetado y validación de datos.** Amazon SageMaker Ground Truth ofrece una interfaz fácil de usar para que los etiquetadores anoten los datos. Los anotadores revisan y corrigen las etiquetas automáticas, y agregan etiquetas, cuadros delimitadores u otras anotaciones relevantes según sea necesario. Incluso puedes implementar medidas de control de calidad, como el etiquetado por consenso (varios anotadores etiquetan el mismo dato) y el muestreo de auditoría (expertos revisan un subconjunto de las etiquetas).
- **Almacenar los datos etiquetados.** Puedes conservar los datos etiquetados en tu bucket de S3, o guardarlos automáticamente en Amazon SageMaker Feature Store para tenerlos a mano durante el entrenamiento y la inferencia del modelo. En cualquier caso, asegúrate de mantener versionados los datos etiquetados para rastrear los cambios. En S3, la forma más directa de lograrlo es activar el **versionado del bucket** (_S3 Versioning_): cada vez que se sobrescribe o se borra un objeto, S3 conserva la versión anterior, y se puede recuperar cualquier versión pasada. Cada versión ocupa espacio y se cobra, por lo que suele combinarse con una regla de ciclo de vida que borre las versiones antiguas pasado cierto tiempo.
- **Entrenar el modelo.** Usa los datos etiquetados para entrenar tus modelos en Amazon SageMaker.

> [!warning] Nota de precisión: cómo funciona el etiquetado automatizado (según la documentación vigente)
>
> - **El orden es el inverso al de la lista.** Ground Truth no empieza con modelos preentrenados que etiquetan primero. Primero envía a **personas** una muestra aleatoria; con esas etiquetas lanza un **trabajo de entrenamiento** de SageMaker, luego un **trabajo de transformación por lotes** para puntuar el resto de los datos, etiqueta automáticamente solo los objetos cuya confianza supera un umbral que calcula él mismo, y devuelve a las personas los de baja confianza. El ciclo se repite hasta etiquetar todo o agotar el presupuesto de etiquetado humano. AWS llama a esta técnica _active learning_.
> - **Tiene costo de cómputo aparte.** Esos trabajos corren en instancias que Ground Truth administra (no aparecen en tu consola de EC2) y se facturan como entrenamiento e inferencia de SageMaker. La documentación indica, por ejemplo, instancias `ml.p3.2xlarge` (con GPU) para entrenar en clasificación de imágenes y detección de objetos.
> - **Tiene límites.** Solo está disponible para cuatro tipos de tarea integrados (clasificación de imágenes y de texto con una etiqueta, detección de objetos con cuadros delimitadores y segmentación semántica), exige un mínimo de **1250 objetos** y AWS recomienda al menos **5000**. Para dar escala: con menos de unos pocos miles de imágenes, el etiquetado automatizado apenas llega a activarse, y el ahorro frente a etiquetar todo a mano es pequeño.
> - **La salida documentada es un archivo de manifiesto de salida** en S3 (JSON Lines, una línea por objeto con su ubicación y su etiqueta). No encontré documentada una opción para guardar los resultados «automáticamente» en Feature Store; tómalo como un paso adicional que harías tú, por ejemplo con un trabajo que lea el manifiesto y llame a `PutRecord`.
> - **Mechanical Turk y datos personales.** Para usar la fuerza de trabajo de Mechanical Turk, Ground Truth exige declarar que los datos no contienen información personal identificable (PII); los datos confidenciales van con una fuerza de trabajo privada o un proveedor. En cualquier caso, **Mechanical Turk cierra el 30 de septiembre de 2026**, así que en adelante solo quedarán esas dos opciones.

Con la combinación adecuada de automatización e intervención humana en el etiquetado, Amazon SageMaker Ground Truth ofrece una solución robusta para crear datasets etiquetados precisos y confiables, que son cruciales para construir modelos de ML de alto rendimiento.

## Gestión del desbalance de clases (_Managing Class Imbalance_)

Hasta aquí aprendiste a transformar tu dataset limpiando primero los datos crudos, luego haciendo la ingeniería de características de tus datos y, por último, etiquetando los datos. Este último paso es un aspecto fundamental del ML, porque se centra en proporcionar las predicciones correctas para cada punto de datos, de modo que tu modelo de ML pueda aprender rápidamente durante el entrenamiento.

Con tu problema de ML planteado y tus datos cuidadosamente procesados, pensarías que ya estás listo para seleccionar un algoritmo de ML y comenzar el proceso de entrenamiento, ¿verdad? ¡Todavía no!

Tu dataset puede verse bien sintácticamente como resultado de seleccionar, extraer y crear características, pero no semánticamente, por una distribución desigual de las etiquetas. Por ejemplo, en un caso de uso de salud, considera un dataset en el que la clase mayoritaria representa a personas sanas y la clase minoritaria a pacientes con una enfermedad rara. El objetivo de tu problema de ML es predecir la enfermedad con precisión. Cuando tu dataset presenta una clase de etiquetas desproporcionada, como en este escenario con una clase mayoritaria de personas sanas, estás ante un desbalance de clases (_class imbalance_).

El desbalance de clases produce un modelo sesgado hacia la clase mayoritaria. Esto puede afectar significativamente el rendimiento y la equidad del modelo, en particular en los casos en que la clase minoritaria es de importancia crítica, como en nuestro caso de uso de salud.

La buena noticia es que AWS ofrece un conjunto completo de productos y servicios para abordar el desbalance de clases, de modo que tus modelos no solo sean precisos, sino también éticos e imparciales.

Antes de profundizar en cómo aborda AWS este desafío, repasemos qué enfoque puedes seguir para tratar el desbalance de clases y los sesgos resultantes.

### Técnicas de mitigación del desbalance de clases (_Class-Imbalance Mitigation Techniques_)

El aumento de datos (_data augmentation_) se considera en general una buena práctica para generar más datos etiquetados de la clase minoritaria, y así cerrar la brecha entre las clases mayoritaria y minoritaria. Técnicas como el aumento de imágenes (rotaciones, volteos y ajustes de color) para datos de imagen, o el aumento de texto (reemplazo de sinónimos, inserción aleatoria) para datos de texto, pueden crear muestras diversas y representativas.

Sin embargo, el aumento de datos no siempre es viable por el costo, la disponibilidad de recursos de cómputo y las restricciones de tiempo. En esos escenarios, considera las siguientes opciones:

- **Sobremuestreo (_oversampling_).** Esta técnica consiste en aumentar el número de puntos de la clase minoritaria duplicando ejemplos existentes o generando ejemplos sintéticos. La técnica de sobremuestreo sintético de la minoría (SMOTE, _synthetic minority over-sampling technique_) es un método popular que crea muestras sintéticas interpolando entre ejemplos existentes de la clase minoritaria. Esto ayuda a balancear el dataset sin simplemente duplicar registros.
- **Submuestreo (_undersampling_).** Esta técnica funciona eliminando aleatoriamente puntos de la clase mayoritaria, lo que hace la distribución de clases más pareja con la clase minoritaria. Aunque puede ser eficaz, el submuestreo puede provocar la pérdida de información valiosa, lo que podría afectar el rendimiento del modelo.
- **Ponderación de clases (_class weighting_).** Al calcular la pérdida durante el entrenamiento, esta técnica asigna pesos más altos al algoritmo de ML por clasificar mal la clase minoritaria, lo que obliga al modelo a prestarle más atención.

> [!note] Recuadro del libro: aumento de datos frente a datos sintéticos
> El aumento de datos es distinto de los datos sintéticos. El aumento de datos crea artificialmente datos nuevos a partir de los datos de entrenamiento existentes, mientras que los datos sintéticos son datos generados que no usan el dataset original.

El «costo» y la «disponibilidad de recursos de cómputo» que menciona el libro son concretos en AWS. Aumentar un dataset de imágenes implica procesar y guardar varias copias transformadas de cada imagen (más almacenamiento en S3 y más horas de cómputo para generarlas), y cada copia alarga después cada época de entrenamiento en instancias con GPU. Las tres alternativas de la lista son, en comparación, casi gratuitas: operan sobre la tabla y no sobre los archivos.

¿Dónde se aplican estas técnicas en AWS? El libro no lo dice en esta sección. En la documentación vigente, SageMaker Data Wrangler tiene una transformación _Balance data_ con tres operadores, **sobremuestreo aleatorio**, **submuestreo aleatorio** y **SMOTE**, y un grupo de transformaciones de imagen (rotación, brillo, canales de color, entre otras) que sirve para el aumento de imágenes. La ponderación de clases, en cambio, no es una transformación de datos: se configura como hiperparámetro del algoritmo al lanzar el trabajo de entrenamiento (por ejemplo, `scale_pos_weight` en el XGBoost integrado de SageMaker).

### Amazon SageMaker Clarify

> [!warning] Estado del servicio (verificado el 25-09-2026)
> Según la documentación de AWS, **Amazon SageMaker Clarify ya no admite clientes nuevos**. Los clientes existentes pueden seguir usándolo, pero AWS no planea añadirle funciones. Como reemplazo, AWS indica calcular las mismas métricas de sesgo (CI, DPL y las demás), que son fórmulas publicadas, con pandas y scikit-learn dentro de tus propios trabajos y pipelines, registrando los resultados en MLflow administrado de SageMaker AI; usar directamente la biblioteca **SHAP**, que es la que Clarify usa por dentro, para la explicabilidad; y usar **Amazon Bedrock Evaluations** solo para evaluar modelos fundacionales. Las fórmulas siguen documentadas y el examen puede preguntarlas.

Amazon SageMaker Clarify ayuda a abordar el desbalance de clases proporcionando métricas de sesgo previas al entrenamiento (_pre-training bias metrics_) para detectar y mitigar el sesgo en tus datasets. Cada métrica corresponde a una noción distinta de equidad (_fairness_).

En términos de infraestructura, un análisis de Clarify es un **trabajo de procesamiento** (_processing job_) de SageMaker: una tarea que SageMaker ejecuta en instancias que levanta y apaga para la ocasión, con un contenedor especializado de Clarify. El contenedor lee de S3 el dataset y un archivo JSON de configuración del análisis, calcula las métricas y escribe los resultados en S3: un `analysis.json` con los valores y un informe visual. Para las métricas posteriores al entrenamiento también necesita predicciones del modelo; si no le das un endpoint existente, crea uno temporal (_shadow endpoint_) y lo borra al terminar. Las métricas previas al entrenamiento, las de esta sección, no necesitan modelo.

> [!warning] Nota de precisión: Clarify mide, tú mitigas
> Clarify **detecta y cuantifica** el sesgo; no modifica el dataset. La mitigación (remuestrear, ponderar, recolectar más datos) la decides y aplicas tú, por ejemplo con la transformación _Balance data_ de Data Wrangler.

Estas métricas son críticas para asegurar que tus modelos sean justos e imparciales desde el principio. La Tabla 3.1 muestra dos de las métricas más usadas: el desbalance de clases (CI, _Class Imbalance_) y la diferencia en proporciones de etiquetas (DPL, _Difference in Proportions of Labels_).

**Tabla 3.1** Ejemplos de métricas de sesgo previas al entrenamiento.

| Métrica de sesgo                                                                     | Interpretación                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Desbalance de clases (_Class Imbalance_, CI)                                         | Rango normalizado: [−1, +1]<br>0: sin desbalance de clases<br>+1: desbalance completo hacia la clase mayoritaria<br>−1: desbalance completo hacia la clase minoritaria                                                                                                                                                                           |
| Diferencia en proporciones de etiquetas (_Difference in Proportions of Labels_, DPL) | Rango para etiquetas de faceta normalizadas binarias y multicategoría: [−1, +1]<br>Rango para etiquetas de faceta continuas: [−∞, +∞]<br>0: igual proporción de resultados positivos entre facetas<br>+1: la faceta _a_ tiene la mayor proporción de resultados positivos<br>−1: la faceta _d_ tiene la mayor proporción de resultados positivos |
| Diferencia de igualdad de oportunidades (_Equal opportunity difference_, EOD)        | _(sin interpretación en el original)_                                                                                                                                                                                                                                                                                                            |
| Sesgo en detección de objetos (_Object detection bias_)                              | _(sin interpretación en el original)_                                                                                                                                                                                                                                                                                                            |
| Diferencia de paridad predictiva (_Predictive parity difference_, PPD)               | _(sin interpretación en el original)_                                                                                                                                                                                                                                                                                                            |
| Sesgo en clasificación de imágenes (_Image classification bias_)                     | _(sin interpretación en el original)_                                                                                                                                                                                                                                                                                                            |

> [!warning] Nota de precisión: la Tabla 3.1 (según la documentación de Clarify)
>
> - **Filas finales.** Las cuatro últimas filas aparecen en el `.txt` sin interpretación, y ninguna figura entre las métricas **previas al entrenamiento** de Clarify, que según la documentación son ocho: CI, DPL, KL, JS, LP, TVD, KS y CDD. EOD y PPD no pueden calcularse antes de entrenar porque requieren predicciones del modelo, y las filas de imágenes parecen restos de otra tabla. Se conservan por fidelidad al original.
> - **Interpretación de CI.** La documentación no habla de «clase mayoritaria o minoritaria», sino de **facetas**: +1 significa que el dataset solo contiene miembros de la faceta _a_, y −1 que solo contiene miembros de la faceta _d_.

En el contexto de los sesgos por desbalance de clases con Amazon SageMaker Clarify, una **faceta** (_facet_) denota una característica específica de tu dataset que quieres analizar en busca de posibles sesgos.

Las facetas se usan para identificar y medir cómo distintos subgrupos dentro de tus datos podrían verse afectados por el desbalance de clases. Con Amazon SageMaker Clarify, la faceta _a_ denota el valor de la característica que define al grupo demográfico que el sesgo favorece, y la faceta _d_ denota el valor de la característica que define al grupo demográfico que el sesgo desfavorece.

Por ejemplo, si analizas un dataset de transacciones con tarjeta de crédito, las facetas pueden incluir características como `is_fraudulent`, cuyo valor puede ser 1 (verdadero) o 0 (falso). Supongamos que el 99.9 % de los valores de esta característica son 0, lo que indica que el 99.9 % de las transacciones del dataset no son fraudulentas. Al examinar estos valores de faceta (1 o 0), Amazon SageMaker Clarify puede ayudarte a entender si ciertos subgrupos de alguno de estos valores de la característica están subrepresentados o sobrerrepresentados en tus datos de entrenamiento, lo que podría producir predicciones sesgadas del modelo.

Puedes encontrar otras métricas de sesgo previas al entrenamiento en https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-measure-data-bias.html.

Aunque el rango de cada métrica varía, todas tienen en común que un valor 0 (o cercano a 0) denota ausencia de desbalance de clases.

Al calcular estas métricas de sesgo previas al entrenamiento, puedes determinar las siguientes acciones sobre tu dataset como preparación para el entrenamiento. En nuestro ejemplo, suponiendo que la faceta _a_ es 0, probablemente obtendríamos un CI cercano a 1. Por lo tanto, necesitaríamos hacer submuestreo.

> [!warning] Nota de precisión: etiqueta y faceta son parámetros distintos en Clarify
> En la configuración de Clarify, la **etiqueta** (la columna que el modelo predice, aquí `is_fraudulent`) y la **faceta** (la columna del grupo que se examina) se declaran por separado, y la documentación usa como facetas atributos de grupo, como la edad o el género. Usar la propia etiqueta como faceta, como hace el ejemplo, reduce CI a medir el desbalance de la etiqueta, que se obtiene igual con un simple conteo. Además, «necesitaríamos hacer submuestreo» es una de las opciones de la sección anterior, no una consecuencia obligada.

La clase `BiasConfig` forma parte de la biblioteca `sagemaker.clarify`. Esta biblioteca te ayuda a detectar y mitigar el sesgo en modelos de ML. La clase `BiasConfig` se usa para configurar los ajustes del análisis de sesgo de tu dataset.

`sagemaker.clarify` es un módulo del **SDK de SageMaker para Python** (el paquete `sagemaker`), la biblioteca que permite crear trabajos y endpoints de SageMaker desde código. En `BiasConfig` se declara, en esencia, qué valor de la etiqueta cuenta como resultado positivo (`label_values_or_threshold`), qué columna es la faceta (`facet_name`) y qué valor o valores de esa columna definen el grupo examinado (`facet_values_or_threshold`). Después, esa configuración se pasa, junto con la ubicación de los datos en S3 (`DataConfig`), a un `SageMakerClarifyProcessor`, que lanza el trabajo de procesamiento con el tipo y número de instancias que indiques y con el rol de IAM que le permite leer y escribir en S3.

> [!warning] Nota de precisión: `sagemaker.clarify` en el SDK v3
> `sagemaker.clarify` es la ruta del SDK v2. En el SDK v3 instalado en esta máquina (3.22.1), `import sagemaker.clarify` falla; la clase está en `sagemaker.core.clarify.BiasConfig`, con los mismos tres parámetros (se comprobó la firma localmente, sin llamar a AWS). Recuerda también que Clarify no admite clientes nuevos.

## División de datos (_Data Splitting_)

Después de gestionar el desbalance de clases, tus datos por fin están listos para dividirse en los datasets de entrenamiento, validación y prueba.

La división de datos consiste en separar tu dataset en subconjuntos distintos para fines de entrenamiento, validación y prueba. Es un paso crítico en ML para asegurar que tu modelo se entrene eficazmente y se evalúe con precisión. Veamos qué debe formar parte de cada dataset y, lo más importante, cuál es su propósito:

- **Dataset de entrenamiento.** El dataset de entrenamiento se usa para enseñar al modelo. Es donde el modelo aprende los patrones y las relaciones dentro de los datos. Durante el entrenamiento, el modelo usa estos datos para ajustar sus parámetros y aprende iterativamente a minimizar los errores y mejorar las predicciones. Por ejemplo, podrías alimentar a un modelo con numerosos ejemplos etiquetados de correos electrónicos (spam y no spam) para enseñarle a clasificar los correos entrantes.
- **Dataset de validación.** El dataset de validación se usa para evaluar el rendimiento del modelo mientras se ajustan finamente sus hiperparámetros. Uno de los beneficios de usar un dataset de validación es evitar que el modelo se sobreajuste, asegurando que generalice bien a datos no vistos.
- **Dataset de prueba.** El dataset de prueba está (y debe estar) formado por datos que el modelo nunca ha visto. Después del entrenamiento y la validación, el rendimiento del modelo se mide sobre el dataset de prueba para obtener una evaluación imparcial de qué tan bien generaliza a datos nuevos. Por ejemplo, podrías crear un dataset de prueba con un conjunto separado de correos que el modelo no ha visto, para probar su precisión al clasificar spam y no spam.

En resumen, el dataset de entrenamiento se usa para construir el modelo, el de validación ayuda a ajustarlo y previene el sobreajuste, y el de prueba proporciona una evaluación imparcial del rendimiento del modelo.

> [!note] Recuadro del libro: la analogía del examen
> Piensa en los datasets de entrenamiento, validación y prueba como etapas de la preparación de un examen importante. El dataset de entrenamiento es como el material de estudio que usas para aprender a fondo la materia, lo que te permite entender e interiorizar la información. El dataset de validación se parece a los exámenes de práctica que haces para evaluar tu preparación, identificar áreas de mejora y hacer los ajustes necesarios sin que eso influya en tu proceso central de aprendizaje. Por último, el dataset de prueba es similar al examen real que presentas en un centro de pruebas con un supervisor, que evalúa objetivamente tu conocimiento y desempeño con preguntas que no has visto, lo que asegura que estés bien preparado para escenarios del mundo real. Esta metáfora resalta los papeles distintos, pero interconectados, que cada dataset cumple en el desarrollo de un modelo de ML exitoso y confiable.

Para dividir eficazmente los datos para ML, según las buenas prácticas, considera asignar aproximadamente entre el 60 y el 80 % al entrenamiento, para asegurar que el modelo aprenda y adquiera una comprensión completa de los datos. Usa entre el 10 y el 20 % para la validación, para ajustar finamente los parámetros del modelo sin influir en el proceso de entrenamiento y asegurar una evaluación imparcial del rendimiento. El 10–20 % final se reserva para la prueba y sirve como medida definitiva de la capacidad del modelo para generalizar a datos nuevos, no vistos. Una partición adecuada es un aspecto clave de la ingeniería de características, porque previene la fuga de datos, evita el sobreajuste y da una medida precisa de qué tan bien funcionará tu modelo de ML en escenarios del mundo real, lo que afecta su confiabilidad y eficacia generales.

En SageMaker, esta división tiene una forma física concreta: cada subconjunto se guarda como archivos separados en S3 (por ejemplo, bajo los prefijos `train/`, `validation/` y `test/`), y al lanzar un trabajo de entrenamiento se le pasan los dos primeros como **canales** (_channels_) de entrada con nombre, cada uno apuntando a su prefijo. El de prueba no se entrega al entrenamiento: se usa después, en un trabajo de evaluación o de transformación por lotes. Mantener los tres en prefijos separados (y, si hace falta, con permisos distintos) es la forma más simple de impedir que los datos de prueba lleguen al entrenamiento por error.

> [!note] Recuadro del libro: fuga de datos
> La fuga de datos (_data leakage_) ocurre cuando parte de los datos de prueba «se filtra» al dataset de entrenamiento. Es un error común que puede comprometer gravemente la integridad de los modelos de ML. Cuando ocurre, el modelo aprende información a la que no tendría acceso en un escenario del mundo real, lo que infla artificialmente las métricas de rendimiento y produce una mala generalización a datos nuevos, no vistos. Evitar la fuga de datos es crucial para asegurar que el rendimiento del modelo se evalúe con precisión y que pueda generalizar bien a situaciones del mundo real. Entre las técnicas eficaces para prevenirla están mantener una separación clara entre los datasets de entrenamiento y de prueba, usar validación cruzada y vigilar con cuidado los pasos de preprocesamiento. Como buena práctica, normaliza o estandariza siempre tus datos después de haberlos dividido en datasets de entrenamiento y de prueba. Si se detecta una fuga de datos, es esencial volver a evaluar el modelo usando un dataset correctamente separado para obtener una valoración justa de su rendimiento.

AWS ofrece Amazon SageMaker Data Wrangler para ayudarte a dividir tu dataset. Veamos cómo con más detalle.

### Amazon SageMaker Data Wrangler

Además de ser tu herramienta integral (_one-stop shop_) para la ingeniería de características, Amazon SageMaker Data Wrangler puede ayudarte a dividir tu dataset en datasets de entrenamiento, validación y prueba con poco o nada de código. Además, en concreto, puedes usar la transformación _Split data_ basada en estas cuatro técnicas de uso común:

- **División aleatoria (_random split_).** Esta técnica asegura que cada subconjunto (entrenamiento, validación, prueba) tenga una distribución similar de categorías, lo que previene el sesgo hacia alguna clase en particular. Este método es particularmente útil cuando no necesitas preservar el orden de tus datos de entrada.
- **División ordenada (_order split_).** Esta técnica asegura que los datos se dividan de forma que se preserve el orden, lo que previene la fuga de datos al asegurar que la información pasada o futura no se superponga entre los datasets de entrenamiento, validación y prueba. Es particularmente útil para datos de series de tiempo o cualquier escenario en el que el orden de los puntos sea crítico.
- **División estratificada (_stratified split_).** Esta técnica asegura que cada subconjunto mantenga la misma proporción de categorías que el dataset original. Es particularmente útil en problemas de clasificación con datos desbalanceados, porque ayuda a mantener la distribución de clases en todos los subconjuntos, lo que produce una evaluación y un entrenamiento del modelo más confiables.
- **División por clave (_split by key_).** Esta técnica asegura que ninguna combinación de valores de las columnas de entrada aparezca en más de una de las particiones. Es particularmente útil para evitar la fuga de datos en datos no ordenados. También permite agrupar de forma consistente, manteniendo juntos los datos relacionados, con lo que se preserva la integridad de las particiones.

> [!warning] Nota de precisión: lo que dice la documentación de Data Wrangler
>
> - **División aleatoria.** La documentación la describe solo como «una muestra aleatoria, sin solapamiento, del dataset original», sin garantizar proporciones de clases; la que las garantiza es la estratificada. Añade que en datasets grandes puede ser más lenta y costosa de calcular que la ordenada.
> - **División ordenada.** En una división 80/20, el primer 80 % de las filas, en el orden en que están, va a entrenamiento y el último 20 % a prueba. El orden lo da el dataset, así que conviene ordenar antes por la columna de tiempo.
> - **División por clave.** El ejemplo de la documentación es una columna `customer_id`: ningún cliente aparece en más de una partición.

Después de elegir la técnica de la transformación de división, puedes especificar los porcentajes para los datasets de entrenamiento, validación y prueba. Por ejemplo, podrías asignar un 70 % al entrenamiento, un 20 % a la validación y un 10 % a la prueba. Por último, puedes aplicar la transformación de división, y Amazon SageMaker Data Wrangler dividirá automáticamente tu dataset en los subconjuntos especificados.

Dos detalles de funcionamiento, según la documentación vigente. Primero, Data Wrangler usa por defecto una **semilla aleatoria** fija, de modo que la misma división se reproduce cada vez que el flujo se ejecuta como trabajo; puedes cambiarla para obtener otra división reproducible. Segundo, para rendir en datasets grandes calcula las proporciones de forma aproximada, con un **umbral de error** configurable entre 0 y 1: con 10 000 filas, una división 80/20 y un error de 0.001, el conjunto de entrenamiento tendrá entre 7990 y 8010 filas. Con un umbral de 0 la división es exacta, pero más lenta. Cada subconjunto resultante aparece como una rama del flujo que se exporta por separado, normalmente a su propio prefijo de S3.

## Escenarios donde este servicio es la opción obligada

Este capítulo no trata de un solo servicio, sino de varios. Por eso cada escenario se centra en uno distinto de los que aparecen en el capítulo, elegido porque, dadas las restricciones del caso, es la única opción razonable dentro de AWS. Los tres cubren las tres capas que recorre el capítulo: dónde viven los datos (S3), quién puede verlos (Lake Formation) y cómo se sirven las características ya construidas (Feature Store). Ground Truth y Clarify quedaron fuera a propósito: como ya no admiten clientes nuevos, no pueden ser la opción obligada de un proyecto que empieza hoy. Las empresas son ficticias; las restricciones son las que aparecen en proyectos reales.

### Escenario 1: detección de fraude en tiempo real en una fintech → Amazon SageMaker Feature Store

**Contexto.** Una fintech que emite tarjetas de crédito procesa, en hora pico, unos miles de autorizaciones de pago por segundo. Su equipo de ciencia de datos entrenó un modelo de fraude con dos años de historia, que usa características como «número de compras de esta tarjeta en la última hora», «monto promedio de los últimos 30 días» y «comercios distintos en las últimas 24 horas». El modelo está desplegado en un endpoint de SageMaker y debe puntuar cada pago **antes** de aprobarlo. Los equipos de riesgo crediticio y de marketing quieren reutilizar esas mismas características en sus propios modelos.

**Restricciones clave.**

1. **Latencia:** el sistema de autorización da a todo el proceso un presupuesto de unas decenas de milisegundos, incluida la lectura de las características de la tarjeta.
2. **Consistencia:** las características deben llegar al modelo con los mismos valores con que se calcularon para entrenar. Un desfase entre dos bases de datos ya les costó un modelo que rendía bien en la evaluación y mal en producción.
3. **Historia exacta:** para reentrenar necesitan saber qué valor tenía cada característica **en el instante** de cada transacción pasada, y auditoría interna exige poder reproducirlo.
4. **Equipo:** es de ciencia de datos, no de infraestructura, y la dirección prohibió construir y mantener a mano la sincronización entre un almacén rápido y uno histórico.
5. **Reutilización:** otros equipos deben poder descubrir las características y leerlas con sus propios permisos.

**Por qué este servicio.**

- (1) → El **almacén online** conserva el último registro de cada tarjeta y lo sirve con `GetRecord` en baja latencia. Con un endpoint privado de PrivateLink en la misma VPC que la aplicación, el tráfico no sale de la red de la empresa ni cruza zonas de disponibilidad innecesariamente.
- (2) y (4) → Con el almacén online y el offline activados en el mismo grupo de características, cada registro se ingiere **una sola vez** y Feature Store mantiene ambos sincronizados. No hay dos caminos de escritura que puedan divergir ni código de sincronización que mantener. El tipo `Standard_V2` permite además actualizar solo el contador que cambió y rechaza escrituras que lleguen desordenadas, usando la hora del evento.
- (3) → El **almacén offline** nunca sobrescribe: agrega cada registro con su hora del evento en archivos Parquet en S3, registrados en el Data Catalog de Glue, y se consulta con Athena. Eso permite reconstruir el valor vigente en cualquier fecha pasada.
- (5) → Los grupos de características se buscan por nombre, descripción y etiquetas, y el acceso de cada equipo se concede con políticas de IAM sobre grupos concretos.

**Por qué no las alternativas.**

- **Amazon DynamoDB** (la base de datos NoSQL de clave-valor, fully managed, de AWS) resuelve la latencia, pero solo guarda el valor actual y obligaría a construir y mantener aparte el flujo histórico hacia S3, que es lo que prohíbe la restricción 4.
- **Amazon ElastiCache** (caché en memoria administrada, con motores como Redis OSS o Valkey) es aún más rápida, pero tampoco conserva la historia ni resuelve la consistencia con el entrenamiento; de hecho, es lo que usa por dentro el nivel `InMemory` de Feature Store, que por eso no admite almacén offline.
- **Amazon S3 consultado con Athena** tiene la historia completa, pero cada consulta tarda segundos, no milisegundos (restricción 1).
- **Amazon Redshift** (el _data warehouse_ administrado de AWS) está pensado para consultas analíticas sobre muchas filas, no para miles de lecturas por segundo de una fila cada una (restricción 1).
- **SageMaker Data Wrangler** y **AWS Glue DataBrew** **calculan** características, pero no las almacenan ni las sirven; pueden alimentar a Feature Store, no reemplazarlo.

### Escenario 2: consorcio hospitalario de investigación → AWS Lake Formation

**Contexto.** Un consorcio de cinco hospitales reúne historias clínicas seudonimizadas y resultados de laboratorio en un data lake en S3, en Parquet y catalogado con AWS Glue. Unos 40 investigadores, cada uno con la cuenta de AWS de su propia institución, construyen modelos de riesgo de reingreso desde notebooks de SageMaker, consultando los datos con Athena.

**Restricciones clave.**

1. **Filas:** cada investigador solo puede ver a los pacientes de su hospital, salvo en proyectos multicéntricos aprobados por el comité de ética.
2. **Columnas:** fuera del equipo de gobierno de datos, nadie puede ver identificadores indirectos, como el código postal completo o la fecha de nacimiento exacta.
3. **Una sola copia:** el comité prohíbe crear copias filtradas por hospital o por proyecto: cada copia es otra superficie de riesgo y acaba desincronizándose de la original.
4. **Auditoría:** los permisos deben estar declarados en un solo lugar, en un formato que el comité pueda revisar, y cada acceso debe quedar registrado.
5. **Varias cuentas:** los investigadores pertenecen a cuentas de AWS distintas.

**Por qué este servicio.**

- (1) y (2) → Los **filtros de datos** de Lake Formation implementan seguridad a nivel de **fila** (una expresión del tipo `hospital_id = 'H3'`), de **columna** (incluir o excluir columnas) y de **celda** (ambas a la vez), y se asignan al conceder el permiso `SELECT` sobre una tabla del catálogo.
- (3) → Los filtros se aplican **al leer**, sobre la misma tabla: cada investigador ve un subconjunto distinto de la única copia de los datos.
- (4) → Los permisos se conceden y revocan de forma centralizada, con una lógica parecida al `GRANT` de SQL, y los accesos quedan registrados en CloudTrail.
- (5) → Lake Formation permite compartir tablas, o partes de ellas definidas por filtros, con otras cuentas de AWS.
- Los investigadores no reciben permisos directos sobre el bucket: consultan a través de motores integrados (Athena, Amazon EMR, Redshift Spectrum), que aplican los filtros con credenciales temporales que entrega Lake Formation. Consulta en la documentación la lista vigente de motores integrados.

**Por qué no las alternativas.**

- **Políticas de IAM y de bucket de S3** controlan el acceso por bucket, prefijo u objeto, y un archivo Parquet contiene todas sus filas y columnas: no hay forma de ocultar una columna dentro de él sin crear archivos separados (restricción 3).
- **S3 Access Points** y **S3 Access Grants** (mecanismos para dar acceso a S3 por aplicación, por usuario o por prefijo) tienen la misma granularidad de objeto: no filtran filas ni columnas.
- **AWS Glue DataBrew** o un **trabajo de Glue** que genere versiones enmascaradas por hospital crean justamente las copias que prohíbe la restricción 3.
- **Amazon Macie** (servicio que descubre y clasifica datos sensibles en S3) ayudaría a encontrar las columnas con datos personales, pero no controla quién puede leerlas.
- **Amazon Redshift** tiene seguridad por fila y por columna, pero exige cargar los datos en el warehouse, lo que crea otra copia (restricción 3), y el acceso directo al lago en S3 quedaría fuera de su control.

### Escenario 3: agricultura de precisión con imágenes de drones → Amazon S3 como base del data lake

**Contexto.** Una empresa de agricultura de precisión opera drones que, en temporada, generan alrededor de 2 TB diarios de imágenes multiespectrales. A eso se suman lecturas de sensores de suelo en JSON cada cinco minutos y datos climáticos en CSV descargados de servicios externos. Con esos datos entrena en SageMaker modelos de detección de plagas, cataloga todo con crawlers de Glue y consulta los metadatos con Athena. El volumen pasará de decenas a cientos de terabytes en pocos años. Por contrato con las aseguradoras agrícolas, las imágenes de cada temporada deben conservarse siete años sin posibilidad de borrado, aunque después de la primera temporada casi no se consultan.

**Restricciones clave.**

1. **Formatos heterogéneos** (imágenes, JSON, CSV, Parquet) que deben guardarse **sin definir un esquema antes**.
2. **Crecimiento sin planificar capacidad:** nadie en la empresa sabe administrar ni ampliar discos.
3. **Un solo lugar para todos los consumidores:** Athena, los crawlers de Glue, el entrenamiento de SageMaker y el almacén offline de Feature Store deben leer los mismos datos **sin copiarlos** entre sistemas.
4. **Costo bajo para datos fríos**, idealmente con transición automática, y **retención inalterable** de siete años.
5. **Durabilidad máxima:** una temporada de imágenes perdida es irrecuperable, porque no se puede volver a volar sobre el campo del año pasado. Aquí la opción «recolectar de nuevo» de la sección de valores faltantes no existe.

**Por qué este servicio.**

- (1) → S3 es almacenamiento de objetos: guarda cualquier archivo tal como llega, que es el _schema-on-read_ del data lake.
- (2) → La capacidad no se aprovisiona: crece sola y se paga por lo que se guarda.
- (3) → Athena consulta archivos en S3, los crawlers de Glue catalogan S3, los trabajos de entrenamiento de SageMaker leen sus canales desde S3 y el almacén offline de Feature Store vive en S3: es la fuente única de verdad del capítulo.
- (4) → Las reglas de **ciclo de vida** de S3 mueven automáticamente los objetos a clases Glacier por antigüedad, y **S3 Object Lock** en modo de cumplimiento impide que nadie, ni siquiera el administrador principal de la cuenta, borre o modifique un objeto antes de que venza su periodo de retención.
- (5) → Su objetivo de diseño de durabilidad es de once nueves, con réplicas en varias zonas de disponibilidad en las clases que no son de una sola zona.

**Por qué no las alternativas.**

- **Amazon EBS** es almacenamiento de bloques: un disco virtual unido a una instancia dentro de una sola zona de disponibilidad, con un tamaño que se fija de antemano (del orden de decenas de TiB como máximo por volumen, según el tipo; verifica los límites vigentes), y Athena y Glue no pueden leerlo (restricciones 2 y 3).
- **Amazon EFS** es un sistema de archivos compartido que se monta por **NFS** (_Network File System_, el protocolo estándar de Linux para montar una carpeta remota como si fuera local); crece solo, pero en su clase estándar cuesta varias veces más por GB que S3 Standard (del orden de diez veces en us-east-1; verifica los precios vigentes), y Athena y los crawlers de Glue no trabajan sobre él (restricciones 3 y 4).
- **Amazon FSx for Lustre** (sistema de archivos paralelo de alto rendimiento, heredado del supercómputo) se usa en ML como caché rápida **vinculada a un bucket de S3** para acelerar el entrenamiento: complementa al lago, no lo reemplaza.
- **Amazon FSx for Windows File Server** (carpetas compartidas por **SMB**, el protocolo de archivos compartidos de Windows), **FSx for NetApp ONTAP** (el sistema de almacenamiento empresarial de NetApp, administrado por AWS) y **FSx for OpenZFS** (el sistema de archivos ZFS, conocido por sus instantáneas y su compresión) están pensados para migrar aplicaciones que dependen de esos sistemas, no para un lago consultado con Athena (restricciones 3 y 4).
- **Amazon Redshift**, **Amazon RDS** (bases de datos relacionales administradas) o **DynamoDB** exigen esquema al escribir y no están hechos para guardar terabytes de imágenes binarias (DynamoDB, por ejemplo, limita cada elemento a 400 KB) (restricción 1).

## Resumen (_Summary_)

En este capítulo aprendiste lo que necesitas hacer para preparar tu dataset crudo como un conjunto significativo de características, que almacena el contenido relevante de tus datos en un formato adecuado que pueda alimentar a un algoritmo de ML y, en última instancia, ser entendido por él para producir predicciones precisas y confiables.

Se presentaron los distintos tipos de datos (categóricos, numéricos, textuales, de imagen y de series de tiempo) para agrupar lógicamente las distintas técnicas de transformación de datos.

Al hacer la ingeniería de tus datos con selección y extracción de características, aprendiste a seguir un enfoque minimalista (reducción de dimensionalidad) para elegir selectivamente las características relevantes que importan a tu modelo de ML. Se cubrieron todos los aspectos de la limpieza de datos, con énfasis en la detección y eliminación de valores atípicos, así como en los métodos de imputación de datos faltantes. Se proporcionaron técnicas de ingeniería de características para datos numéricos, categóricos, de series de tiempo y no estructurados, con ejemplos en el lenguaje de programación Python.

Aprendiste que Amazon SageMaker Data Wrangler y Amazon SageMaker Feature Store son los más adecuados, respectivamente, para realizar la ingeniería de características y para almacenar, compartir y gestionar las características resultantes.

Por último, aprendiste a desarrollar una estrategia de etiquetado eficaz con Amazon SageMaker Ground Truth, a mitigar el desbalance de clases con Amazon SageMaker Clarify y a dividir tu dataset procesado en datasets de entrenamiento, validación y prueba usando Amazon SageMaker Data Wrangler.

En los próximos capítulos aprenderás a elegir un enfoque de modelado, a entrenar y refinar tu modelo y a evaluar su rendimiento.

## Puntos esenciales para el examen (_Exam Essentials_)

**Conoce las técnicas de ingeniería de características que manejan valores atípicos.** Para manejar valores atípicos en la ingeniería de características, es esencial aplicar técnicas que mitiguen su impacto en tu modelo de ML. Estos métodos incluyen eliminar los valores atípicos por completo (siempre que sea posible) o transformarlos con enfoques como la transformación logarítmica, que puede reducir el sesgo de tus datos. Además, los métodos de imputación, como reemplazar los valores atípicos por la mediana o la media, pueden ser eficaces para mantener la integridad del dataset reduciendo a la vez su impacto en el modelo de ML resultante.

**Conoce las técnicas de ingeniería de características que manejan el sesgo.** Para manejar el sesgo (la asimetría de la distribución), técnicas como la transformación logarítmica, la transformación de raíz cuadrada y las transformaciones de Box-Cox o Yeo-Johnson son opciones excelentes. Estas técnicas pueden ayudar a normalizar las distribuciones y hacerlas más adecuadas para el análisis estadístico, sobre todo cuando se trabaja con datos sesgados, en los que la mayoría de los valores se agrupan hacia un lado de la distribución con una cola larga del otro.

**Conoce las técnicas de ingeniería de características que no manejan el sesgo.** La estandarización por puntuación Z no aborda directamente el sesgo. Escala los datos para que tengan una media de 0 y una desviación estándar de 1, pero si los datos originales están sesgados, siguen sesgados después de la transformación.

El escalado MinMax tampoco aborda directamente el sesgo. Reajusta el tamaño de los datos para que quepan dentro de un rango específico, normalmente de 0 a 1, pero no transforma la distribución de los datos. Si tus datos están sesgados, el sesgo se mantiene incluso después del escalado.

**Conoce cuándo hacer ingeniería de características con normalización frente a estandarización.** La normalización es útil cuando importa la escala, mientras que la estandarización se prefiere cuando la clave es la distribución.

**Conoce las técnicas de ingeniería de características que se usan para la estandarización.** Para la estandarización, cuyo propósito es asegurar que cada característica de tu dataset contribuya por igual al rendimiento del modelo de ML, considera usar la puntuación Z con datos de distribución normal.

**Conoce las técnicas de ingeniería de características que se usan para la normalización.** Para la normalización, cuyo propósito es escalar cada característica de tu dataset al mismo rango (normalmente de 0 a 1), considera usar la técnica de escalado MinMax.

**Conoce las técnicas de ingeniería de características que se usan para datos categóricos.** Usa la codificación por etiquetas (_label encoding_) para datos categóricos ordinales y modelos de ML basados en árboles. Usa la codificación one-hot para conjuntos pequeños de datos categóricos nominales y modelos de ML no basados en árboles. Usa la codificación binaria para reducir la dimensionalidad que causan las técnicas one-hot. Usa el hashing de características para datos categóricos de alta cardinalidad, sobre todo en casos de uso en los que hay que optimizar los recursos de cómputo y la solución debe ser económica.

**Conoce las técnicas de ingeniería de características que se usan para datos de imagen.** Para datos de imagen, puedes extraer características de tu dataset crudo con Amazon SageMaker JumpStart. Amazon Rekognition también es una opción válida para el análisis básico de imágenes y la extracción de características, según tu caso de uso. Después de extraer las características relevantes del dataset, puedes aprovechar Amazon SageMaker Data Wrangler para transformar tu dataset con normalización y reducción de dimensionalidad. Por último, puedes almacenar tus características en Amazon SageMaker Feature Store.

**Conoce las técnicas de ingeniería de características que se usan para datos textuales.** Para datos textuales, puedes extraer características de tu dataset crudo con Amazon Comprehend o Amazon Textract. Para una extracción de características más personalizada, Amazon SageMaker te permite usar la tokenización, el stemming y la lematización para hacer una selección básica de palabras o reducir las palabras a su forma raíz para un análisis semántico posterior.

> [!warning] Nota de precisión
> La tokenización, el stemming y la lematización no son funciones de SageMaker: «SageMaker te permite usarlas» significa que corres bibliotecas de Python (como NLTK o spaCy) en un notebook o en un trabajo de procesamiento de SageMaker. Sin código, AWS Glue DataBrew ofrece tokenización, eliminación de palabras vacías y stemming en su paso `TOKENIZATION`. Y el orden habitual con los servicios administrados es Textract primero (de documento escaneado a texto) y Comprehend después (de texto a entidades y sentimiento).

**Conoce los servicios de AWS que se usan para el etiquetado de datos.** Para el etiquetado de datos, cuyo propósito es enriquecer tu dataset con las predicciones reales o con metadatos significativos que ayuden a tu modelo a aprender más rápido, considera usar Amazon SageMaker Ground Truth.

> [!warning] Estado del servicio
> Ground Truth ya no admite clientes nuevos, y Mechanical Turk cierra el 30-09-2026 (ver la sección de Ground Truth).

**Conoce cómo gestionar el desbalance de clases.** Para mitigar los riesgos que introduce el desbalance de clases, selecciona una característica sensible al sesgo (faceta) y considera calcular para ella las métricas de sesgo previas al entrenamiento (que proporciona Amazon SageMaker Clarify), como el desbalance de clases (CI) o la diferencia en proporciones de etiquetas (DPL). Determina la estrategia adecuada en consecuencia. Puede ser el sobremuestreo de datos, el submuestreo o la ponderación de clases.

> [!warning] Estado del servicio
> Clarify ya no admite clientes nuevos. Las métricas CI y DPL se calculan igual a partir de sus fórmulas publicadas, y el remuestreo (sobremuestreo aleatorio, submuestreo aleatorio y SMOTE) está disponible sin código en la transformación _Balance data_ de Data Wrangler.

**Conoce las diferencias entre los datasets de entrenamiento, validación y prueba.** El dataset de entrenamiento es donde el modelo aprende patrones, relaciones y características de los datos. En esencia, es donde el modelo recibe su «educación». Una vez entrenado el modelo, el dataset de validación ayuda a ajustar los hiperparámetros y a afinar el modelo para mejorar su rendimiento y su generalización a datos no vistos. Es como un examen de práctica antes del examen final. El dataset de prueba se usa para evaluar el rendimiento general del modelo. Proporciona una valoración imparcial de qué tan bien generaliza el modelo a datos nuevos, no vistos. Piensa en él como el examen final que determina la calificación del modelo.

> [!tip] Lo que añaden estas notas para el examen
> Además de «qué técnica resuelve qué problema», que es lo que resumen los puntos anteriores, las preguntas de servicios suelen girar en torno a estas distinciones de infraestructura:
>
> - **S3** es el almacenamiento del data lake; **Glue** cataloga (crawler + Data Catalog) y transforma a escala (trabajos de Spark); **Athena** consulta con SQL sin servidores; **Lake Formation** gobierna permisos por fila, columna y celda.
> - **DataBrew** es preparación visual sin código de la familia Glue (recetas y trabajos); **Data Wrangler** es preparación visual _low-code_ dentro de SageMaker (Canvas), con exportación a Feature Store, Pipelines y S3, y con _Split data_ y _Balance data_.
> - **Feature Store** separa el almacén **online** (último valor, baja latencia, tiempo real) del **offline** (historial en S3, entrenamiento y consultas _point-in-time_), sincronizados con una sola ingesta.
> - **Rekognition** y **Comprehend** extraen características sin entrenar nada; **JumpStart** da modelos preentrenados que tú despliegas; **Textract** es OCR con estructura (formularios y tablas).
> - **Bedrock** cobra los modelos de texto por tokens de entrada y salida.

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

4. Estás entrenando un modelo y, durante el análisis de datos, observas distintas variables de entrada que varían significativamente. Quieres asegurarte de que tu dataset no tenga características con un valor más grande que influyan mucho en la capacidad predictiva del modelo. ¿Qué transformación es la más adecuada en este escenario?
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

> [!warning] Nota sobre las preguntas 3 y 9
> Ambas mencionan servicios que ya no admiten clientes nuevos (Clarify en la 3, Ground Truth en la 9). Siguen siendo válidas como preguntas del libro, pero en el MLA-C02 las habilidades evaluadas ya no nombran esos servicios. En la 9, recuerda que Comprehend y Rekognition **producen** predicciones con modelos ya entrenados; no son herramientas para que personas etiqueten datos.

## Glosario

- **Algoritmo integrado (SageMaker).** Algoritmo que AWS ya empaquetó en un contenedor; solo se configura un trabajo de entrenamiento que lo use.
- **Almacén offline (Feature Store).** Historial completo de registros de características, en Parquet dentro de un bucket de S3 y registrado en el Data Catalog de Glue; sirve para entrenar y para consultas _point-in-time_.
- **Almacén online (Feature Store).** Último registro de cada identificador, con lecturas de baja latencia por `GetRecord`; sirve para responder en tiempo real.
- **Almacenamiento de archivos.** Carpeta compartida por red que varias máquinas montan a la vez (en AWS: EFS y FSx).
- **Almacenamiento de bloques.** Disco virtual conectado a una máquina, que el sistema operativo usa como disco propio (en AWS: EBS).
- **Almacenamiento de objetos.** Archivos guardados enteros, con metadatos y una clave, dentro de buckets y accesibles por una API web (en AWS: S3).
- **Amazon Athena.** Motor SQL serverless que consulta archivos directamente en S3 y cobra por datos escaneados.
- **Amazon Bedrock.** Servicio fully managed que da acceso por API a modelos fundacionales de varios proveedores; cobra los de texto por tokens.
- **Amazon Comprehend.** Servicio de NLP ya entrenado que extrae entidades, frases clave, sentimiento e idioma por API, de forma sincrónica o en trabajos por lotes sobre S3.
- **Amazon DynamoDB.** Base de datos NoSQL de clave-valor, fully managed, con lecturas de milisegundos y elementos de hasta 400 KB.
- **Amazon EBS (Elastic Block Store).** Discos virtuales para instancias, dentro de una sola zona de disponibilidad.
- **Amazon EFS (Elastic File System).** Sistema de archivos compartido por NFS que crece automáticamente.
- **Amazon ElastiCache.** Caché en memoria administrada (Redis OSS, Valkey o Memcached); respalda el nivel `InMemory` de Feature Store.
- **Amazon FSx.** Familia de sistemas de archivos administrados: Lustre, Windows File Server, NetApp ONTAP y OpenZFS.
- **Amazon Macie.** Servicio que descubre y clasifica datos sensibles en S3, sin controlar quién accede a ellos.
- **Amazon Mechanical Turk (MTurk).** Mercado de crowdsourcing de tareas humanas, usable como fuerza de trabajo en Ground Truth; cierra el 30-09-2026.
- **Amazon RDS.** Servicio de bases de datos relacionales administradas.
- **Amazon Redshift.** Data warehouse administrado de AWS para consultas analíticas SQL.
- **Amazon Rekognition.** Servicio de visión ya entrenado que detecta objetos, escenas, texto y rostros por API y cobra por imagen.
- **Amazon S3 (Simple Storage Service).** Almacenamiento de objetos de AWS, base habitual de un data lake.
- **Amazon SageMaker (AI).** Plataforma de ML de AWS: notebooks, preparación de datos, entrenamiento, despliegue y monitoreo; hoy se llama SageMaker AI.
- **Amazon SageMaker Canvas.** Entorno visual de SageMaker AI desde el que hoy se usa Data Wrangler.
- **Amazon SageMaker Clarify.** Herramienta de SageMaker para medir sesgo y explicar predicciones mediante trabajos de procesamiento; ya no admite clientes nuevos.
- **Amazon SageMaker Data Wrangler.** Herramienta visual low-code de SageMaker para importar, transformar, analizar, balancear y dividir datos; hoy se usa desde Canvas.
- **Amazon SageMaker Feature Store.** Repositorio fully managed para almacenar, compartir y servir características, con almacenes online y offline.
- **Amazon SageMaker Ground Truth.** Servicio de etiquetado con personas y automatización; ya no admite clientes nuevos.
- **Amazon SageMaker JumpStart.** Catálogo de modelos preentrenados que se despliegan en endpoints o se usan en trabajos por lotes.
- **Amazon SageMaker Pipelines.** Servicio de SageMaker para encadenar automáticamente preparación de datos, entrenamiento y despliegue.
- **Amazon Textract.** Servicio de OCR que además extrae formularios (clave-valor), tablas y respuestas a consultas; cobra por página.
- **Aplicaciones de línea de negocio (LOB).** Sistemas que sostienen los procesos centrales de una empresa, como el ERP, el CRM o la nómina.
- **Archivo de manifiesto (Ground Truth).** Archivo JSON Lines en S3 que enumera los objetos que se etiquetarán (entrada) o sus etiquetas resultantes (salida).
- **At-least-once (entrega al menos una vez).** Garantía de entrega de un sistema de ingesta que prefiere reenviar un mensaje antes que perderlo, lo que puede generar duplicados.
- **AWS CloudTrail.** Servicio que registra el historial de llamadas a la API de AWS de una cuenta, útil para auditoría.
- **AWS Config.** Servicio que evalúa la configuración de los recursos contra reglas y reporta los que incumplen.
- **AWS Glue.** Servicio serverless de integración de datos: Data Catalog, crawlers y trabajos ETL.
- **AWS Glue Data Catalog.** Registro central de metadatos (esquemas y ubicaciones en S3) que convierte archivos en tablas consultables.
- **AWS Glue DataBrew.** Herramienta visual sin código para limpiar y normalizar datos con recetas reutilizables que se ejecutan como trabajos serverless.
- **AWS KMS (Key Management Service).** Servicio que crea y custodia claves de cifrado y registra cada uso.
- **AWS Lake Formation.** Capa de gobernanza de un data lake con permisos centralizados a nivel de base de datos, tabla, columna, fila y celda.
- **AWS Marketplace.** Tienda de software y servicios de terceros integrada en AWS; ahí se contratan proveedores de etiquetado.
- **AWS Nitro Enclaves.** Entorno aislado dentro de una instancia EC2 para procesar datos sensibles en memoria (cifrado en uso).
- **AWS PrivateLink.** Tecnología para llegar a un servicio de AWS desde una VPC por una dirección privada, sin salir a internet.
- **Base de datos operacional.** Base que usa una aplicación en su día a día para registrar transacciones.
- **Boto3.** SDK de AWS para Python.
- **BoundingBox (Rekognition).** Rectángulo que devuelve la API para ubicar un objeto, expresado con `Left`, `Top`, `Width` y `Height` como fracciones del tamaño de la imagen.
- **Bucket.** Contenedor de objetos de S3, con nombre único, dentro del cual cada objeto se identifica por su clave.
- **Canal (_channel_) de entrenamiento.** Entrada con nombre de un trabajo de entrenamiento de SageMaker (por ejemplo, `train` o `validation`) que apunta a un prefijo de S3.
- **Cifrado en reposo.** Protección de los datos mientras están guardados.
- **Cifrado en tránsito.** Protección de los datos mientras viajan por la red, con TLS.
- **Cifrado en uso.** Protección de los datos mientras se procesan en memoria, por ejemplo con enclaves.
- **Clase de almacenamiento (S3).** Nivel de precio y de acceso de S3 (Standard, Standard-IA, Glacier, Intelligent-Tiering, etc.).
- **Clave (S3).** Nombre completo de un objeto dentro de un bucket, como `ventas/2025/enero.parquet`.
- **Compliance (cumplimiento normativo).** Capacidad de demostrar ante auditores o reguladores que se cumplen normas como el RGPD o la HIPAA.
- **Consulta _point-in-time_ (Feature Store).** Reconstrucción del valor que tenía una característica en un instante pasado, usando la hora del evento del almacén offline.
- **Crawler (AWS Glue).** Proceso que recorre archivos en S3, infiere su esquema y lo registra como tabla en el Data Catalog.
- **Crowdsourcing.** Reparto de una tarea entre una multitud de personas externas que cobran por tarea.
- **Data lake (lago de datos).** Repositorio central que guarda datos crudos en su formato original y aplica el esquema al leer.
- **Data warehouse (almacén de datos).** Base analítica que exige un esquema definido antes de cargar los datos.
- **Disponibilidad.** Probabilidad de poder acceder a un dato o servicio en un momento dado.
- **DPU (Data Processing Unit).** Unidad de capacidad de un trabajo de Glue (según la documentación, 4 vCPU y 16 GB de memoria); se cobra por DPU-hora.
- **Durabilidad.** Probabilidad de no perder un dato guardado; el objetivo de diseño de S3 es 99.999999999 %.
- **Endpoint (SageMaker).** Punto de acceso HTTPS respaldado por instancias que alojan un modelo; se cobra por hora mientras exista.
- **Feature group (grupo de características).** Recurso principal de Feature Store: definición de columnas, identificador de registro y hora del evento.
- **Filtro de datos (Lake Formation).** Especificación de columnas y de una expresión de filas que se asigna al conceder `SELECT` sobre una tabla.
- **Flujo de datos (Data Wrangler).** Secuencia de pasos de importación, transformación y análisis que se diseña sobre una muestra y se ejecuta sobre el dataset completo.
- **Formato disperso (_sparse_).** Forma de almacenar datos con muchos ceros guardando solo los valores distintos de cero y su posición.
- **Fuente única de verdad.** Una sola copia autorizada de los datos a la que apuntan todos los sistemas.
- **Fuerza de trabajo privada.** Empleados o contratistas propios que etiquetan en un portal privado de Ground Truth.
- **Fully managed (totalmente administrado).** Servicio cuya infraestructura opera AWS; el usuario solo lo consume y paga por uso.
- **Glue job (trabajo de Glue).** Script, normalmente PySpark, que Glue ejecuta en infraestructura serverless cobrando por DPU-hora.
- **Gobernanza de datos.** Reglas y procesos que determinan quién puede usar qué datos, para qué y cómo se audita.
- **Guardrails (barreras de seguridad).** Reglas que acotan lo que cualquiera puede hacer en una cuenta, como S3 Block Public Access o las SCP.
- **Hora del evento (_event time_).** Marca de tiempo que indica cuándo eran válidos los valores de un registro de Feature Store.
- **Human-in-the-loop.** Diseño en el que las personas revisan o deciden en ciertos puntos de un proceso automático.
- **IAM (Identity and Access Management).** Servicio donde se define, mediante políticas, qué identidad puede hacer qué acción sobre qué recurso.
- **Identificador de registro (_record identifier_).** Columna que identifica cada entidad en un grupo de Feature Store, como `customer_id`.
- **Intelligent-Tiering (S3).** Clase de S3 que mueve cada objeto entre niveles de precio según su patrón de acceso.
- **Low-code.** Herramienta en la que se trabaja sobre todo configurando pasos visuales, con poco código.
- **Lustre.** Sistema de archivos paralelo de alto rendimiento del supercómputo; en AWS, FSx for Lustre.
- **Marca de tiempo Unix.** Número de segundos (o fracciones) transcurridos desde el 1 de enero de 1970.
- **Modelo fundacional (en Bedrock).** Modelo grande ya entrenado por AWS o por terceros que Bedrock ofrece por API.
- **NetApp ONTAP.** Sistema operativo de almacenamiento empresarial de NetApp; en AWS, FSx for NetApp ONTAP.
- **NFS (Network File System).** Protocolo estándar, sobre todo en Linux, para montar una carpeta remota como si fuera local.
- **Notebook de SageMaker.** Entorno de Jupyter que corre en una instancia administrada por SageMaker, en Studio o como _notebook instance_.
- **OCR (reconocimiento óptico de caracteres).** Tecnología que convierte la imagen de un texto en texto de computadora.
- **OpenZFS.** Versión de código abierto del sistema de archivos ZFS; en AWS, FSx for OpenZFS.
- **Parquet.** Formato de archivo por columnas y comprimido, que abarata y acelera las consultas en S3.
- **Payload.** Contenido útil de un envío o mensaje individual.
- **Pipeline de inferencia en serie.** Endpoint de SageMaker que ejecuta en cadena un contenedor de preprocesamiento y el del modelo.
- **Política (IAM).** Documento JSON que permite o niega acciones sobre recursos.
- **Prefijo (S3).** Parte inicial de la clave de un objeto, como `proyecto-a/imagenes/`, que la consola muestra como carpeta.
- **Prompt.** Texto de entrada de una petición a un modelo fundacional.
- **Receta (DataBrew).** Secuencia guardada de transformaciones que se puede reaplicar a otros datasets o programar.
- **Rol de IAM / rol de ejecución.** Identidad con permisos que asumen temporalmente personas o servicios, como un notebook o un trabajo que lee S3.
- **S3 Access Points / Access Grants.** Mecanismos para dar acceso a S3 por aplicación, usuario o prefijo, con granularidad de objeto.
- **S3 Block Public Access.** Configuración que impide que un bucket o sus objetos se hagan públicos.
- **S3 Object Lock.** Función que impide borrar o modificar un objeto durante un periodo de retención.
- **S3 Versioning (versionado).** Opción de un bucket que conserva las versiones anteriores de cada objeto.
- **Schema-on-read.** Aplicar el esquema al leer los datos, no al guardarlos.
- **SCP (Service Control Policy).** Política de AWS Organizations que limita las acciones permitidas en todas las cuentas de una organización.
- **SDK (software development kit).** Biblioteca para manejar un servicio desde código, como boto3 o el SDK de SageMaker para Python.
- **Serverless.** Modelo en que no se aprovisionan servidores: el servicio asigna cómputo al vuelo y cobra por uso.
- **SMB (Server Message Block).** Protocolo de carpetas compartidas de Windows.
- **Spark / PySpark.** Motor de procesamiento distribuido que reparte los datos entre varias máquinas; PySpark es su interfaz de Python.
- **Tipo de instancia.** Tamaño de máquina virtual (CPU, memoria, GPU) que se elige para un notebook, un trabajo o un endpoint.
- **TLS.** Protocolo que cifra la comunicación por red, el mismo del candado de HTTPS.
- **Token (en Bedrock).** Unidad en que cada modelo divide el texto y con la que se factura el uso de los modelos de texto.
- **Trabajo de entrenamiento.** Tarea de SageMaker que levanta instancias, lee los canales de S3, entrena y deja el modelo en S3.
- **Trabajo de etiquetado.** Recurso de Ground Truth que reúne datos, tarea, interfaz, fuerza de trabajo y rol de IAM.
- **Trabajo de procesamiento.** Tarea de SageMaker que ejecuta un contenedor sobre datos de S3 en instancias temporales y escribe el resultado en S3.
- **Trabajo de transformación por lotes (_batch transform_).** Tarea de SageMaker que pasa por un modelo todos los datos de un prefijo de S3 y escribe los resultados en otro.
- **VPC (Virtual Private Cloud).** Red privada de una cuenta en AWS, donde viven instancias y endpoints privados.
- **Zona de disponibilidad.** Uno o más centros de datos físicamente separados dentro de una región de AWS.
