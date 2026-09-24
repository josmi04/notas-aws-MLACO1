---
tema: "Capítulo 2 — Ingesta y almacenamiento de datos (versión explicada)"
fuente: Guia Oficial/02_data_ingestion_and_storage.txt
guia-mla-c01: [Dominio 1, 1.1 Ingest and store data]
perfil-lector: profesional de datos/estadística sin experiencia administrando infraestructura
verificado: 2026-09-23
tags: [aws, mla-c01, ingesta, almacenamiento, formatos-de-datos, firehose, kinesis, msk, flink, datasync, glue, s3, athena, efs, fsx, lustre, ontap, openzfs, ebs, rds, dynamodb]
---

> [!info] Cómo leer esta versión
> - **Qué es.** Traducción íntegra al español del capítulo 2 de la guía oficial de estudio, con los mismos encabezados, tablas, casos de uso y afirmaciones. No se eliminó nada. Los encabezados conservan entre paréntesis el título original en inglés, porque el examen se presenta en inglés.
> - **Qué se añadió.** Explicaciones de la terminología de infraestructura (almacenamiento, redes, protocolos, métricas de rendimiento) integradas en el texto la primera vez que aparece cada término; aclaraciones de razonamientos que el libro da por sentados; escalas concretas para las cifras abstractas; **notas de precisión** (recuadros amarillos) cuando el original es impreciso o está desactualizado; la sección «Escenarios donde este servicio es la opción obligada» y un glosario al final.
> - **Figuras.** El `.txt` no incluye las imágenes, así que solo quedan los pies de figura.
> - **Comparaciones con hardware de consumo.** Para dar escala a las cifras de rendimiento se comparan con discos típicos de laptop. Esas cifras de referencia son órdenes de magnitud aproximados de hardware de consumo, no especificaciones de AWS.
> - **Cifras y estado de los servicios.** Se verificaron en la documentación de AWS el 23 de septiembre de 2026. Cambian con frecuencia, así que confírmalos en la documentación oficial vigente antes de usarlos en una decisión real.

> [!warning] Cambios en AWS posteriores a la edición del libro (verificado el 23-09-2026)
> - **Amazon SageMaker** pasó a llamarse **Amazon SageMaker AI**. En este texto se usa el nombre del libro.
> - **Amazon Kinesis Data Analytics for SQL**, el servicio que permitía consultar flujos de datos con SQL y que ofrecía la función `RANDOM_CUT_FOREST` que aparece en las preguntas de repaso, **dejó de funcionar el 27 de enero de 2026**. AWS recomienda Amazon Managed Service for Apache Flink, que también admite SQL.
> - **AWS DataSync Discovery**, la función de descubrimiento que el libro describe en la sección de DataSync, **llegó al fin de su soporte el 20 de mayo de 2025** y ya no se puede usar. El resto de DataSync sigue disponible.
> - AWS descontinuó **AWS Snowcone** en noviembre de 2024, y **AWS Snowball Edge** solo está disponible para clientes existentes desde el **7 de noviembre de 2025**. Para clientes nuevos, AWS sugiere DataSync (transferencia por red) o AWS Data Transfer Terminal (transferencia física en instalaciones de AWS).
> - **AWS Glue for Ray** no admite clientes nuevos desde el **30 de abril de 2026**. Los clientes existentes pueden seguir usándolo, y AWS recomienda ejecutar Ray en Amazon EKS como alternativa.
> - **Apache MXNet**, el framework al que el libro asocia el formato RecordIO, se retiró en septiembre de 2023 y pasó al *Apache Attic* (el archivo de proyectos inactivos de la fundación Apache) en febrero de 2024. Los algoritmos integrados de SageMaker siguen aceptando RecordIO-protobuf.
> - **Amazon AppStream 2.0** se llama ahora **Amazon WorkSpaces Applications**.
> - El servicio independiente **Amazon Glacier** (el original, basado en «bóvedas») no admite clientes nuevos desde el 7 de noviembre de 2025. Esto **no** afecta a las clases de almacenamiento **S3 Glacier**, que son las que menciona este capítulo.

**Capítulo 2**

# Ingesta y almacenamiento de datos (*Data Ingestion and Storage*)

> LOS OBJETIVOS DEL EXAMEN AWS CERTIFIED MACHINE LEARNING (ML) ENGINEER – ASSOCIATE QUE CUBRE ESTE CAPÍTULO PUEDEN INCLUIR, ENTRE OTROS, LOS SIGUIENTES:
>
> ✔ Dominio 1: Preparación de datos para machine learning
> - 1.1 Ingerir y almacenar datos

## Introducción a la ingesta y el almacenamiento (*Introducing Ingestion and Storage*)

La ingesta y el almacenamiento de datos son los dos elementos centrales de la fase *Collect Data* (recolectar datos) del ciclo de vida de machine learning (ML), como se muestra en la Figura 2.1.

*Figura 2.1 El ciclo de vida de ML.*

Como aprendiste en el capítulo 1, tu solución de ML necesita datos para entrenar el modelo seleccionado y generar inferencias. En ML, *inferencia* significa usar el modelo ya entrenado para producir predicciones sobre datos nuevos; no es la inferencia estadística. Los datos pueden llegar en distintas formas (estructurados, semiestructurados o no estructurados), desde distintas fuentes y en distintos momentos.

La **ingesta** es el proceso que recolecta los datos para tu solución de ML y los envía a AWS. Tú, como ingeniero de machine learning en AWS, tendrás que seleccionar el mejor servicio de ingesta de AWS para reunir los datos de distintas fuentes según su volumen, velocidad y variedad. El **volumen** es cuántos datos hay, la **velocidad** es el ritmo al que llegan (un archivo al día o miles de eventos por segundo) y la **variedad** es cuántos formatos y fuentes distintos hay que combinar. Son las «tres V» clásicas del big data, y cada una empuja hacia servicios distintos. Una velocidad alta, por ejemplo, exige los servicios de *streaming* que verás más adelante.

Una vez recolectados, tendrás que guardar los datos ingeridos en una ubicación adecuada y segura que cumpla los requisitos de durabilidad y disponibilidad. Así, tu solución de ML tendrá acceso a los datos para entrenar tu modelo cuando se necesiten (**disponibilidad**) y durante todo el tiempo que se necesiten (**durabilidad**). Son dos propiedades distintas que a menudo se confunden. Un dato puede estar perfectamente a salvo (es durable) y, aun así, no poder leerse durante una hora porque el servicio está caído (no está disponible). La sección «Elegir servicios de almacenamiento de AWS» las cuantifica.

## Ingerir y almacenar datos (*Ingesting and Storing Data*)

Antes de profundizar en la ingesta y el almacenamiento, demos un paso atrás para entender mejor todos los elementos de un problema de ML.

El machine learning forma parte de una disciplina más amplia conocida como **ingeniería de datos**. En este contexto, el ML es una de las tres formas en que se sirven los datos. El libro no dice cuáles son las otras dos. El ciclo que describe el texto coincide con el del libro *Fundamentals of Data Engineering* (Reis y Housley), aunque, como la figura no está en el `.txt`, esta atribución es una inferencia. En ese ciclo, la etapa final, *serving* (servir los datos), tiene tres destinos: la **analítica** (reportes y tableros), el **ML** y el ***reverse ETL***. El reverse ETL devuelve datos ya procesados a los sistemas operativos de la empresa, por ejemplo, un puntaje de propensión calculado en el data warehouse que se envía de vuelta al CRM para que lo vean los vendedores.

La Figura 2.2 ilustra el ciclo de vida de la ingeniería de datos. El primer paso es la **generación**, que se centra en el origen de los datos. Los datos pueden generarse en distintos sistemas fuente, como una base de datos, un dispositivo de Internet de las cosas (IoT) o un sistema de streaming, entre otros. Un dispositivo **IoT** (*Internet of Things*) es un aparato físico con sensores y conexión a la red, como un medidor eléctrico inteligente o un sensor de temperatura de una planta industrial, que envía lecturas continuamente. Un **sistema de streaming** emite los datos como un flujo continuo de eventos pequeños, en lugar de hacerlo en archivos completos. Un ingeniero de datos diseña procesos para consumir datos de los sistemas fuente, pero puede que no los controle. En la práctica, el sistema fuente puede cambiar su esquema, caerse o enviar datos duplicados sin avisar, y la ingesta debe estar preparada para absorberlo.

*Figura 2.2 El ciclo de vida de la ingeniería de datos.*

Tras su generación, los datos se ingieren en su formato o formatos crudos en un sistema de almacenamiento ubicado en AWS, donde se guardan y luego se procesan. Para el examen, necesitas saber elegir un almacén de datos adecuado, y la elección depende del caso de uso. Por ejemplo, podrías querer guardar tus datos crudos como **almacenamiento de objetos** en un **bucket** de S3 porque tus datos se producen por lotes y este tipo de almacenamiento es relativamente barato comparado con otros medios. En el almacenamiento de objetos, cada archivo se guarda entero como un *objeto*: los bytes del archivo, unos metadatos (tipo de contenido, fecha, etiquetas) y una clave única que lo identifica, como `ventas/2024/05/dia-01.csv`. Los objetos viven dentro de un **bucket**, un contenedor cuyo nombre es único en todo S3, y se leen y escriben mediante peticiones HTTP a una API. Es barato porque AWS lo implementa sobre enormes conjuntos de discos compartidos entre millones de clientes. Además, encaja con los datos por lotes, porque un lote suele ser justamente un archivo completo que se escribe una vez y se lee muchas.

Por el contrario, si tus datos se produjeran y consumieran casi en tiempo real, podrías querer guardarlos en un **medio de almacenamiento de streaming**, como Amazon Kinesis Data Streams. *Casi en tiempo real* (*near real time*) significa que el dato puede consumirse segundos después de generarse, no horas. Un medio de almacenamiento de streaming guarda los eventos en el orden en que llegan, como un registro al que solo se le agregan líneas al final, y los conserva durante un periodo configurable para que uno o varios consumidores los lean a su propio ritmo. Se parece a una cola de mensajes, con una diferencia importante: leer un evento no lo borra, así que otro consumidor puede leerlo después.

La Figura 2.3 muestra una vista jerárquica de las capas de almacenamiento en AWS. La capa superior ilustra el almacenamiento a nivel conceptual, con distintos tipos de abstracciones de almacenamiento. La capa intermedia describe el almacenamiento a nivel lógico, con algunos de los sistemas y tipos de almacenamiento más usados. La capa inferior enumera los componentes de infraestructura que hacen funcionar los sistemas de almacenamiento y sus abstracciones. Para cada sistema de almacenamiento suele haber varios servicios de AWS disponibles.

*Figura 2.3 Sistemas de almacenamiento de AWS.*

Como la figura no está en el `.txt`, estos ejemplos de cada capa son ilustrativos y no la transcriben. Una **abstracción** es una forma de organizar los datos para un propósito, como un *data lake* (repositorio central de archivos crudos) o un *data warehouse* (base de datos analítica con esquema fijo). Un **sistema** es la tecnología que la implementa, como un almacén de objetos, una base de datos relacional o un almacenamiento de streaming. La **infraestructura** son los ingredientes físicos: discos, memoria, red y procesadores.

Casi todo este capítulo depende de distinguir tres tipos de almacenamiento, y los tres se apoyan en un concepto previo, el sistema de archivos:

- **Sistema de archivos (concepto base).** Un disco, por sí solo, es una larga fila de bloques numerados de tamaño fijo (por ejemplo, de 4 KB) sin noción de «archivo». El sistema de archivos es la capa de software del sistema operativo que organiza esos bloques en carpetas y archivos con nombre. Lleva la cuenta de qué bloques pertenecen a cada archivo, quién puede leerlo y quién lo está modificando en ese momento. NTFS en Windows, APFS en macOS y ext4 en Linux son sistemas de archivos. Cuando en pandas ejecutas `pd.read_csv("datos/ventas.csv")`, es el sistema de archivos el que traduce esa ruta a bloques del disco.
- **Almacenamiento de bloques (*block storage*).** Entrega un disco «en crudo», es decir, bloques direccionables que una máquina conecta y formatea con su propio sistema de archivos. Es el equivalente del SSD que va dentro de tu laptop. Es el tipo más rápido y de más bajo nivel, pero normalmente lo usa una sola máquina a la vez. En AWS es Amazon EBS.
- **Almacenamiento de archivos (*file storage*).** Es un sistema de archivos que vive en otra máquina, un servidor, y que varias computadoras usan a la vez por la red, con carpetas, permisos y bloqueos incluidos. Es la «unidad de red» compartida de una oficina. En AWS son Amazon EFS y la familia Amazon FSx.
- **Almacenamiento de objetos (*object storage*).** No tiene carpetas reales ni disco que conectar: es un espacio plano de claves, cada una con su objeto completo, al que se accede por HTTP. Se parece a un diccionario de Python gigantesco (`{clave: (bytes, metadatos)}`) expuesto como servicio web. Un fragmento de un objeto no se puede modificar; hay que reemplazar el objeto entero. A cambio, escala prácticamente sin límite y es el más barato por gigabyte. En AWS es Amazon S3.

Para ML, el tipo de almacenamiento decide cómo lo lee tu código. Con bloques o archivos, el código abre rutas como si el disco fuera local. Con objetos, usa una API o una biblioteca que la envuelve (`boto3`, `s3fs`), o un servicio que copia los datos a un disco local antes de entrenar.

Para el examen también necesitas entender cómo extraer los datos almacenados y cómo prepararlos para el algoritmo de ML seleccionado, que se usará para entrenar tu modelo y obtener inferencias. Estas tareas se cubren en los próximos capítulos.

En las próximas secciones aprenderás a elegir el almacén de datos más adecuado para recolectar y alojar los datos de tu solución de ML. Se cubrirán los servicios principales de AWS para ingerir datos de distintas fuentes, junto con sus principales casos de uso.

## Formatos de datos y técnicas de ingesta (*Data Formats and Ingestion Techniques*)

Existen muchos formatos de datos para ingerir y almacenar eficientemente los datos de tu modelo de ML. Si usas el formato y el algoritmo adecuados, tú, como ingeniero de machine learning, puedes optimizar el rendimiento, mejorar la escalabilidad y reducir el tiempo de procesamiento.

AWS admite una variedad de formatos de datos para distintos casos de uso y servicios. Estos son algunos de los formatos más usados para ingerir y almacenar datos en AWS:

- Valores separados por comas (CSV)
- JavaScript Object Notation (JSON)
- Apache Parquet
- Apache Optimized Row Columnar (ORC)
- Apache Avro
- RecordIO

CSV se usa mucho para guardar datos estructurados en formato tabular, mientras que JSON es ideal para datos semiestructurados basados en documentos.

Apache Parquet y Apache ORC son formatos de datos **columnares**, diseñados para optimizar las operaciones de almacenamiento (lecturas y escrituras) con datasets grandes. Los valores de cada columna se guardan en posiciones de memoria contiguas. Dicho de otro modo, dentro del archivo se escriben primero todos los valores de la columna `edad`, luego todos los de `ciudad`, y así sucesivamente, en lugar de escribir fila por fila como un CSV. Esto da los siguientes beneficios:

- La compresión específica por columna ahorra espacio de almacenamiento. Comprime mejor porque los valores de una misma columna son del mismo tipo y suelen repetirse, así que el compresor encuentra patrones con más facilidad que en una fila que mezcla fechas, textos y números.
- Se pueden usar técnicas de codificación y compresión específicas para cada columna. Una **codificación** es una forma compacta de representar valores antes de comprimirlos. Con la *codificación por diccionario*, una columna `país` con millones de repeticiones de diez valores se guarda como enteros pequeños más un diccionario de diez entradas. Con la *codificación por longitud de corrida* (*run-length encoding*), una secuencia como `MX, MX, MX, MX` se guarda como «`MX` × 4».
- Las consultas que piden valores de columnas específicas no necesitan leer la fila completa, lo que mejora el rendimiento. Si una tabla tiene 100 columnas y tu consulta usa 3, el motor lee aproximadamente el 3 % de los datos. Es lo mismo que hace `pd.read_parquet(ruta, columns=[...])`, y en servicios que cobran por datos leídos, como Amazon Athena (que verás más adelante), también reduce el costo.

A diferencia de Apache Parquet y Apache ORC, **Apache Avro** está diseñado para guardar datos en formato **por filas**. Los datos Avro dependen de esquemas, y cuando se leen, el esquema con que se escribieron siempre está presente. Esto permite escribir cada componente de los datos sin sobrecarga por valor, lo que hace que la **serialización** sea rápida y compacta. Serializar es convertir un objeto que está en memoria (un registro, un diccionario) en una secuencia de bytes que se puede guardar en disco o enviar por la red. La «sobrecarga por valor» se entiende al compararlo con JSON, donde cada registro repite los nombres de los campos (`{"edad": 34, "ciudad": "Lima"}`). En Avro, el esquema indica el orden y el tipo de los campos, así que en el archivo solo se escriben los valores, en binario. Esto también facilita su uso con lenguajes de scripting dinámicos, como Python o JavaScript, porque los datos, junto con su esquema, se describen a sí mismos por completo: el programa no necesita conocer el esquema de antemano. Cuando los datos Avro se guardan en un archivo, su esquema se guarda con ellos, de modo que cualquier programa puede procesar los archivos más adelante. Si el programa que lee los datos espera un esquema distinto, el conflicto se resuelve fácilmente, porque ambos esquemas están presentes. Esto se conoce como **evolución de esquema**: si el lector espera un campo nuevo que no existía al escribir, Avro le asigna el valor por defecto que declara el esquema del lector; si el archivo trae un campo que el lector ya no usa, lo ignora.

El libro no lo dice, pero es la razón de que convivan ambos tipos de formato. Los formatos por filas, como Avro, son eficientes para **escribir** registros completos uno a uno, que es lo que ocurre durante la ingesta de streaming (Avro es muy común con Apache Kafka, que verás más adelante). Los formatos columnares, como Parquet, son eficientes para **leer** pocas columnas de muchos registros, que es lo que hace el análisis y el entrenamiento.

Por último, **RecordIO** es un formato de datos que usa principalmente Apache MXNet, un framework de deep learning. La idea básica es dividir los datos en fragmentos individuales llamados *registros* (*records*) y anteponer a cada registro su longitud en bytes, seguida de los datos. Como resultado, RecordIO implementa un formato de archivo para una secuencia de registros. Saber de antemano cuántos bytes ocupa cada registro permite leer el archivo como un flujo continuo y repartirlo entre varios lectores en paralelo sin tener que interpretar su contenido. En SageMaker lo verás casi siempre como **RecordIO-protobuf**: cada registro contiene datos serializados con **Protocol Buffers** (*protobuf*), el formato binario, compacto y con tipos definidos que creó Google. (Recuerda que MXNet ya está retirado; ver el recuadro del inicio.)

Un factor clave para elegir el formato es cuáles admite el algoritmo de ML que piensas usar. Ese algoritmo se entrenará con tus datos, así que la compatibilidad entre el formato de los datos y los requisitos del algoritmo es crucial para el rendimiento y la eficiencia. Si alineas el formato de tus datos con las capacidades del algoritmo, puedes simplificar la ingesta, reducir el preprocesamiento y obtener resultados más precisos y rápidos en tu flujo de trabajo de ML.

La Tabla 2.1 enumera los algoritmos integrados (*built-in*) de Amazon SageMaker junto con los formatos de datos que aceptan. En el capítulo 4 se cubre en detalle cada uno de ellos. Un **algoritmo integrado** es una implementación que AWS ya empaqueta y mantiene: solo le pasas los datos y los hiperparámetros, sin escribir el código de entrenamiento.

**Tabla 2.1** Formatos de datos admitidos por los algoritmos integrados de ML en Amazon SageMaker.

| Algoritmo | Formatos de datos aceptados |
| --- | --- |
| BlazingText | Archivo de texto (una oración por línea) |
| DeepAR forecasting | JSON Lines, Parquet |
| Factorization Machines | RecordIO-protobuf, CSV |
| Image classification | RecordIO, archivos de imagen (.jpg, .png) |
| IP Insights | CSV |
| K-means | RecordIO-protobuf, CSV |
| K-nearest neighbors (k-NN) | RecordIO-protobuf, CSV |
| Linear learner | RecordIO-protobuf, CSV |
| LDA | RecordIO-protobuf, CSV |
| Neural topic model | RecordIO-protobuf, CSV |
| Object detection | RecordIO, archivos de imagen (.jpg, .png) |
| PCA | RecordIO-protobuf, CSV |
| Random cut forest | RecordIO-protobuf, CSV |
| Semantic segmentation | RecordIO, archivos de imagen (.jpg, .png) |
| Seq2Seq | RecordIO-protobuf, archivo de texto |
| XGBoost | CSV, LibSVM, Parquet |

Dos formatos de la tabla no aparecieron antes. **JSON Lines** es un archivo de texto con un objeto JSON completo por línea, lo que permite procesarlo línea a línea sin cargar todo el archivo. **LibSVM** es un formato de texto para datos dispersos (*sparse*), es decir, con la mayoría de los valores en cero. Cada línea es `etiqueta índice:valor índice:valor ...` y solo lista las características distintas de cero.

> [!warning] Nota de precisión: la tabla no es exhaustiva
> - La tabla no incluye todos los algoritmos integrados actuales de SageMaker. Consulta la lista vigente en la documentación.
> - La documentación vigente de XGBoost en SageMaker menciona también el formato protobuf (RecordIO-protobuf) entre sus entradas posibles, y aclara que el entrenamiento distribuido con Dask solo admite CSV y Parquet. Confirma la lista completa de formatos en la página del algoritmo antes de decidir.

Otro factor importante es tu **patrón de acceso a los datos** (*data access pattern*). Un patrón de acceso a los datos define cómo interactúan los productores y los consumidores con los datos para satisfacer las necesidades del negocio. Implica entender y documentar cómo se consultan, se guardan y se recuperan los datos. Estos son los principales factores que definen tu patrón de acceso:

- **Tamaño de los datos.** Conocer el volumen de datos ayuda a determinar un particionamiento eficaz. **Particionar** es dividir los datos en trozos según una clave para que cada consulta lea solo el trozo que necesita. En S3, lo habitual es organizar los archivos por prefijos como `anio=2024/mes=05/`, de modo que una consulta de mayo lea solo esa «carpeta».
- **Forma de los datos.** Organizar los datos según los requisitos de las consultas puede mejorar la velocidad y la escalabilidad.
- **Velocidad de los datos.** Entender las cargas pico de consultas ayuda a optimizar el particionamiento para lograr una mejor capacidad de E/S. **E/S** (entrada/salida, en inglés *I/O*) son las operaciones de lectura y escritura sobre el almacenamiento. La *capacidad de E/S* es cuántas de esas operaciones puede atender el sistema por segundo, y si todas las consultas del pico caen en la misma partición, esa partición se satura aunque el resto esté ociosa.

La Tabla 2.2 ilustra un ejemplo de lo que debes buscar en un patrón de acceso a los datos.

**Tabla 2.2** Ejemplo de un patrón de acceso a los datos.

| Campo | Ejemplo |
| --- | --- |
| Nombre del patrón de acceso | *Find orders* (buscar pedidos) |
| Descripción del patrón de acceso | Buscar pedidos por ID de cliente e intervalo de tiempo |
| Prioridad | Media |
| Operación (lectura/escritura) | Lectura |
| Tipo (un elemento / varios elementos / todos) | Varios |
| Filtro | ID de cliente = 123, intervalo de tiempo = últimas 24 horas |
| Orden | Por tiempo, descendente |

En esencia, los patrones de acceso a los datos ayudan a diseñar soluciones de ingesta eficientes y escalables, porque alinean la organización de los datos con los requisitos de acceso.

Dicho de otro modo, la forma en que tus datos crudos se consumen desde distintas fuentes y se ingieren y guardan en AWS te ayudará a determinar el formato de datos y el servicio de AWS que debes usar.

La Tabla 2.3 muestra cómo se agrupan los servicios de AWS según si tus datos son estructurados, semiestructurados o no estructurados.

**Tabla 2.3** Servicios de AWS para datos estructurados, semiestructurados y no estructurados.

| Estructurados | Semiestructurados | No estructurados |
| --- | --- | --- |
| Amazon RDS | Amazon DynamoDB | Amazon S3 |
| Amazon Aurora | Amazon DocumentDB | Amazon Rekognition |
| Amazon Redshift | Amazon Athena | Amazon Transcribe |
| Amazon S3 | Amazon S3 | Amazon Comprehend |
| Amazon Athena | | |

La mayoría de estos servicios aparecen aquí por primera vez; RDS, DynamoDB, S3 y Athena se desarrollan más adelante en el capítulo:

- **Amazon RDS** y **Amazon Aurora** son bases de datos relacionales (SQL) administradas por AWS. Aurora es el motor propio de AWS, compatible con MySQL y PostgreSQL.
- **Amazon Redshift** es el *data warehouse* de AWS, una base de datos relacional optimizada para consultas analíticas sobre volúmenes grandes.
- **Amazon DynamoDB** es una base de datos NoSQL de clave-valor y documentos. **Amazon DocumentDB** es una base de datos de documentos JSON compatible con MongoDB.
- **Amazon Athena** es un motor de consultas SQL sobre archivos guardados en S3.
- **Amazon Rekognition** (visión por computadora), **Amazon Transcribe** (voz a texto) y **Amazon Comprehend** (procesamiento de lenguaje natural) son servicios de IA ya entrenados que se usan mediante una API.

> [!warning] Nota de precisión: la tabla mezcla almacenamiento con procesamiento
> Rekognition, Transcribe y Comprehend no **almacenan** datos no estructurados: los **procesan** y extraen de ellos información estructurada o semiestructurada (etiquetas de una imagen, el texto de un audio, las entidades de un documento). Athena tampoco almacena nada; consulta datos que viven en S3. Para una pregunta de examen sobre dónde **guardar** datos no estructurados, la respuesta de esta tabla es S3.

Como habrás notado, Amazon S3 es el servicio más flexible, porque sirve para las tres categorías de datos. Además, puedes complementar Amazon S3 con Amazon Athena y consultar tus datos directamente en S3 sin darles formato ni administrar la infraestructura. Aquí, «sin darles formato» significa que no hace falta cargar los datos en una base de datos antes de consultarlos. Basta con declarar un esquema de tabla que apunta a los archivos (el esquema se aplica al leer, lo que se conoce como *schema-on-read*), y esa definición se guarda en el catálogo de datos de AWS Glue, que verás en la sección de Glue.

En las próximas secciones profundizaremos en estos servicios en lo que respecta a la ingesta y el almacenamiento.

## Elegir servicios de ingesta de AWS (*Choosing AWS Ingestion Services*)

En servicios de ingesta, AWS ofrece un amplio abanico de opciones. El servicio que mejor se ajusta a tu caso de uso depende de varios factores:

- **Escalabilidad.** Tu solución de ingesta debe poder soportar la velocidad y el volumen de tus datos a medida que llegan de todas sus fuentes.
- **Resiliencia.** Tu solución de ingesta debe poder recuperarse de fallas y reanudar sin problemas desde el punto donde ocurrió la falla. Esto supone que el sistema registra hasta dónde procesó (una marca de posición que se suele llamar *checkpoint* u *offset*), de modo que al reanudar no pierda datos ni los procese dos veces.
- **Seguridad y cumplimiento normativo.** Los datos deben protegerse durante la ingesta de modo que ningún actor no autorizado pueda consumirlos nunca, ni en tránsito ni almacenados. Además, el proceso de ingesta debe cumplir las regulaciones de cada industria, como el Payment Card Industry Data Security Standard (**PCI DSS**) o la Health Insurance Portability and Accountability Act (**HIPAA**). PCI DSS es el estándar de seguridad que exige la industria de tarjetas de pago a cualquier empresa que guarde o procese datos de tarjetas. HIPAA es la ley estadounidense que protege la información de salud de los pacientes. También deben examinarse los requisitos de **residencia de datos** para asegurar que los datos se ingieren y almacenan conforme a ellos. La residencia de datos es la obligación legal o contractual de que ciertos datos permanezcan físicamente en un país o región. En AWS se cumple, en primer lugar, eligiendo la región donde se crean los recursos, y revisando que ningún servicio copie los datos a otra región sin que lo sepas.
- **Costo.** Con el modelo de pago por uso (*pay-as-you-go*) de la nube, siempre conviene mantener el costo bajo control, y más aún con datos de streaming, que prácticamente nunca se detienen. Por eso, los costos de ingerir y almacenar datos para entrenar tus modelos de ML pueden crecer rápidamente. Tú, como ingeniero de ML, necesitas elegir un servicio de ingesta de AWS cuyo modelo de precios sea el más económico para tu caso de uso. La razón de fondo es que un servicio de streaming funciona las 24 horas: si se cobra por hora de capacidad reservada, pagas unas 730 horas al mes aunque el tráfico sea bajo, mientras que si se cobra por gigabyte procesado, pagas en proporción al tráfico real.
- **Flexibilidad.** Tu solución de ingesta debe poder adaptarse a los cambios. Los servicios de AWS son altamente personalizables. Adapta tus *pipelines* de ingesta a tus requisitos técnicos y de negocio, pero contempla también el cambio como un elemento más de la arquitectura de tu solución. Un **pipeline** es una cadena automatizada de pasos por la que pasan los datos (leer, validar, transformar, guardar) sin intervención manual.

Empecemos con los servicios de ingesta de AWS para datos de streaming.

### Amazon Data Firehose

Amazon Data Firehose (antes llamado Amazon Kinesis Data Firehose) es un servicio **totalmente administrado** (*fully managed*) que permite recolectar, transformar y entregar flujos de datos a data lakes, data warehouses y servicios de analítica casi en tiempo real (en segundos). *Totalmente administrado* significa que AWS opera todo lo que hay debajo (servidores, sistema operativo, parches de seguridad, escalado y recuperación ante fallas) y tú solo configuras el servicio y pagas por usarlo. Es la diferencia entre usar un notebook alojado y montar tú mismo el servidor de Jupyter. Importa porque un equipo sin especialistas en infraestructura puede operar un pipeline de producción; a cambio, se renuncia a parte del control sobre la configuración.

Como servicio totalmente administrado, Amazon Data Firehose procesa el flujo continuamente, escala automáticamente según el volumen de datos que llegan y los entrega a su destino en segundos.

Para usar Amazon Data Firehose, configuras un flujo de datos con un origen, un destino y las transformaciones que tus datos necesitan antes de llegar al destino. En la documentación actual, este recurso se llama *Firehose stream* (antes, *delivery stream*).

Debes seleccionar el origen de tu flujo, como un *topic* de Amazon Managed Streaming for Kafka (MSK) o un *stream* de Kinesis Data Streams (ambos se explican en las secciones siguientes), o puedes escribir datos directamente con la API (interfaz de programación de aplicaciones) Firehose Direct PUT. Con *Direct PUT*, tu propia aplicación envía los registros a Firehose con una llamada a la API, sin un servicio de streaming intermedio. Amazon Data Firehose está integrado con más de 20 servicios de AWS, así que puedes configurar un flujo desde orígenes como estos:

- **Amazon CloudWatch Logs**, el servicio donde las aplicaciones y los servicios de AWS escriben sus registros (*logs*).
- Los registros de las *web ACL* de **AWS Web Application Firewall (WAF)**. WAF es un firewall que filtra el tráfico web (HTTP) según reglas, y una *web ACL* es la lista de reglas que aplica. Sus registros anotan cada petición permitida o bloqueada.
- Los registros de **AWS Network Firewall**, un firewall que filtra el tráfico a nivel de red dentro de tu red privada en AWS.
- **Amazon Simple Notification Service (SNS)**, un servicio de mensajería de publicación y suscripción: un mensaje publicado en un tema llega a todos sus suscriptores.
- **AWS IoT**, el servicio que conecta dispositivos IoT con la nube.

Debes seleccionar un destino para tu flujo, como Amazon S3, Amazon OpenSearch Service, Amazon Redshift, Splunk, Snowflake o un endpoint HTTP personalizado. **Amazon OpenSearch Service** es un motor de búsqueda y análisis de logs. **Splunk** es una plataforma comercial de análisis de logs y seguridad. **Snowflake** es un data warehouse en la nube de otra empresa. Un **endpoint HTTP** personalizado es la dirección (URL) de un servicio propio que recibe los datos mediante peticiones HTTP. Según la documentación vigente, la lista de destinos creció desde la edición del libro e incluye, entre otros, tablas de **Apache Iceberg** (un formato de tablas abiertas sobre archivos en S3) y varias plataformas de monitoreo de terceros.

Opcionalmente, puedes indicar si quieres convertir tu flujo de datos a un formato como Parquet u ORC, descomprimir los datos, aplicar transformaciones personalizadas con tu propia función de AWS Lambda o particionar dinámicamente los registros de entrada según sus atributos para entregarlos en ubicaciones distintas.

- **AWS Lambda** es el servicio de funciones *serverless* de AWS: subes una función (por ejemplo, en Python) y AWS la ejecuta cada vez que llega un evento, cobrando por milisegundo de ejecución, sin que haya un servidor a tu cargo. *Serverless* (sin servidor) no significa que no haya servidores, sino que no los ves ni los administras.
- La **conversión de formato** de Firehose espera registros de entrada en JSON y toma el esquema de destino de una tabla del catálogo de datos de AWS Glue.
- El **particionamiento dinámico** usa un campo de cada registro para decidir la carpeta de destino. Por ejemplo, con el campo `cliente_id`, cada registro se escribe bajo el prefijo `cliente_id=123/` de S3, lo que después abarata las consultas que filtran por cliente.
- La **descompresión** existe porque algunos orígenes, como CloudWatch Logs, entregan los datos comprimidos con GZIP.

> [!warning] Nota de precisión: «en segundos» depende del búfer
> Firehose no entrega cada registro al instante. Acumula los datos en un **búfer** hasta alcanzar un tamaño o un tiempo máximo, lo que ocurra primero, y entonces escribe. Para el destino S3, los valores por defecto son **5 MB o 300 segundos** (5 minutos), y el intervalo se puede configurar entre 0 y 900 segundos. Con un intervalo de cero, la documentación indica que entrega «en unos pocos segundos». Intervalos más cortos producen más archivos pequeños y más peticiones a S3, lo que sube el costo. Para el examen, la regla práctica es que Firehose es *casi* tiempo real y Kinesis Data Streams es tiempo real (menos de un segundo). Verifica los valores en la documentación vigente.

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos:

- **Streaming hacia data lakes y data warehouses.** Puedes enviar datos en streaming a Amazon S3, Amazon Redshift y otros destinos, convirtiéndolos a formatos como Parquet para su análisis, sin construir pipelines de procesamiento complejos. Sin Firehose, tendrías que escribir y mantener un programa que lea el flujo, agrupe los registros, los convierta, escriba los archivos y reintente cuando algo falle.
- **Observabilidad de seguridad.** Puedes monitorear la seguridad de la red en tiempo real y crear alertas cuando surjan amenazas potenciales, usando herramientas compatibles de gestión de información y eventos de seguridad (**SIEM**, *security information and event management*). Un SIEM es una plataforma que centraliza los registros de seguridad de muchas fuentes y los correlaciona para detectar ataques, como Splunk. *Observabilidad* es la capacidad de entender qué pasa dentro de un sistema a partir de lo que emite: logs, métricas y trazas.
- **Aplicaciones de machine learning.** Puedes enriquecer los flujos de datos con modelos de ML que analicen los datos y predigan resultados mientras los datos viajan a su destino. El libro no dice cómo se hace. El mecanismo habitual es la transformación con Lambda: la función llama al modelo (por ejemplo, un endpoint de SageMaker) y agrega la predicción a cada registro antes de la entrega.

### Amazon Kinesis Data Streams

Amazon Kinesis Data Streams es otro servicio administrado para ingerir y almacenar flujos de datos para su procesamiento. Su valor añadido es que hace streaming de datos en tiempo real y se integra ampliamente con el ecosistema de servicios de ingeniería de datos de AWS. La diferencia de fondo con Firehose es que Kinesis Data Streams **almacena** los registros durante un periodo de retención (24 horas por defecto, ampliable hasta 365 días; verifícalo en la documentación vigente), y cualquier número de aplicaciones puede leerlos, cada una a su ritmo, e incluso releerlos. Firehose, en cambio, es un conducto de entrega: recibe, transforma y deposita los datos en un destino, pero ninguna aplicación puede leer de él.

Kinesis Data Streams te permite construir aplicaciones personalizadas en tiempo real con Amazon Managed Service for Apache Flink o con otros frameworks populares, como **Apache Spark**, el motor de procesamiento distribuido más usado en big data, que tiene un módulo de streaming. También puedes enviar tus datos en streaming directamente a aplicaciones consumidoras que corren en instancias de Amazon EC2. Además, AWS Lambda puede actuar como consumidor para procesar los datos casi en tiempo real sin administrar servidores.

Con Kinesis Data Streams, tus datos se colocan en *data streams* de Kinesis, que garantizan durabilidad y elasticidad. La durabilidad viene de que AWS replica los datos en varias **zonas de disponibilidad** (*Availability Zones*, AZ). Una zona de disponibilidad es uno o varios centros de datos con energía, red y refrigeración independientes, separados físicamente de las demás zonas de la misma región. Una región tiene varias, así que un incendio o un corte eléctrico en una zona no afecta a las copias guardadas en otra. La elasticidad se basa en los **shards**: un stream se divide en shards, que son sus unidades de capacidad. Según la documentación, cada shard acepta hasta 1 MB/s o 1000 registros por segundo de escritura. Cada registro se asigna a un shard según una **clave de partición** que eliges tú, como el ID del dispositivo. Más shards significan más capacidad.

El retraso entre el momento en que un registro se coloca en el stream y el momento en que puede consumirse (retraso *put-to-get*) suele ser menor a un segundo. Dicho de otro modo, una aplicación de Kinesis Data Streams puede empezar a consumir los datos del stream casi inmediatamente después de que se agregan. Como Kinesis Data Streams es un servicio administrado, no tienes que preocuparte por crear y operar un pipeline de entrada de datos.

La elasticidad de Kinesis Data Streams te permite escalar automáticamente el stream hacia arriba o hacia abajo para que nunca pierdas registros antes de que expiren. Los registros *expiran* cuando se cumple el periodo de retención.

> [!warning] Nota de precisión: el escalado automático depende del modo de capacidad
> - Kinesis Data Streams tiene dos familias de modos. En el modo **on-demand** (con sus variantes *Standard* y *Advantage*), AWS administra los shards y escala solo. Según la documentación, un stream on-demand admite hasta el doble del pico de escritura de los últimos 30 días, y si el tráfico supera ese doble en menos de 15 minutos puede haber rechazos temporales que el productor debe reintentar. En el modo **provisioned**, tú fijas el número de shards y lo cambias cuando haga falta; no escala solo.
> - El escalado protege la **escritura**: evita que se rechacen registros por falta de capacidad. No protege contra otra forma de pérdida: si un consumidor se atrasa más que el periodo de retención, los registros expiran sin que nadie los haya leído.

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de Amazon Kinesis Data Streams:

- **Ingesta y procesamiento acelerados de logs y fuentes de datos.** Los **productores** (los programas que escriben en el stream; los que leen son los **consumidores**) pueden enviar los datos directamente al stream sin que te preocupe perderlos si falla el servidor de aplicaciones. Kinesis Data Streams acelera la entrada de datos porque no agrupas los datos en lotes en los servidores antes de enviarlos. El razonamiento implícito es este: en el esquema tradicional, la aplicación escribe sus logs en un archivo local y un proceso los sube cada hora. Si el servidor se cae antes de la subida, esa hora de logs se pierde. Si cada evento sale del servidor en cuanto se produce, ya está guardado de forma durable fuera de él.
- **Métricas y reportes en tiempo real.** Puedes usar los datos recolectados en Kinesis Data Streams para análisis y reportes sencillos en tiempo real. Por ejemplo, tu aplicación de procesamiento puede calcular métricas y reportes sobre los logs del sistema y de las aplicaciones a medida que llegan, en lugar de esperar a recibir lotes de datos.
- **Analítica de datos en tiempo real.** Combina la potencia del procesamiento en paralelo con el valor de los datos en tiempo real. Por ejemplo, puedes procesar en tiempo real el ***clickstream*** de un sitio web (la secuencia de clics y páginas vistas de cada visitante) y luego analizar la usabilidad y el nivel de interacción del sitio con varias aplicaciones de Kinesis Data Streams que corren en paralelo. Cada aplicación lee el mismo stream de forma independiente, así que una puede calcular embudos de conversión mientras otra detecta errores de navegación.
- **Procesamiento complejo de flujos.** Puedes crear aplicaciones sofisticadas que combinan varios data streams en nuevos data streams para su procesamiento posterior (*downstream*, es decir, en las etapas siguientes del pipeline).

### Amazon Managed Streaming for Apache Kafka (MSK)

Si conoces la plataforma de streaming de eventos Apache Kafka, la buena noticia es que puedes construir y ejecutar en AWS aplicaciones que usan Apache Kafka con poco esfuerzo y una administración mínima. Si no la conoces, estos son los conceptos que necesitas para esta sección:

- **Apache Kafka** es una plataforma distribuida de código abierto, creada en LinkedIn, para almacenar y transmitir flujos de eventos. Es el estándar de facto de la industria para streaming.
- Los eventos se guardan en **topics**: registros con nombre (por ejemplo, `pedidos`), ordenados y persistentes, a los que solo se agregan eventos al final. Un topic es a los eventos lo que una tabla es a las filas.
- Cada topic se divide en **particiones** para repartir la carga: cada partición es un registro ordenado independiente que puede vivir en un servidor distinto. Es la misma idea que los shards de Kinesis.
- Los **productores** escriben eventos en un topic. Los **consumidores** los leen y llevan la cuenta de su posición (el *offset*). Pueden volver a una posición anterior y releer el historial (*replay*) mientras los eventos sigan retenidos. Por defecto, Apache Kafka retiene los eventos 7 días, y es configurable.
- Alrededor de Kafka existe un ecosistema enorme de herramientas que hablan su protocolo, como los conectores de *Kafka Connect* o las bibliotecas de procesamiento *Kafka Streams*. Por eso la compatibilidad con Kafka suele pesar tanto en la elección de un servicio.

AWS ofrece **Amazon MSK**, que te permite ingerir y procesar datos de streaming en tiempo real de forma segura con un servicio de Apache Kafka totalmente administrado y de **alta disponibilidad**. Alta disponibilidad significa que el servicio sigue funcionando aunque falle un servidor o una zona de disponibilidad completa, porque mantiene copias y servidores de reserva en otras zonas.

Amazon MSK proporciona el **plano de control** (*control plane*) para crear, actualizar y eliminar **clústeres** de Apache Kafka. Un clúster es un grupo de servidores que funcionan juntos como un solo sistema. El plano de control agrupa las operaciones que administran la infraestructura, como crear el clúster o cambiar cuántos servidores tiene, y se usan a través de la API de AWS. El **plano de datos** (*data plane*) agrupa las operaciones sobre los datos, como escribir y leer eventos, y se usan con el protocolo de Kafka, conectándose directamente a los servidores del clúster. En una base de datos, la diferencia sería la que hay entre crear la base o agregarle réplicas (control) y ejecutar `SELECT` o `INSERT` (datos).

Para casos de uso de streaming bajo demanda y sin operación (*zero-operations*), puedes elegir **Amazon MSK Serverless**. MSK Serverless aprovisiona y escala la capacidad automáticamente mientras administra las particiones de tu topic, así que puedes hacer streaming de datos sin preocuparte por dimensionar (*right-sizing*) ni escalar clústeres. *Right-sizing* es elegir el número y el tamaño correctos de servidores: si te quedas corto, el clúster se satura; si te pasas, pagas capacidad ociosa. Con Amazon MSK Serverless, solo pagas por lo que usas.

> [!warning] Nota de precisión: el costo de MSK Serverless
> «Solo pagas por lo que usas» es una simplificación. Hasta donde sé, el precio de MSK Serverless combina un cargo por hora por clúster, un cargo por hora por partición, el almacenamiento y los datos que entran y salen. Un clúster serverless tiene, por tanto, un costo base aunque no reciba tráfico. Verifica la estructura de precios vigente en la página de precios de Amazon MSK.

Amazon MSK también te permite usar las operaciones del plano de datos de Apache Kafka, como las de producir y consumir datos.

Además, Amazon MSK ejecuta versiones de código abierto de Apache Kafka. Por eso admite, sin cambios en el código de la aplicación, las aplicaciones, herramientas y *plugins* que ya existen, tanto de socios comerciales como de la comunidad de Apache Kafka. La razón es que MSK habla exactamente el mismo protocolo que un Kafka instalado a mano. Lo que sí cambia al migrar es la configuración: la dirección del clúster y el método de autenticación.

El examen no espera que domines todos los aspectos de Amazon MSK. Aun así, tendrás que conocer los componentes principales de la arquitectura de Amazon MSK y su propósito:

- **Nodos broker (*broker nodes*).** Un **nodo** es cada servidor (máquina virtual) del clúster, y un **broker** es un servidor de Kafka. Los brokers son los nodos de trabajo que realizan las tareas «pesadas»: ingerir, almacenar (en las particiones de los topics) y procesar tus datos de streaming. Al crear un clúster de Amazon MSK, indicas cuántos nodos broker quieres que Amazon MSK cree en cada zona de disponibilidad. Algunos de estos brokers se eligen como **nodos controladores** (*controller nodes*). Estos se encargan de administrar el estado de las particiones y sus réplicas, de tareas administrativas como reasignar particiones y de mantener la relación **líder-seguidor** entre brokers para cada partición. Esa relación funciona así: cada partición se copia en varios brokers (normalmente tres), y esas copias son sus **réplicas**. Una réplica es la **líder**, que recibe todas las escrituras; las demás son **seguidoras** y copian a la líder. Si el broker de la líder falla, una seguidora pasa a ser la nueva líder y los datos no se pierden. El mínimo es un nodo broker por zona de disponibilidad. Cada zona de disponibilidad tiene su propia **subred** de la **nube privada virtual** (VPC). Una **VPC** es tu red privada y aislada dentro de AWS, como la red interna de una empresa. Una **subred** es un rango de direcciones de esa red que vive en una sola zona de disponibilidad. Por eso, para repartir los brokers en tres zonas, necesitas tres subredes.
- **Nodos ZooKeeper.** Amazon MSK también crea por ti los nodos de Apache ZooKeeper. Apache ZooKeeper es un servidor de código abierto que permite una coordinación distribuida muy confiable entre los nodos broker. *Coordinación distribuida* es lograr que varias máquinas se pongan de acuerdo en hechos compartidos (qué brokers están vivos, cuál es el líder de cada partición, qué configuración rige) aunque alguna falle o la red se corte a ratos.
- **Controladores KRaft.** La comunidad de Apache Kafka desarrolló KRaft para reemplazar a Apache ZooKeeper en la gestión de los metadatos de los clústeres de Apache Kafka. Los **metadatos** del clúster son los datos sobre los datos: qué topics existen, cuántas particiones tiene cada uno, qué broker lidera cada partición y qué permisos hay. En el modo KRaft, los metadatos del clúster se propagan dentro de un grupo de controladores de Kafka, que forman parte del propio clúster, en lugar de hacerlo entre nodos ZooKeeper. Los controladores KRaft se incluyen sin costo adicional y no requieren configuración ni administración de tu parte. El nombre viene de *Kafka* + *Raft*, el algoritmo de consenso que usan los controladores para ponerse de acuerdo.
- **Productores, consumidores y creadores de topics.** Amazon MSK te permite usar las operaciones del plano de datos de Apache Kafka para crear topics y para producir y consumir datos.
- **Operaciones del clúster.** Puedes usar la consola de administración de AWS (la interfaz web), la interfaz de línea de comandos de AWS (AWS CLI) o las API de los SDK para realizar operaciones del plano de control. Por ejemplo, puedes crear o eliminar un clúster de Amazon MSK, listar todos los clústeres de una cuenta, ver las propiedades de un clúster y actualizar el número y el tipo de brokers de un clúster.

> [!warning] Nota de precisión: ZooKeeper, KRaft y zonas de disponibilidad
> - Amazon MSK admite el modo KRaft desde la versión 3.7.x de Kafka, y los clústeres existentes con ZooKeeper se pueden migrar a KRaft. Además, Apache Kafka 4.0 (marzo de 2025) eliminó por completo el soporte de ZooKeeper. En clústeres nuevos con versiones recientes, lo normal es KRaft. Consulta en la documentación de MSK qué versiones de Kafka admite hoy.
> - Aunque el mínimo es un broker por zona, un clúster aprovisionado de MSK se reparte en **dos o tres zonas de disponibilidad**, según el tipo de broker y la región. Los brokers de tipo *Express* exigen tres. Es decir, el mínimo real de un clúster es de dos o tres brokers, no uno.

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de Amazon MSK:

- **Ingerir, almacenar, procesar y entregar flujos de eventos en tiempo real.** Amazon MSK te permite capturar y procesar en tiempo real grandes volúmenes de eventos de aplicaciones y de bases de datos, e ingerirlos continuamente en un data lake o en otros destinos compatibles para su procesamiento posterior. Los «eventos de bases de datos» suelen obtenerse con **captura de cambios de datos** (*change data capture*, CDC), una técnica que publica como evento cada `INSERT`, `UPDATE` o `DELETE` de una base de datos, sin modificar la aplicación que la usa.
- **Sistema de registro para datos de streaming.** Amazon MSK puede actuar como fuente única de verdad del estado de tus datos. Esto significa que todos los cambios de los datos se capturan y se guardan en topics durables de Amazon MSK, lo que permite que distintas aplicaciones accedan al estado y lo actualicen de forma consistente. Un **sistema de registro** (*system of record*) es la fuente autorizada de un dato: si dos sistemas discrepan, manda la suya. El razonamiento implícito es que, como el topic contiene cada cambio en orden, cualquier aplicación puede reconstruir el estado actual releyéndolo desde el principio, lo que exige una retención larga o indefinida.
- **Impulsar tus arquitecturas orientadas a eventos en AWS.** Puedes aprovechar la versatilidad de Apache Kafka para desarrollar aplicaciones modernas y seguras orientadas a eventos en AWS. En una **arquitectura orientada a eventos** (*event-driven*), los componentes no se llaman directamente unos a otros. Uno publica un hecho («se creó el pedido 881») y los interesados (facturación, inventario, un modelo de fraude) reaccionan cada uno por su cuenta. Así, los componentes quedan desacoplados: si facturación está caída, los eventos esperan en Kafka hasta que vuelva.

### Amazon Managed Service for Apache Flink

Con Amazon Managed Service for Apache Flink (antes llamado Amazon Kinesis Data Analytics), puedes usar el framework Apache Flink para procesar y analizar datos de streaming en tiempo real. Apache Flink es un framework de código abierto para el procesamiento de flujos que realiza **cómputos con estado** y analítica compleja. *Con estado* (*stateful*) significa que el cálculo recuerda información entre un evento y otro: el número de compras de cada tarjeta en los últimos 10 minutos, un promedio móvil o la sesión en curso de cada usuario. Para un estadístico, es como calcular un promedio móvil sobre una serie infinita sin poder guardar toda la historia. Flink guarda ese estado de forma periódica en un almacenamiento durable (lo que llama *checkpoints*), para que una falla no obligue a empezar de cero.

A diferencia de Apache Kafka, Apache Flink no tiene su propio sistema de almacenamiento de datos. Sin embargo, se integra por completo con varios sistemas de almacenamiento externos para manejar su estado y procesar datos, como Amazon S3, Amazon MSK, Amazon Kinesis Data Streams y otras fuentes. Dicho de otro modo, Kafka o Kinesis son el lugar donde **viven** los eventos y Flink es lo que **calcula** sobre ellos. Una arquitectura típica es Kinesis o MSK → Flink → S3 o una base de datos.

Entre las capacidades distintivas de Apache Flink están su sólido soporte para cargas de trabajo de streaming a gran escala, su tolerancia a fallas y sus fuertes garantías de corrección *exactly-once*.

- Una **carga de trabajo** (*workload*) es, en la jerga de la nube, cualquier aplicación o proceso que ejecutas con su patrón de uso de recursos, como «un job de entrenamiento que lee 2 TB por época» o «un pipeline de streaming que procesa 5000 eventos por segundo».
- La **tolerancia a fallas** es la capacidad de seguir produciendo resultados correctos aunque fallen máquinas durante el cálculo.
- ***Exactly-once*** (exactamente una vez) garantiza que cada evento afecta al resultado una sola vez, aunque una falla obligue a reprocesar. Las alternativas más débiles son *at-most-once* (como máximo una vez, que puede perder eventos) y *at-least-once* (al menos una vez, que puede duplicarlos). Importa cuando contar dos veces no es aceptable: facturación, saldos o conteos para detectar fraude. La garantía completa, de punta a punta, requiere además que el origen y el destino colaboren, por ejemplo, con escrituras transaccionales.

Estas capacidades resultan especialmente atractivas para las empresas globales, que necesitan una infraestructura confiable y de alto rendimiento para transferir grandes volúmenes de datos en forma de eventos en tiempo real.

Además, el servicio ofrece acceso a las API expresivas de Apache Flink, que incluyen tanto interfaces de programación en Java, Scala y Python como SQL. Con **Amazon Managed Service for Apache Flink Studio**, un entorno de notebooks (basado en Apache Zeppelin) conectado a los flujos en vivo, puedes consultar data streams de forma interactiva o lanzar aplicaciones con estado en pocos pasos. Con este servicio administrado, puedes empezar a usar Apache Flink y desplegar y operar rápidamente tus aplicaciones de procesamiento de flujos.

Igual que Apache MSK y la mayoría de los servicios totalmente administrados de AWS, no hay servidores ni clústeres que administrar ni infraestructura de cómputo que configurar. Solo pagas por los recursos que usas.

> [!warning] Nota de precisión
> - «Apache MSK» es una errata del original: el servicio es **Amazon** MSK.
> - La comparación es imprecisa para MSK: en un clúster **aprovisionado** de MSK sí eliges el número y el tipo de brokers (ver la sección anterior). Solo MSK Serverless encaja en «sin clústeres que administrar».
> - Hasta donde sé, Managed Service for Apache Flink factura por hora las unidades de procesamiento asignadas a la aplicación (KPU, cada una con 1 vCPU y 4 GB de memoria), así que «los recursos que usas» son los que asignas, aunque estén ociosos. Verifícalo en la página de precios vigente.

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de Amazon Managed Service for Apache Flink:

- **Pipelines de datos de streaming.** Ingerir, enriquecer y transformar flujos de datos continuamente, y cargarlos en sistemas de destino para actuar a tiempo (frente al procesamiento por lotes). Algunos ejemplos son la ingesta en un data lake, los pipelines de ML y el ETL (extraer, transformar, cargar) de streaming. El **ETL de streaming** aplica las transformaciones evento a evento, de forma continua, en lugar de hacerlo en una ejecución nocturna.
- **Analítica de flujos y por lotes.** Amazon Managed Service for Apache Flink admite tanto consultas por lotes tradicionales sobre **datasets acotados** (*bounded*), es decir, finitos, como un archivo con las ventas del mes pasado, como consultas continuas en tiempo real sobre **flujos no acotados** (*unbounded*), que no tienen final porque los eventos siguen llegando. Una *consulta continua* no termina: su resultado se actualiza con cada evento nuevo. Algunos ejemplos son la medición de consumo y la facturación (contar cuánto usa cada cliente para cobrarle), el monitoreo de redes, la ingeniería de características (calcular en vivo variables como «transacciones en la última hora» para un modelo en tiempo real; el capítulo 3 trata el tema) y el rendimiento de campañas.
- **Aplicaciones orientadas a eventos.** Puedes aprovechar las capacidades de Apache Flink para desarrollar y mantener fácilmente aplicaciones orientadas a eventos en AWS. Una aplicación orientada a eventos es una aplicación con estado que ingiere eventos de uno o más flujos y reacciona a ellos disparando cálculos, actualizaciones de estado o acciones externas. Algunos ejemplos son la detección de fraude, el monitoreo de procesos de negocio y el ***geo-fencing***. El geo-fencing define un perímetro virtual en un mapa y dispara un evento cuando la posición GPS de un dispositivo entra o sale de él, por ejemplo, para avisar que un camión llegó al almacén.

### AWS DataSync

AWS DataSync es un servicio de transferencia y descubrimiento de datos que simplifica la **migración de datos** y te ayuda a transferir de forma rápida, sencilla y segura tus datos de archivos u objetos hacia, desde y entre servicios de almacenamiento de AWS. Una migración de datos es el traslado, normalmente una sola vez y a gran escala, de los datos de un sistema de almacenamiento a otro, por ejemplo, del centro de datos de la empresa a la nube.

AWS DataSync logra el descubrimiento de datos usando un **agente de DataSync** que se conecta a la interfaz de administración de tu sistema de almacenamiento de origen. El agente es una máquina virtual que proporciona AWS y que tú instalas dentro de tu red (por ejemplo, en el mismo entorno de virtualización donde corren tus servidores). Hace falta porque AWS no puede entrar por su cuenta a tu red privada: el agente lee desde dentro y envía hacia AWS. La **interfaz de administración** es la API o consola con la que se configura y monitorea un equipo de almacenamiento empresarial. El agente recolecta información sobre tus recursos de almacenamiento, incluidas métricas de rendimiento y el uso de capacidad. Luego esos datos se envían a AWS DataSync Discovery, que los analiza y da recomendaciones para migrar tus datos a los servicios de almacenamiento de AWS. Este proceso automatizado te ayuda a entender el uso de tu almacenamiento y a planificar tu migración con más eficacia.

> [!warning] Estado del servicio (verificado el 23-09-2026)
> **AWS DataSync Discovery llegó al fin de su soporte el 20 de mayo de 2025** y ya no se puede usar. Todo lo que el libro dice sobre descubrimiento y recomendaciones de migración describe una función que ya no existe. La transferencia de datos con DataSync, que es el resto de esta sección, sigue vigente.

Con AWS DataSync también puedes transferir datos entre otros sistemas de almacenamiento en la nube o sistemas de almacenamiento *on-premises* y los servicios de AWS. **On-premises** (en las instalaciones) se refiere a la infraestructura que la empresa compra y opera en sus propios **centros de datos**, es decir, en salas o edificios con servidores, red, energía y refrigeración propios, en contraposición a la nube. En este contexto, los sistemas de almacenamiento pueden incluir los siguientes:

- Sistemas de almacenamiento **autoadministrados**, como un servidor de archivos NFS en tu VPC dentro de AWS. *Autoadministrado* (*self-managed*) significa que tú instalas y operas el software, por ejemplo, en instancias EC2, en lugar de usar un servicio administrado. Un **servidor de archivos** (*file server*) es una computadora cuyo trabajo es guardar archivos y compartirlos por la red con otras computadoras. NFS se explica en la lista siguiente.
- Sistemas o servicios de almacenamiento alojados en otro proveedor de nube.
- Sistemas o servicios de almacenamiento alojados on-premises, en los centros de datos de tu empresa.

AWS DataSync admite los siguientes sistemas de almacenamiento. Los tres primeros son **protocolos** o sistemas de archivos en red: las reglas con que una computadora le pide a otra, por la red, operaciones como «abre este archivo» o «lee sus bytes del 0 al 4095».

- **Network File System (NFS).** Es el protocolo estándar del mundo Linux y Unix para compartir archivos. Lo creó Sun Microsystems en los años 80. Con él, una computadora **monta** una carpeta de un servidor remoto, es decir, la conecta en un punto de su propio árbol de carpetas (por ejemplo, `/mnt/datos`), y a partir de ahí los programas la usan como si fuera un disco local. Importa porque la mayoría de los servidores Linux y de los clústeres de cómputo científico comparten datos por NFS. Sus versiones (v3, v4.x) se explican en la sección de Amazon EFS.
- **Server Message Block (SMB).** Es el equivalente de NFS en el mundo Windows: el protocolo de las carpetas compartidas de una red Windows, las rutas del tipo `\\servidor\carpeta` y las «unidades de red» con letra, como `S:`. Se integra con las cuentas de usuario y los permisos de Windows. También lo usan macOS y Linux. A una versión antigua de SMB se le llama CIFS.
- **Hadoop Distributed File System (HDFS).** Es el sistema de archivos distribuido del ecosistema **Hadoop**, la plataforma de código abierto de big data que precedió a Spark. HDFS divide cada archivo grande en bloques (de 128 MB por defecto) y los reparte entre las máquinas de un clúster, con tres copias de cada bloque. Muchas empresas tienen su data lake histórico en HDFS, on-premises, y lo migran a S3.
- **Almacenamiento de objetos** (Google Cloud Storage, Azure Blob Storage, Wasabi Cloud Storage y almacenamiento de objetos autoadministrado **compatible con la API de Amazon S3**). Que un producto sea compatible con la API de S3 significa que implementa las mismas operaciones HTTP que S3, con los mismos nombres y formatos, así que las herramientas escritas para S3 funcionan con él sin cambios.

AWS DataSync admite los siguientes servicios de almacenamiento de AWS:

- Amazon S3
- Amazon EFS
- Amazon FSx for Windows File Server
- Amazon FSx for Lustre
- Amazon FSx for OpenZFS
- Amazon FSx for NetApp ONTAP
- AWS Snowcone
- AWS Snowball Edge

**Amazon EFS** y la familia **Amazon FSx** son servicios de almacenamiento de archivos que se explican más adelante en este capítulo. Cada variante de FSx es un sistema de archivos administrado construido sobre una tecnología de terceros, y lo que va después de «for» indica cuál: Lustre, Windows File Server, NetApp ONTAP u OpenZFS. **AWS Snowcone** y **AWS Snowball Edge** son dispositivos físicos de la familia *Snow*. AWS te los envía, tú copias los datos localmente, los devuelves por mensajería y AWS los carga en S3. Tienen sentido cuando la red es demasiado lenta. Por ejemplo, 500 TB por una línea de 1 Gbps tardan unos 46 días aun a plena velocidad, porque 500 TB son 4 × 10¹⁵ bits y a 10⁹ bits por segundo eso da 4 × 10⁶ segundos. (Recuerda que Snowcone se descontinuó y Snowball Edge ya no admite clientes nuevos; ver el recuadro del inicio.)

Con DataSync puedes obtener los siguientes beneficios:

- **Simplificar la planificación de la migración.** Con la recolección automática de datos y las recomendaciones, DataSync Discovery puede minimizar el tiempo, el esfuerzo y los costos de planificar tus migraciones a AWS. Puedes usar las recomendaciones para tu presupuesto y volver a ejecutar los trabajos de descubrimiento para validar tus supuestos a medida que te acercas a la migración. (Función retirada; ver el recuadro anterior.)
- **Automatizar el movimiento de datos.** DataSync facilita transferir datos por la red entre sistemas y servicios de almacenamiento. Automatiza tanto la gestión de los procesos de transferencia como la infraestructura necesaria para una transferencia segura y de alto rendimiento. En la práctica, se encarga de programar las copias, reintentar lo que falla, verificar el resultado y, en ejecuciones sucesivas, copiar solo lo que cambió.
- **Transferir datos de forma segura.** DataSync ofrece seguridad de punta a punta, incluidos cifrado y **validación de integridad**, para que tus datos lleguen de forma segura, intactos y listos para usarse. La validación de integridad compara sumas de verificación (*checksums*), una especie de huella digital de los bytes, en el origen y en el destino para confirmar que la copia es idéntica. DataSync accede a tu almacenamiento de AWS mediante los mecanismos de seguridad integrados de AWS, como los **roles** de AWS Identity and Access Management (**IAM**). IAM es el servicio donde se define qué identidad puede hacer qué acción sobre qué recurso, y un rol es una identidad con permisos que un servicio asume temporalmente; aquí, el permiso de DataSync para escribir en tu bucket. También admite **endpoints de VPC**, con los que puedes transferir datos sin pasar por el internet público, lo que aumenta aún más la seguridad de los datos copiados en línea. Un endpoint de VPC es una puerta de entrada privada, dentro de tu red de AWS, hacia un servicio de AWS, de modo que el tráfico viaja por la red interna de AWS.
- **Mover datos más rápido.** DataSync usa un protocolo de red construido para este propósito y una arquitectura paralela y multihilo (*multithreaded*) para acelerar tus transferencias. En lugar de copiar un archivo tras otro con un protocolo genérico, abre muchas transferencias simultáneas y usa un protocolo optimizado para enlaces largos, donde cada ida y vuelta de la red tarda. Es como abrir muchas cajas en un supermercado en lugar de una. Esto acelera las migraciones, los flujos recurrentes de procesamiento de datos para analítica y ML, y los procesos de protección de datos (copias de seguridad y réplicas).
- **Reducir los costos operativos.** Mueve datos de forma económica con el precio fijo por gigabyte de DataSync. Evitas escribir y mantener scripts propios o usar costosas herramientas comerciales de transferencia. (El modelo de precios puede haber cambiado desde la edición del libro; verifícalo en la página de precios vigente.)

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de AWS DataSync:

- **Descubrir datos.** Obtén visibilidad sobre el rendimiento y el uso de tu almacenamiento on-premises. AWS DataSync Discovery también puede darte recomendaciones para migrar tus datos a los servicios de almacenamiento de AWS. (Función retirada en mayo de 2025.)
- **Migrar datos.** Transfiere rápidamente por la red *datasets activos* a los servicios de almacenamiento de AWS. Un dataset activo es uno que se sigue usando y modificando durante la migración, por eso importa poder repetir la copia y traer solo los cambios antes del corte final. DataSync incluye cifrado automático y validación de la integridad de los datos para que lleguen de forma segura, intactos y listos para usarse.
- **Archivar datos fríos.** Mueve los datos fríos (*cold data*, los que casi nunca se consultan) del almacenamiento on-premises directamente a clases de almacenamiento de largo plazo durables, de alta disponibilidad y seguras, como S3 Glacier Flexible Retrieval y S3 Glacier Deep Archive (ver la Tabla 2.4). Así puedes liberar capacidad de almacenamiento on-premises y apagar sistemas heredados (*legacy*, sistemas antiguos que siguen en uso aunque ya estén desfasados).
- **Replicar datos.** Copia datos en cualquier clase de almacenamiento de Amazon S3 y elige la más económica para tus necesidades. También puedes enviar datos a Amazon EFS, FSx for Windows File Server, FSx for Lustre o FSx for OpenZFS como sistema de archivos de reserva (*standby*), es decir, una copia lista para tomar el relevo si falla el sistema principal.
- **Transferir datos para procesarlos a tiempo en la nube.** Transfiere datos hacia AWS o desde AWS para procesarlos. Este enfoque puede acelerar flujos de trabajo críticos de **nube híbrida**, aquellos en que una parte corre on-premises y otra en la nube, en muchas industrias. Entre ellos están el ML en ciencias de la vida, la producción de video en medios y entretenimiento, la analítica de big data en servicios financieros y la investigación sísmica en la industria del petróleo y el gas.

### AWS Glue

El proceso de preparar tus datos para que estén listos para un algoritmo de ML es un paso clave del ciclo de vida de ML. AWS Glue es un servicio de ETL serverless, optimizado para la nube, que te permite descubrir, preparar, mover e integrar datos de múltiples fuentes de tu negocio para que estén listos para usarse.

AWS Glue se diferencia de otros servicios de ETL en cuatro aspectos importantes:

- **Serverless.** No necesitas aprovisionar, configurar ni levantar servidores, ni administrar su ciclo de vida.
- **Inferencia automática del esquema.** AWS Glue incluye **crawlers**, que analizan tus datasets, descubren los tipos de archivo, extraen el esquema y guardan todos estos metadatos en un catálogo centralizado para consultarlos y analizarlos después. Un crawler (rastreador) es un proceso que recorre, por ejemplo, un prefijo de S3, lee muestras de los archivos, deduce columnas y tipos, y registra el resultado como una tabla. Ese catálogo, el **AWS Glue Data Catalog**, es el mismo que usa Athena para saber qué columnas tiene cada tabla y dónde están sus archivos.
- **Generación automática de scripts de ETL.** AWS Glue genera automáticamente los scripts que necesitas para extraer, transformar y cargar tus datos desde el origen hasta el destino. Por ejemplo, diseñas el flujo en un editor visual y Glue produce el código PySpark equivalente.
- **Conectividad extensa.** AWS Glue ofrece una amplia gama de **conectores** para integrarse con distintas fuentes de datos. Incluye soporte integrado para almacenes de datos de uso común, como Amazon Redshift, Amazon Aurora, Microsoft SQL Server, MySQL, MongoDB, MariaDB y PostgreSQL. Además, AWS Glue permite usar controladores **JDBC** personalizados para integrarse con otras fuentes de datos. JDBC (*Java Database Connectivity*) es el estándar de Java para conectarse a bases de datos, y un controlador (*driver*) JDBC es la biblioteca que sabe hablar con un motor concreto. Cumple el mismo papel que un «dialecto» de SQLAlchemy en Python.

Con estos beneficios, AWS Glue hace que la preparación de datos sea simple, rápida, segura y económica. Tu equipo de ingenieros de ML y de datos puede usar AWS Glue para crear, ejecutar y monitorear visualmente pipelines de ETL que cargan datos en tus data lakes y preparan tus datos para seleccionar un algoritmo de ML.

Además, AWS Glue es expresivo y flexible, y permite desarrollar scripts de ETL personalizados con frameworks populares como Spark, PySpark, Scala y Ray (AWS Glue for Ray). Esto permite construir soluciones de procesamiento a la medida de las necesidades de tus proyectos de ML, con eficiencia y precisión en la preparación de los datos. La lista mezcla niveles distintos. **Spark** es el motor de procesamiento distribuido, y **PySpark** y **Scala** son dos lenguajes para programarlo: la API de Spark en Python y el lenguaje en que está escrito Spark. **Ray** es un framework de cómputo distribuido para Python, popular en ML. (AWS Glue for Ray ya no admite clientes nuevos; ver el recuadro del inicio.)

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de AWS Glue:

- **Desarrollo de pipelines de ETL complejos.** Gracias a su capacidad de escalado automático (*Auto Scaling*), AWS Glue es el servicio ideal para ejecutar trabajos de ETL con demandas de cómputo irregulares, una cantidad de datos impredecible y un gran número de fuentes de datos. Con el escalado automático, el trabajo agrega o quita *workers* (las máquinas que ejecutan el procesamiento) durante la ejecución según la carga, en lugar de mantener un número fijo todo el tiempo, y solo pagas por los que realmente se usaron.
- **Descubrimiento de datos.** Las capacidades nativas de AWS Glue facilitan identificar datos en AWS, on-premises y en otras nubes, y dejarlos disponibles de inmediato para consultarlos y transformarlos. Los crawlers y el catálogo son la pieza que lo hace posible; las fuentes on-premises se alcanzan mediante conexiones JDBC.
- **Soporte para frameworks de procesamiento de datos.** Con AWS Glue puedes conectarte a más de 70 fuentes de datos distintas, implementar varios tipos de cargas de trabajo (por lotes, por **microlotes** y streaming) y administrar tus datos en un catálogo centralizado. Un microlote (*micro-batch*) procesa un flujo en pequeños lotes cada pocos segundos o minutos, un término medio entre el lote nocturno y el evento a evento; así funciona el streaming de Spark.
- **Experiencia de ingeniería de datos simplificada.** Con las sesiones interactivas de AWS Glue, los ingenieros de datos y de ML pueden explorar y preparar datos de forma interactiva desde el **entorno de desarrollo integrado** (IDE, como VS Code o PyCharm) o el notebook que prefieran. Una sesión interactiva levanta bajo demanda un motor Spark serverless conectado a tu notebook y se cobra por el tiempo de uso.

## Elegir servicios de almacenamiento de AWS (*Choosing AWS Storage Services*)

Los datos con los que entrenarás tu modelo de ML tendrán que persistirse en su formato crudo en un servicio de almacenamiento adecuado. Esto es crucial, porque asegura que tus datos estén accesibles y en un estado óptimo para el preprocesamiento y el entrenamiento del modelo. Además, el almacenamiento es esencial no solo para los datos de entrenamiento, sino también para guardar los **artefactos** que resultan del entrenamiento, como los parámetros del modelo, los pesos y otros datos derivados. En SageMaker, el artefacto típico es un archivo comprimido (`model.tar.gz`) en S3 que contiene los pesos del modelo entrenado y que después se usa para desplegarlo. Estos artefactos son vitales para hacer predicciones precisas y te permiten desplegar y operar tus modelos de ML con eficacia. Una buena gestión del almacenamiento facilita las transiciones entre la ingesta de datos, el entrenamiento, la evaluación y el despliegue del modelo, y así da soporte a todo el ciclo de vida de ML.

Estos son los principales factores que debes considerar al seleccionar el servicio de almacenamiento adecuado:

- **Durabilidad.** Responde a la pregunta «¿durante cuánto tiempo necesitas conservar tus datos?». La durabilidad mide la capacidad de un servicio de almacenamiento de conservar los datos frente a los problemas de la operación normal a lo largo de su vida útil (discos que se estropean, errores de escritura). Se expresa como la probabilidad de no perder un dato en un año. S3, por ejemplo, está diseñado para una durabilidad del 99.999999999 % («once nueves»). AWS lo ilustra así: si guardas 10 millones de objetos, puedes esperar perder uno cada 10 000 años en promedio. Como contraste, la documentación de EBS indica para sus volúmenes de uso general una durabilidad del 99.8 % al 99.9 % anual, es decir, una tasa de falla anual del 0.1 % al 0.2 %. La durabilidad varía mucho entre servicios.
- **Disponibilidad.** Responde a la pregunta «¿qué tan pronto necesitas usar tus datos?». La disponibilidad mide el porcentaje del tiempo en que el servicio de almacenamiento puede usarse, entendiendo por «puede usarse» que cumple su función acordada cuando se le requiere. La disponibilidad (también llamada disponibilidad del servicio) es una métrica muy usada para medir cuantitativamente la confiabilidad. Para darle escala, un 99.99 % de disponibilidad (el diseño de S3 Standard) equivale a unos 53 minutos al año sin servicio, un 99.9 % a unas 8.8 horas y un 99.5 % a unas 44 horas.
- **Tipo de almacenamiento.** Responde a la pregunta «¿en qué formato necesitas tus datos para usarlos eficazmente en la preparación?». Los tipos de almacenamiento incluyen objetos, bloques, archivos y otros tipos específicos según el caso de uso, como bases de datos o streaming. Los tres primeros se explicaron junto a la Figura 2.3.
- **Costo.** Responde a la pregunta «¿cuánto dinero estás dispuesto a gastar para guardar tus datos?». El costo es uno de los pilares del **AWS Well-Architected Framework** y aplica a cualquier solución que diseñes, arquitectes y construyas en la nube. El Well-Architected Framework es la guía de buenas prácticas de AWS para diseñar arquitecturas en la nube, organizada en seis pilares: excelencia operativa, seguridad, confiabilidad, eficiencia del rendimiento, optimización de costos y sostenibilidad. En las próximas secciones veremos el factor costo de cada servicio de almacenamiento de AWS relevante para el ciclo de vida de ML.
- **Seguridad.** Responde a la pregunta «¿qué tipo de protección necesitan tus datos en reposo?». La seguridad es otro pilar del Well-Architected Framework y aplica a cualquier solución que diseñes, arquitectes y construyas en la nube. La sensibilidad de tus datos determina qué controles de protección se requieren mientras están almacenados, es decir, **en reposo** (a diferencia de *en tránsito*, cuando viajan por la red). El control básico es el **cifrado en reposo**: los datos se guardan cifrados, de modo que quien obtenga el disco o una copia del archivo no pueda leerlos sin la clave.

En las próximas secciones se cubren los principales servicios de almacenamiento de AWS para cargas de trabajo de ML. Empezaremos con los tres servicios de almacenamiento que se integran de forma nativa con Amazon SageMaker: Amazon S3, Amazon Elastic File System (EFS) y Amazon FSx for Lustre. Que la integración sea nativa significa que, al crear un job de entrenamiento, puedes declarar cualquiera de los tres como origen de datos de un **canal de entrada**, y SageMaker se encarga de copiar o montar los datos en las instancias de entrenamiento. Con cualquier otro almacenamiento, tu propio código tendría que ir a buscar los datos.

### Amazon Simple Storage Service (S3)

Amazon S3 es un servicio de almacenamiento de objetos que ofrece durabilidad, disponibilidad, escalabilidad, seguridad y rendimiento líderes en la industria.

El almacenamiento de objetos es un tipo de almacenamiento que persiste y gestiona los datos en un formato no estructurado llamado *objetos*. Aquí, «no estructurado» significa que S3 no interpreta el contenido: para S3, un archivo Parquet con una tabla perfectamente estructurada es una secuencia opaca de bytes. A medida que las organizaciones avanzan en su transformación digital, su capacidad para almacenar grandes cantidades de datos no estructurados, como fotos, videos, correos electrónicos, páginas web, datos de sensores y archivos de audio, se ha vuelto un aspecto clave de esa transformación.

Amazon S3 distribuye estos datos entre múltiples dispositivos físicos, pero permite que los usuarios accedan al contenido de forma eficiente a partir de un identificador único, el **URI** (*Uniform Resource Identifier*) del objeto, que tiene la forma `s3://mi-bucket/ventas/2024/05/dia-01.csv`.

Los principales beneficios del almacenamiento de objetos son una escalabilidad prácticamente ilimitada y un costo menor para guardar grandes cantidades de datos, en casos de uso como data lakes, aplicaciones **nativas de la nube** (*cloud native*, diseñadas desde el principio para aprovechar servicios administrados y escalar agregando máquinas), analítica, archivos de log y ML.

El almacenamiento de objetos también ofrece mayor durabilidad y resiliencia, porque guarda los objetos en múltiples dispositivos, en múltiples sistemas e incluso en múltiples zonas de disponibilidad y regiones. Esto permite una escala prácticamente ilimitada y mejora la resiliencia y la disponibilidad de los datos.

> [!warning] Nota de precisión: zonas sí, regiones solo si lo configuras
> Según la documentación de S3, las clases de almacenamiento estándar guardan los datos en **al menos tres zonas de disponibilidad de una misma región**. S3 **no** copia tus objetos a otra región por su cuenta; para eso hay que configurar la replicación entre regiones (*Cross-Region Replication*). Y las clases *One Zone* guardan los datos en una sola zona.

Gracias a su integración nativa con Amazon SageMaker, Amazon S3 es una de las opciones de almacenamiento más económicas y fáciles de usar para las operaciones de ML en AWS. Ofrece una variedad de clases de almacenamiento para distintos patrones de acceso y presupuestos, lo que lo convierte en una opción versátil para guardar tanto datos de entrenamiento como artefactos de modelos. Además, la integración de Amazon S3 con varios servicios de AWS, como AWS Lambda, Amazon Elastic Container Service (**ECS**) y Amazon Elastic Kubernetes Service (**EKS**), simplifica el proceso de construir, entrenar y desplegar modelos de ML. ECS y EKS ejecutan **contenedores**, aplicaciones empaquetadas junto con todas sus dependencias (el formato más común es Docker). EKS es la versión administrada de **Kubernetes**, el orquestador de contenedores estándar de la industria, que decide en qué máquina corre cada contenedor y lo reinicia si falla.

La Figura 2.4 ilustra un flujo de trabajo sencillo que muestra cómo Amazon S3 y Amazon SageMaker pueden interactuar durante el ciclo de vida de ML.

*Figura 2.4 Uso de Amazon S3 en el ciclo de vida de ML.*

Para el examen, asegúrate de entender bien las clases de almacenamiento de Amazon S3, que se describen en la Tabla 2.4. Una **clase de almacenamiento** es un nivel de precio y de acceso que se asigna a cada objeto: cuanto menos frecuente es el acceso esperado, más barato es guardar el dato y más caro (o más lento) es leerlo.

**Tabla 2.4** Clases de almacenamiento de Amazon S3.

| Clase de almacenamiento de S3 | Descripción |
| --- | --- |
| S3 Standard | Diseñada para datos a los que se accede con frecuencia |
| S3 Intelligent-Tiering | Optimiza los costos moviendo automáticamente los datos entre dos niveles de acceso según los cambios en los patrones de acceso |
| S3 Express One Zone | Clase de alto rendimiento en una sola zona de disponibilidad, que ofrece acceso a los datos con una latencia constante de un solo dígito de milisegundos para aplicaciones sensibles a la latencia |
| S3 Standard-IA (acceso poco frecuente) | Adecuada para datos a los que se accede con menos frecuencia pero que requieren acceso rápido cuando se necesitan, a un costo menor que S3 Standard |
| S3 One Zone-IA | Ideal para datos de acceso poco frecuente que no requieren la resiliencia de múltiples zonas de disponibilidad; ofrece costos de almacenamiento más bajos |
| S3 Glacier Instant Retrieval | Diseñada para datos de larga vida a los que se accede rara vez pero que requieren acceso inmediato; ofrece tiempos y costos de recuperación bajos |
| S3 Glacier Flexible Retrieval | Se usa para datos de archivo con tiempos de acceso flexibles, de minutos a horas; ofrece almacenamiento de bajo costo |
| S3 Glacier Deep Archive | Ofrece el almacenamiento de menor costo para archivar datos a largo plazo, con tiempos de recuperación de hasta 12 horas |
| S3 Outposts | Lleva el almacenamiento de S3 a tus entornos on-premises, lo que asegura un rendimiento constante y cumple los requisitos de residencia de datos |

Algunos términos de la tabla necesitan explicación:

- **Latencia** es el tiempo que pasa entre que pides un dato y empieza a llegar la respuesta. «Un solo dígito de milisegundos» son entre 1 y 9 ms. Como escala aproximada, leer de un SSD NVMe de laptop tarda del orden de 0.1 ms, un disco duro mecánico tarda de 5 a 10 ms (su cabezal tiene que moverse) y S3 Standard suele responder objetos pequeños en unos 100 a 200 ms, según las guías de rendimiento de S3. Importa en ML porque si un entrenamiento lee un millón de imágenes pequeñas una tras otra, 100 ms por lectura son casi 28 horas solo de espera, frente a menos de 3 horas con 10 ms. En la práctica se lee en paralelo, pero la latencia sigue marcando el ritmo.
- **Recuperación** (*retrieval*) es el paso previo a leer un objeto archivado. En las clases Glacier Flexible Retrieval y Deep Archive los objetos no se pueden leer directamente: primero se pide su restauración, que tarda de minutos a horas, y después se leen.
- **S3 on Outposts** usa **AWS Outposts**, bastidores de hardware de AWS que se instalan en tu propio centro de datos, AWS los administra y ejecutan servicios de AWS localmente.

> [!warning] Nota de precisión: la Tabla 2.4 está desactualizada o simplificada en dos filas
> - **S3 Intelligent-Tiering** no mueve los datos entre **dos** niveles, sino entre **tres niveles automáticos**: *Frequent Access*, *Infrequent Access* (tras 30 días sin acceso) y *Archive Instant Access* (tras 90 días). Además, tiene **dos niveles opcionales** de archivo, *Archive Access* y *Deep Archive Access*, que hay que activar y cuyos objetos deben restaurarse antes de leerlos. Los objetos de menos de 128 KB no se monitorean y se quedan siempre en el nivel frecuente.
> - En **S3 Glacier Deep Archive**, 12 horas es el plazo de la recuperación *estándar*. La recuperación masiva (*bulk*), más barata, puede tardar hasta 48 horas.
> - Verifica estos valores en la documentación vigente de S3.

La tabla siguiente **no está en el libro**. Resume datos de la documentación de S3 (consultada el 23-09-2026) que suelen aparecer en preguntas de examen:

| Clase | Disponibilidad de diseño | Zonas de disponibilidad | Duración mínima facturada | Cargo por recuperación |
| --- | --- | --- | --- | --- |
| S3 Standard | 99.99 % | ≥ 3 | Ninguna | No |
| S3 Intelligent-Tiering | 99.9 % | ≥ 3 | Ninguna | No (hay cargo de monitoreo por objeto) |
| S3 Express One Zone | 99.95 % | 1 | Ninguna | No |
| S3 Standard-IA | 99.9 % | ≥ 3 | 30 días | Sí |
| S3 One Zone-IA | 99.5 % | 1 | 30 días | Sí |
| S3 Glacier Instant Retrieval | 99.9 % | ≥ 3 | 90 días | Sí |
| S3 Glacier Flexible Retrieval | 99.99 % (tras restaurar) | ≥ 3 | 90 días | Sí |
| S3 Glacier Deep Archive | 99.99 % (tras restaurar) | ≥ 3 | 180 días | Sí |

Todas están diseñadas para una durabilidad del 99.999999999 %. La *duración mínima facturada* significa que, si borras un objeto antes de ese plazo, pagas igualmente los días restantes.

#### Amazon Athena

Amazon Athena es un servicio de consultas interactivas y serverless que te permite analizar datos directamente en Amazon S3 con SQL estándar. *Interactivo* quiere decir que las consultas devuelven resultados en segundos, lo bastante rápido para ir iterando como en un notebook. Athena lee las definiciones de las tablas (columnas y ubicación de los archivos) del catálogo de datos de AWS Glue y cobra según la cantidad de datos que escanea cada consulta. Por eso los formatos columnares y el particionamiento, explicados antes en este capítulo, abaratan directamente las consultas.

Con Amazon Athena, puedes ejecutar consultas SQL sobre datos guardados en S3, por ejemplo, en CSV, en JSON o en formatos columnares como Apache Parquet y Apache ORC.

#### Casos de uso (*Use Cases*)

Construido sobre una arquitectura resiliente diseñada específicamente para persistir grandes cantidades de datos no estructurados, Amazon S3 es el más adecuado para los casos de uso siguientes. El «más adecuado» se apoya en lo explicado antes: durabilidad de once nueves, crecimiento sin aprovisionar capacidad, el menor costo por gigabyte, acceso por API desde cualquier servicio y una integración nativa con casi todo AWS.

- **Data lakes y ML.** Un data lake es un repositorio centralizado que te permite almacenar todos tus datos en su formato crudo, a cualquier escala. Puedes usar un data lake para ejecutar procesamiento de datos, analítica, ML y aplicaciones de computación de alto rendimiento (**HPC**, *high-performance computing*) que extraigan el valor de tus datos. HPC es el uso de muchos servidores trabajando en paralelo sobre un mismo problema de cálculo, como simulaciones climáticas, genómica o dinámica de fluidos.
- **Copias de seguridad y restauración.** Con las sólidas capacidades de replicación y protección de datos de Amazon S3, puedes construir aplicaciones que cumplan tu **objetivo de tiempo de recuperación** (RTO), tu **objetivo de punto de recuperación** (RPO) y tus requisitos de cumplimiento normativo. El RTO es el tiempo máximo aceptable para volver a dar servicio después de un desastre («debemos estar operando en menos de 4 horas»). El RPO es la cantidad máxima de datos que se acepta perder, medida en tiempo («como mucho, la última hora de transacciones»).
- **Archivo de datos.** Puedes conservar tus datos «fríos» (datos de acceso poco frecuente que deben conservarse por cumplimiento normativo) en las clases de almacenamiento Amazon S3 Glacier para reducir costos, eliminar complejidades operativas y cumplir los requisitos de archivo de datos de tu organización.
- **Inteligencia artificial (IA) generativa.** En el momento de escribir el libro, Amazon S3 almacenaba exabytes de datos en más de 350 billones de objetos (350 *trillion* en inglés, es decir, 350 × 10¹²) y atendía un promedio de más de 100 millones de peticiones por segundo. Un exabyte son mil petabytes, o un millón de terabytes: el equivalente a un millón de discos de laptop de 1 TB. Con este nivel masivo de escalabilidad, Amazon S3 es el servicio de almacenamiento ideal para preparar y entrenar tus **modelos de lenguaje grandes** (**LLM**), las redes neuronales con miles de millones de parámetros que generan texto. Los LLM se entrenan para aprender relaciones estadísticas a partir de enormes cantidades de datos durante un proceso de entrenamiento autosupervisado y semisupervisado. En el aprendizaje **autosupervisado**, el modelo aprende de datos sin etiquetar prediciendo partes del propio dato, como la siguiente palabra de un texto. En el **semisupervisado**, se combinan pocos ejemplos etiquetados con muchos sin etiquetar. Veremos estos conceptos en los próximos capítulos.

> [!warning] Cifras actualizadas de S3
> En marzo de 2026, con motivo de los 20 años de S3, AWS informó que el servicio almacena **más de 500 billones de objetos**, en **cientos de exabytes**, y atiende **más de 200 millones de peticiones por segundo**. Las cifras del libro eran correctas en su momento, pero ya quedaron cortas.
>
> Hay además un matiz sobre el entrenamiento de LLM: S3 suele ser el **repositorio** de los datos, pero durante el entrenamiento es habitual interponer una capa más rápida entre S3 y las GPU, como FSx for Lustre (ver más abajo) o S3 Express One Zone, porque la latencia de S3 Standard puede dejar a las GPU esperando datos.

### Amazon Elastic File System (EFS)

Amazon EFS es un servicio de almacenamiento de AWS basado en archivos, serverless y totalmente elástico. Que sea serverless significa que no hace falta administrar, configurar ni crear la infraestructura subyacente, porque AWS hace esas tareas por ti. Que sea **totalmente elástico** significa que no eliges un tamaño: el sistema de archivos crece y se encoge solo a medida que agregas o borras archivos, y pagas por los gigabytes que realmente guardas.

Amazon EFS te permite crear y configurar **sistemas de archivos distribuidos** en AWS y montarlos en diversos recursos de cómputo de AWS, como instancias de Amazon EC2, clústeres de Amazon EKS, Amazon ECS, funciones de AWS Lambda y otros. Un sistema de archivos distribuido reparte sus datos entre muchos servidores (y, en EFS, entre varias zonas de disponibilidad), pero los presenta como un único árbol de carpetas. No requiere aprovisionamiento (reservar capacidad antes de usarla), despliegue, parches (instalar actualizaciones de software, sobre todo de seguridad) ni mantenimiento.

También admite los populares protocolos NFS versiones 4.0 y 4.1 (NFSv4), lo que permite integrarlo sin fricción con cargas de trabajo que requieren NFS. Las versiones de NFS importan porque cliente y servidor tienen que hablar la misma. **NFSv3** es *sin estado* (*stateless*): el servidor no recuerda qué cliente tiene abierto qué archivo, lo que lo hace simple y robusto, pero deja los bloqueos de archivos a un protocolo aparte. **NFSv4** es *con estado*: mantiene sesiones, integra los bloqueos de archivos, mejora la seguridad y usa un único puerto de red, lo que simplifica los firewalls. **NFSv4.1** mejora las sesiones y agrega extensiones para acceso en paralelo. Los clientes Linux modernos hablan NFSv4.1 sin configuración especial. Según la documentación de EFS, las instancias EC2 con Windows no están soportadas.

Después, puedes compartir estos archivos, optimizar costos con la gestión del ciclo de vida de Amazon EFS y proteger aún más tus datos con AWS Backup y la replicación de Amazon EFS. La **gestión del ciclo de vida** aplica reglas que mueven a clases más baratas los archivos que no se leen desde hace cierto número de días. **AWS Backup** es el servicio centralizado de copias de seguridad de AWS. La **replicación de EFS** mantiene una copia del sistema de archivos en otra región o zona.

Su capacidad elástica significa que el servicio escala las cargas de trabajo bajo demanda hasta petabytes de almacenamiento y gigabytes por segundo de **throughput**, sin configuración adicional. El throughput (tasa de transferencia) es la cantidad de datos que se mueven por segundo, en MB/s o GB/s. Si el almacenamiento fuera una carretera, el throughput serían los autos que pasan por hora y la latencia, el tiempo que tarda cada auto en llegar. Para dar escala, un disco duro mecánico lee de forma secuencial unos 150 a 250 MB/s, un SSD NVMe de laptop entre 3 y 7 GB/s, y una conexión doméstica de 1 Gbps transfiere 0.125 GB/s. Un **petabyte** son mil terabytes.

> [!note] Cifras de rendimiento de EFS (documentación consultada el 23-09-2026)
> En el modo de throughput *Elastic*, un sistema de archivos regional de EFS alcanza, según la región, de 20 a 60 GiB/s de lectura y de 1 a 5 GiB/s de escritura, con latencias de alrededor de 1 ms en lectura y 2.7 ms en escritura. Cada cliente individual llega como máximo a unos 1500 MiB/s (con el cliente `amazon-efs-utils` 2.0 o posterior). Es decir, los GB/s del total se alcanzan sumando muchos clientes, no desde una sola máquina. Verifica los valores vigentes antes de dimensionar.

Con su modelo de precios de pago por uso, puedes reducir el **costo total de propiedad** (TCO, *total cost of ownership*) aprovechando su función integrada de gestión del ciclo de vida, que mueve de forma inteligente los datos «fríos» a las clases de almacenamiento de EFS optimizadas en costo: Infrequent Access y Archive. El TCO suma todos los costos de una solución a lo largo del tiempo (hardware, personal que la administra, energía, licencias, fallas), no solo el precio por gigabyte. Según la documentación, la clase Standard de EFS usa SSD con latencias de lectura de alrededor de 1 ms, mientras que las clases Infrequent Access y Archive tienen latencias de decenas de milisegundos para el primer byte.

Como alternativa a Amazon S3, si tus datos de entrenamiento ya residen en Amazon EFS, puedes acceder a ellos fácilmente y usarlos en Amazon SageMaker para entrenar modelos. Esta integración simplifica el flujo de trabajo, porque te permite aprovechar el almacenamiento escalable y de alto rendimiento de Amazon EFS mientras desarrollas y despliegas tus modelos de ML en Amazon SageMaker de forma eficiente.

La Figura 2.5 ilustra un flujo de trabajo de ejemplo que muestra cómo se puede usar Amazon EFS para guardar datos de entrenamiento y artefactos de aprendizaje durante el ciclo de vida de ML.

*Figura 2.5 Uso de Amazon EFS en el ciclo de vida de ML.*

Al decidir entre Amazon EFS y Amazon S3 para el almacenamiento de ML en AWS, es esencial considerar los beneficios particulares de cada uno según el caso de uso. Amazon EFS ofrece una interfaz de sistema de archivos con consistencia fuerte, bloqueo de archivos y soporte para los protocolos NFS, lo que lo hace ideal para aplicaciones que requieren la **semántica** tradicional de un sistema de archivos y un acceso rápido a los datasets de entrenamiento.

- La **semántica de sistema de archivos** son las operaciones que un programa da por hecho al trabajar con archivos: abrir un archivo y modificar bytes en medio, agregar al final, renombrar una carpeta completa de una vez, listar un directorio o aplicar permisos de usuario y grupo al estilo **POSIX** (el estándar de interfaces de los sistemas tipo Unix, que incluye cómo se manejan los archivos y sus permisos). S3 no permite modificar un fragmento de un objeto ni renombrar una «carpeta» de forma atómica, porque las carpetas no existen como tales.
- La **consistencia fuerte** garantiza que, una vez terminada una escritura, cualquier lectura posterior, desde cualquier cliente, ve el dato nuevo.
- El **bloqueo de archivos** (*file locking*) es el mecanismo con el que un programa marca un archivo, o una parte, como «en uso» para que otro no escriba al mismo tiempo y lo corrompa. Es imprescindible cuando varias instancias escriben en los mismos archivos.

Sin embargo, es importante tener en cuenta que configurar Amazon EFS con Amazon SageMaker requiere algo de trabajo de **DevOps** (el rol y las prácticas que unen el desarrollo con la operación de la infraestructura: redes, permisos, despliegues). Hay que configurar un endpoint de VPC de tipo interfaz para asegurar una conectividad segura. Esto implica crear el endpoint en tu VPC, configurar los **grupos de seguridad** y asegurarte de que las políticas de IAM sean las adecuadas. Un grupo de seguridad es un firewall virtual asociado a un recurso que indica qué tráfico de red se permite entrar y salir; para NFS, por ejemplo, debe permitir el puerto 2049.

> [!warning] Nota de precisión: qué se configura realmente para usar EFS con SageMaker
> - **Consistencia.** Desde diciembre de 2020, Amazon S3 también ofrece consistencia fuerte de lectura tras escritura. La consistencia ya no distingue a EFS de S3; lo que los distingue es la semántica de sistema de archivos y el bloqueo.
> - **Red.** Según la documentación de SageMaker, el requisito para leer de EFS (y de FSx for Lustre) es que el job de entrenamiento **se conecte a una VPC**: le indicas subredes y grupos de seguridad desde los que pueda alcanzar el sistema de archivos. EFS no se alcanza por un endpoint de interfaz, sino por sus **puntos de montaje** (*mount targets*), unas interfaces de red que EFS coloca en tus subredes. Los endpoints de VPC entran en juego si la VPC no tiene salida a internet, para que el job pueda llegar a S3 y a las demás API de AWS que necesite.

En cambio, Amazon S3 es un servicio versátil de almacenamiento de objetos que destaca en el manejo de datasets grandes y no estructurados, y que cubre diversas necesidades de almacenamiento de tus cargas de trabajo de ML.

Aunque S3 suele ser más económico para el almacenamiento general, la elección entre Amazon EFS y Amazon S3 debe basarse en los requisitos específicos de tu aplicación. Amazon EFS es preferible para cargas de trabajo que necesitan acceso rápido a datos de entrenamiento guardados en un sistema de archivos compartido, con consistencia fuerte. S3 se adapta mejor al almacenamiento de objetos económico y a soluciones de almacenamiento a gran escala. Para dar escala a la diferencia, a precios de lista de la región us-east-1, un gigabyte en la clase Standard de EFS cuesta del orden de diez veces más que en S3 Standard. Verifica los precios vigentes.

#### Casos de uso (*Use Cases*)

Por sus características serverless, de escalabilidad y de confiabilidad, Amazon EFS es el más adecuado para casos de uso en que cantidades impredecibles de datos en archivos deben compartirse entre múltiples consumidores. El razonamiento es que «cantidades impredecibles» pide un almacenamiento que no haya que dimensionar (EFS es elástico), y «compartidos entre múltiples consumidores» pide que muchas máquinas monten lo mismo a la vez (EFS usa NFS). Justamente esas dos cosas son las que no ofrecen EBS (un disco de tamaño fijo para una instancia) ni S3 (sin semántica de archivos). Estos son algunos casos:

- **Ciencia de datos y ML.** Amazon EFS ofrece el rendimiento y la consistencia que necesitan las cargas de trabajo de ML y de analítica de big data. Un ejemplo conocido: SageMaker Studio Classic guarda en EFS los directorios personales de los usuarios, de modo que cada científico de datos encuentra sus notebooks y archivos en cualquier instancia que abra.
- **Desarrollo moderno de aplicaciones.** Amazon EFS permite a los desarrolladores compartir código y otros archivos de forma segura y organizada, lo que aumenta la agilidad de DevOps y permite responder más rápido a los comentarios de los clientes. Sus capacidades de servidor de archivos distribuido y de alta disponibilidad permiten que los desarrolladores compartan datos entre sus contenedores y aplicaciones serverless sin ninguna administración. Esto importa porque los contenedores son efímeros: cuando uno se reinicia, pierde su disco local, y EFS les da un almacenamiento persistente y compartido.
- **Sistemas de gestión de contenidos (CMS).** Como servidor de archivos distribuido, serverless, elástico y de alta disponibilidad, Amazon EFS simplifica el almacenamiento de las cargas de trabajo de los CMS modernos, lo que se traduce en una salida al mercado (*go-to-market*) más rápida y en más confiabilidad, más seguridad y menores costos. Un **CMS** es el software con que se publica y administra el contenido de un sitio web, como WordPress o Drupal. Cuando varios servidores web atienden el mismo sitio, todos deben ver las mismas imágenes subidas por los editores, y un sistema de archivos compartido lo resuelve.

### Amazon FSx for Lustre

Amazon FSx for Lustre es un servicio de almacenamiento totalmente administrado que proporciona un sistema de archivos de alto rendimiento y **escalado horizontal** (*scale-out*). Escalar horizontalmente es crecer agregando más servidores que trabajan en paralelo, a diferencia del escalado vertical (*scale-up*), que consiste en reemplazar un servidor por uno más grande. El horizontal no tiene el techo de una sola máquina.

Este servicio está construido sobre **Lustre**, un sistema de archivos paralelo y distribuido de código abierto. El nombre combina *Linux* y *cluster*, y Lustre es el sistema de archivos que usan muchas de las supercomputadoras más rápidas del mundo. *Paralelo* significa que cada archivo se divide en franjas (*stripes*) guardadas en muchos servidores de almacenamiento a la vez, mientras que los nombres, las carpetas y los permisos (los metadatos) viven en servidores aparte. Un cliente que lee un archivo grande obtiene sus trozos simultáneamente de muchos servidores, así que el throughput de todos se suma. Es como repartir un libro entre 50 fotocopiadoras en lugar de hacer cola en una. Para usarlo, cada máquina necesita el software cliente de Lustre, que existe para Linux.

Está diseñado para cargas de trabajo intensivas en cómputo, con latencias por debajo del milisegundo, hasta cientos de gigabytes por segundo de throughput y millones de operaciones de entrada/salida por segundo (IOPS), lo que lo hace ideal para ML, HPC, procesamiento de video y modelado financiero. Estas cifras necesitan escala:

- **IOPS** (*input/output operations per second*) es el número de operaciones individuales de lectura o escritura, normalmente pequeñas (de 4 a 16 KB), que un almacenamiento atiende por segundo. Es la métrica clave cuando se accede a muchos archivos pequeños o a posiciones dispersas, como en una base de datos o en un dataset de millones de imágenes pequeñas. Con archivos grandes que se leen de principio a fin, importa más el throughput. Las dos se relacionan así: throughput ≈ IOPS × tamaño de cada operación. Como escala, un disco duro mecánico da unas 100 a 200 IOPS, y un SSD NVMe de laptop llega a cientos de miles en pruebas sintéticas con muchas peticiones simultáneas (bastantes menos con una petición a la vez). «Millones de IOPS» equivale, entonces, a varios de los SSD más rápidos a su máximo, o a unos 10 000 discos mecánicos, pero entregado por la red y compartido entre cientos de máquinas.
- **Latencia por debajo del milisegundo** (*submillisecond*): menos de 1 ms por operación, a través de la red. Está en el orden de un SSD local y es unas cien veces menor que la latencia típica de S3 Standard.
- **Cientos de GB/s de throughput**: 100 GB/s equivalen a entre 15 y 30 SSD NVMe de laptop leyendo a plena velocidad, o a leer 1 TB en 10 segundos.
- El **modelado financiero** incluye, por ejemplo, simulaciones de Monte Carlo para el riesgo de una cartera, con miles de escenarios que leen los mismos datos de mercado en paralelo.

> [!note] Cifras vigentes (documentación consultada el 23-09-2026)
> AWS anuncia hoy para FSx for Lustre «hasta varios TB/s» de throughput por sistema de archivos, millones de IOPS, latencias por debajo del milisegundo y hasta 1200 Gbps por cliente en instancias con **EFA** (*Elastic Fabric Adapter*, una interfaz de red especial para HPC). Las cifras del libro («cientos de GB/s») eran correctas en su momento, pero se quedaron cortas. Verifica los valores vigentes.

Al ofrecer almacenamiento escalable, seguro y durable, Amazon FSx for Lustre te permite procesar datasets grandes de forma rápida y económica.

Igual que Amazon S3 y Amazon EFS, Amazon FSx for Lustre también se integra de forma nativa con Amazon SageMaker. Esta integración te permite usar este servicio de almacenamiento como origen de datos de tus jobs de entrenamiento de ML, lo que acelera mucho el entrenamiento. Como elimina la necesidad de descargar los datos de Amazon S3 a las instancias de entrenamiento, Amazon FSx for Lustre asegura tiempos de arranque y de entrenamiento más rápidos, y mejora la eficiencia general. El razonamiento implícito es este: con el modo de entrada por defecto de S3 (*File mode*), SageMaker copia el dataset completo al disco local de cada instancia antes de empezar a entrenar. Con 2 TB, eso son muchos minutos o incluso horas de espera, y se repite en cada job. FSx for Lustre, en cambio, se **monta**, y el entrenamiento empieza a leer de inmediato.

Como se ilustra en la Figura 2.6, las instancias de entrenamiento no saben que los datos de entrenamiento vienen de S3, porque Amazon FSx for Lustre actúa como una **capa de abstracción** entre el bucket de S3 y las instancias de entrenamiento. El resultado es una interfaz de sistema de archivos de alto rendimiento que puede configurarse como **caché intermedia** (*buffer cache*). Una capa de abstracción es una capa intermedia que oculta los detalles de la que está debajo: el código de entrenamiento lee rutas como `/fsx/imagenes/0001.jpg` sin saber nada de S3. Una caché intermedia es un almacenamiento rápido y temporal que guarda cerca del consumidor los datos que acaba de usar o que va a necesitar: la primera lectura de un archivo viene de S3 y las siguientes salen de Lustre. Esto pone de relieve el papel de FSx for Lustre como capa que abstrae la transferencia de datos, de modo que las instancias de entrenamiento trabajan con un sistema de archivos de alto rendimiento sin tener que gestionar ellas mismas la descarga de datos desde S3. En la documentación, el vínculo entre el sistema de archivos y el bucket se llama **asociación con un repositorio de datos** (*data repository association*).

*Figura 2.6 Uso de Amazon FSx for Lustre en el ciclo de vida de ML.*

Amazon FSx for Lustre admite dos patrones de diseño para cargar los datos: una carga única de S3 a Lustre y la **carga diferida** (*lazy loading*), que carga los datos de S3 poco a poco, a medida que se accede a ellos. La primera es una gran opción cuando necesitas poner rápidamente un dataset grande a disposición del procesamiento: ofrece un rendimiento superior, a un costo inicial más alto por la transferencia rápida de datos. La segunda es ideal cuando quieres minimizar la transferencia inicial y sus costos, porque los datos se cargan solo cuando se necesitan, aunque el primer acceso a cada dato puede tardar un poco más. Es la diferencia entre descargar una temporada completa de una serie antes de verla y reproducir cada episodio en streaming cuando lo abres. Según la documentación, al vincular un bucket, FSx importa de inmediato el **listado** de todos los objetos (sus nombres y metadatos), así que los archivos «aparecen» enseguida, y el **contenido** de cada uno se trae de S3 la primera vez que se lee, salvo que lo precargues. Esta flexibilidad convierte a Amazon FSx for Lustre en una opción excelente para cargas de trabajo que requieren alto throughput y acceso a los datos con baja latencia.

Además de sus ventajas de rendimiento, Amazon FSx for Lustre admite varias opciones de despliegue, incluidos sistemas de archivos *scratch* y persistentes, para adaptarse a las distintas necesidades de procesamiento de datos de tus aplicaciones de ML. Los sistemas de archivos *scratch* son ideales para almacenamiento efímero y procesamiento de corto plazo, mientras que los persistentes se adaptan mejor al almacenamiento de largo plazo y a las cargas de trabajo centradas en el throughput. La diferencia de fondo, según la documentación, es que en un sistema *scratch* los datos **no se replican** y no sobreviven a la falla de un servidor de archivos. En un sistema persistente, los datos se replican y los servidores que fallan se reemplazan. *Scratch* (borrador) sirve cuando los datos originales siguen a salvo en S3 y el sistema de archivos solo existe mientras dura el trabajo.

Con sus sólidas capacidades de integración (conectividad nativa con Amazon SageMaker), su configurabilidad expresiva (carga única y carga diferida) y sus opciones de despliegue flexibles (sistemas de archivos *scratch* y persistentes), Amazon FSx for Lustre es un servicio de almacenamiento potente para aplicaciones de ML que requieren acceso a los datos de alto rendimiento, escalable y de baja latencia.

> [!warning] Nota de precisión: lo que el libro no dice de FSx for Lustre
> - **Solo clientes Linux.** El acceso requiere el cliente de Lustre, que AWS ofrece para las distribuciones Linux habituales (Amazon Linux, RHEL, Ubuntu, SUSE). No hay acceso desde Windows ni por SMB.
> - **Red.** Para usarlo desde SageMaker, el job de entrenamiento debe conectarse a una VPC, y la documentación indica usar una subred de la misma zona de disponibilidad en la que está el sistema de archivos.
> - **Clases de almacenamiento.** Hoy existen tres: SSD (latencia constante por debajo del milisegundo en todo el dataset), HDD (latencia de un solo dígito de milisegundos) e *Intelligent-Tiering* (elástica, con latencia por debajo del milisegundo para los datos de acceso frecuente y una caché SSD opcional).
> - **Alternativas en SageMaker.** Además de copiar el dataset completo (*File mode*), SageMaker ofrece modos de entrada que leen de S3 bajo demanda (*FastFile mode*) o como flujo (*Pipe mode*), que también reducen el tiempo de arranque sin necesidad de FSx. Para el examen, sigue valiendo la regla del libro: si la prioridad absoluta es el tiempo de entrenamiento, la respuesta es FSx for Lustre.

Hasta aquí has visto los tres servicios de almacenamiento de AWS que se integran de forma nativa con el mecanismo de ingesta de datos de entrenamiento de Amazon SageMaker: Amazon S3, Amazon EFS y Amazon FSx for Lustre. Para el examen, asegúrate de entender los distintos equilibrios entre tiempos de entrenamiento, rendimiento y costos. Cada uno de estos servicios tiene características propias que pueden afectar a tus flujos de trabajo de ML. Amazon S3 es muy escalable y económico, lo que lo hace adecuado para datasets grandes y almacenamiento de largo plazo. Amazon EFS ofrece almacenamiento de archivos simple, escalable y totalmente administrado para usar con los servicios de AWS Cloud y con recursos on-premises, con acceso compartido y flexibilidad. Amazon FSx for Lustre está optimizado para la computación de alto rendimiento y baja latencia y para cargas de trabajo de ML, con distintos patrones de carga de datos y opciones de despliegue.

Como se muestra en la Figura 2.7, si se comparan los tres servicios usando el tiempo de entrenamiento como única dimensión, Amazon FSx for Lustre es tu mejor opción. Sin embargo, si el costo es un factor crítico, Amazon S3 es la opción más económica, seguida de Amazon EFS, que equilibra eficazmente costo y rendimiento. El libro da el orden sin justificarlo. Lustre es el más rápido porque lee en paralelo desde muchos servidores, con latencia por debajo del milisegundo y sin descarga inicial. Es el más caro porque, en sus clases SSD y HDD, se paga la capacidad aprovisionada mientras el sistema de archivos exista, se use o no, y además los datos originales siguen guardados (y facturados) en S3. EFS cobra solo por lo guardado, pero a una tarifa por GB bastante mayor que S3. Y S3 tiene la tarifa más baja, a cambio de la latencia más alta.

*Figura 2.7 Comparación de tiempos de carga de los datos de entrenamiento.*

#### Casos de uso (*Use Cases*)

Como servicio totalmente administrado construido sobre el sistema de archivos de alto rendimiento Lustre, Amazon FSx for Lustre es el más adecuado para los siguientes casos de uso:

- **Machine learning.** Gracias a su integración nativa con Amazon SageMaker, FSx for Lustre es perfecto para los jobs de entrenamiento de ML que requieren latencias por debajo del milisegundo para acceder a datasets grandes. Al usar FSx for Lustre como origen de datos, los ingenieros de ML y los científicos de datos pueden reducir mucho el tiempo de entrenamiento, gracias a su acceso y procesamiento de alta velocidad. La flexibilidad de cargar datos desde Amazon S3, ya sea con cargas únicas o con carga diferida, mejora la eficiencia de los flujos de trabajo de ML, así que es una excelente opción para tareas que requieren acceso rápido a los datos y alto throughput.
- **Computación de alto rendimiento.** Amazon FSx for Lustre también es ideal para cargas de trabajo de HPC que exigen procesar cantidades masivas de datos con baja latencia. Esto incluye aplicaciones como la investigación científica, las simulaciones y tareas de cómputo como el modelado meteorológico y la **secuenciación genómica**, lo que permite a investigadores e ingenieros hacer cálculos y análisis complejos de forma rápida y eficiente. La secuenciación genómica lee el ADN de una muestra y produce archivos enormes (del orden de cien gigabytes por genoma humano, según la profundidad de lectura) que luego procesan cadenas de programas que corren en paralelo.
- **Procesamiento de medios.** Amazon FSx for Lustre destaca en aplicaciones de procesamiento de medios como el **renderizado** (generar los fotogramas finales de una escena 3D), la **transcodificación** (convertir un video de un formato o resolución a otro) y la edición de video, donde un throughput alto y una latencia baja son críticos. Sus características de alto rendimiento permiten a los profesionales de medios procesar eficientemente archivos multimedia grandes y cumplir los exigentes requisitos de los entornos de producción. Esto asegura que las tareas terminen antes y que el flujo de trabajo sea más fluido, lo que lo convierte en un servicio de almacenamiento excelente para la industria de medios y entretenimiento.

### Amazon FSx for NetApp ONTAP

Amazon FSx for NetApp ONTAP es otro servicio de almacenamiento totalmente administrado de AWS que usa el archivo como tipo de almacenamiento y está construido sobre ONTAP, el popular sistema operativo de NetApp. **NetApp** es una empresa estadounidense, uno de los principales fabricantes de **arreglos de almacenamiento** empresarial: los gabinetes llenos de discos, con su propio software, que guardan los datos en los centros de datos corporativos. **ONTAP** es el sistema operativo que corre en esos equipos y administra volúmenes, copias instantáneas, replicación y protocolos. Importa porque muchas empresas grandes llevan años operando NetApp, con procedimientos, scripts y replicación hacia sitios de respaldo, y FSx for ONTAP les permite conservar todo eso en AWS.

Amazon FSx for NetApp ONTAP ofrece almacenamiento de archivos de alto rendimiento que permite usar las capacidades unificadas de gestión de datos de ONTAP, entre ellas las siguientes:

- Gestión unificada del almacenamiento que puede abarcar flash, disco y nube, con cargas de trabajo SAN, NAS y de objetos. *Flash* es el almacenamiento en chips de memoria (SSD). Una **SAN** (*storage area network*, red de área de almacenamiento) es una red dedicada por la que los servidores acceden a almacenamiento de **bloques** remoto: ven un volumen del arreglo como si fuera un disco propio. Una **NAS** (*network attached storage*, almacenamiento conectado a la red) es un equipo que sirve **archivos** por la red, mediante NFS o SMB. Dicho de forma breve, SAN es bloques por la red y NAS es archivos por la red. «Unificado» significa que el mismo sistema ofrece ambos, y además objetos.
- Acceso con baja latencia.
- Compresión de datos.
- **Deduplicación de datos**: detectar bloques idénticos guardados varias veces y conservar una sola copia con referencias a ella. Por ejemplo, cien discos de máquinas virtuales con el mismo sistema operativo comparten la mayor parte de sus bloques.
- Escalabilidad del almacenamiento.

La compresión y la deduplicación de datos generan ahorros, porque reducen el tamaño de los datos que hay que almacenar.

> [!note] Capacidades vigentes que el libro no menciona (documentación consultada el 23-09-2026)
> Según la documentación, FSx for ONTAP ofrece acceso **multiprotocolo** (NFS, SMB, iSCSI y NVMe), de modo que los mismos datos pueden servirse a la vez a clientes Linux y Windows. También ofrece **SnapMirror** (la replicación nativa de NetApp, compatible con equipos NetApp on-premises), FlexCache, copias instantáneas (*snapshots*) y clones, un nivel automático más barato para los datos poco usados, integración con Active Directory y hasta decenas de GB/s de throughput por sistema de archivos.

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de Amazon FSx for NetApp ONTAP:

- **Migración de cargas de trabajo.** Migra sin fricción a AWS las cargas de trabajo que corren en NetApp o en otros servidores NFS, SMB, iSCSI y NVMe-over-TCP, sin modificar el código de las aplicaciones ni la forma en que gestionas los datos. Migrar sin modificar la aplicación, cambiando solo dónde corre, es lo que en la industria se llama ***lift and shift*** («levantar y trasladar»). Es la migración más rápida y barata, porque no exige reescribir nada. **iSCSI** transporta los comandos de disco por una red IP común, de modo que un servidor ve un volumen remoto como si fuera un disco local. Es almacenamiento de bloques por la red, el protocolo típico de una SAN económica. **NVMe** es el protocolo moderno diseñado para los SSD, con menos latencia que los protocolos de disco anteriores, y **NVMe-over-TCP** lo lleva a través de una red Ethernet estándar. (El original dice «NFSs»; por el contexto, se refiere a servidores NFS.)
- **Continuidad del negocio y recuperación ante desastres (BCDR).** Logra copias de seguridad, archivo y replicación de datos seguros desde servidores de archivos on-premises o entre regiones de AWS. La **continuidad del negocio** es la capacidad de seguir operando durante un incidente, y la **recuperación ante desastres** es la capacidad de restaurar los sistemas después de una catástrofe, como la pérdida de un centro de datos. (El original escribe «BDCR», una errata de BCDR.)
- **Cargas de trabajo de bases de datos de alto rendimiento.** Con latencias por debajo del milisegundo y escalabilidad hasta millones de IOPS por sistema de archivos, Amazon FSx for NetApp ONTAP ofrece almacenamiento de archivos compartido y de alta disponibilidad para tus bases de datos de alto rendimiento. Motores como Oracle o SQL Server pueden guardar sus archivos de datos en volúmenes NFS o SMB, o en volúmenes de bloques por iSCSI. Además, Amazon FSx for NetApp ONTAP te permite escalar horizontalmente los sistemas de archivos repartiendo las cargas de trabajo de los clientes entre varios servidores de archivos. (Verifica en la documentación vigente las cifras máximas de IOPS según el tipo de despliegue.)

### Amazon FSx for Windows File Server

Amazon FSx for Windows File Server es un servicio totalmente administrado que ofrece almacenamiento de archivos basado en Windows, muy confiable, escalable y de alto rendimiento. Se integra sin fricción con tus aplicaciones y entornos Windows existentes y tiene soporte nativo para el protocolo SMB de Windows. Esto lo convierte en una solución ideal para el almacenamiento de archivos compartido en casos de uso como los directorios personales, los perfiles de usuario y las aplicaciones empresariales que requieren almacenamiento de archivos.

- Un **directorio personal** (*home directory*) es la carpeta de red propia de cada empleado, normalmente mapeada como una unidad con letra, que lo acompaña en cualquier computadora de la empresa.
- Un **perfil de usuario** es la carpeta donde Windows guarda el escritorio, la configuración y los documentos de cada persona. Si se guarda en un servidor, el usuario encuentra el mismo entorno en cualquier equipo.
- Una **aplicación empresarial** de este tipo es, por ejemplo, un sistema de gestión documental o un ERP que guarda sus archivos en una carpeta compartida.

Gracias a su integración con **Active Directory**, asegura un control de acceso y una gestión de usuarios seguros, y sus funciones de deduplicación de datos ayudan a reducir los costos de almacenamiento eliminando archivos duplicados. Active Directory (AD) es el servicio de directorio de Microsoft: la base de datos central de usuarios, grupos y computadoras de una organización, y el sistema que los autentica, el que hace posible iniciar sesión una vez con la cuenta del trabajo. En Windows, los permisos de cada archivo o carpeta se expresan con **listas de control de acceso** (ACL) que nombran a usuarios y grupos de AD («Finanzas puede leer, Contabilidad puede escribir»). Por eso, según la documentación, un sistema FSx for Windows File Server debe unirse a un Active Directory al crearse. La deduplicación de Windows Server trabaja, además, a nivel de fragmentos dentro de los archivos, no solo de archivos completos duplicados.

Este servicio de almacenamiento ofrece un rendimiento robusto con opciones de almacenamiento SSD, lo que asegura acceso a tus archivos con baja latencia y altas IOPS para las aplicaciones exigentes. Su escalabilidad flexible te permite ajustar fácilmente el tamaño del sistema de archivos a tus necesidades crecientes de almacenamiento, lo que lo convierte en una solución de almacenamiento potente y eficiente para entornos basados en Windows.

> [!note] Datos vigentes (documentación consultada el 23-09-2026)
> Según la documentación, FSx for Windows File Server admite SMB desde la versión 2.0 hasta la 3.1.1, y se accede a él desde Windows (a partir de Windows 7 y Windows Server 2008) y desde versiones actuales de Linux. Ofrece almacenamiento SSD y también HDD (más barato, pensado para directorios personales y carpetas departamentales), despliegues en una zona o en varias zonas con un servidor de reserva, y latencias constantes por debajo del milisegundo. Se administra con PowerShell y, en algunos casos, con las herramientas gráficas nativas de Windows.

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de Amazon FSx for Windows File Server:

- **Migración de servidores de archivos Windows.** Como ofrece un sistema de archivos Windows nativo, totalmente administrado, con soporte para el protocolo SMB e integración con Active Directory, Amazon FSx for Windows File Server permite a las organizaciones trasladar sus sistemas de archivos existentes a AWS sin modificar sus aplicaciones, con una transición fluida y mínimas interrupciones.
- **Ahorro de costos en SQL Server.** Amazon FSx for Windows File Server permite despliegues de alta disponibilidad sin necesidad de licencias de SQL Server Enterprise, lo que genera ahorros importantes. Esto es especialmente beneficioso para las organizaciones centradas en SQL Server que estén considerando migrar a AWS. El libro da por sentado el razonamiento. SQL Server tiene dos mecanismos principales de alta disponibilidad. Los *Always On Availability Groups*, en su versión completa, requieren la edición Enterprise, mucho más cara por núcleo. Las *Failover Cluster Instances* (FCI) están disponibles en la edición Standard (con dos nodos), pero exigen un **almacenamiento compartido** al que accedan ambos servidores. FSx for Windows File Server aporta ese almacenamiento compartido por SMB, de modo que la edición Standard más FSx logra la alta disponibilidad sin pagar Enterprise. Confirma los detalles de licenciamiento con los términos vigentes de Microsoft.
- **Escritorios virtuales y streaming de aplicaciones.** Con Amazon FSx for Windows File Server, puedes guardar los datos de los perfiles de usuario en un almacenamiento compartido y persistente, accesible desde Amazon WorkSpaces y Amazon AppStream 2.0. Este servicio de almacenamiento puede simplificar la experiencia de los usuarios de escritorios virtuales, porque reduce los tiempos de inicio de sesión y mejora la productividad general. **Amazon WorkSpaces** ofrece escritorios virtuales (VDI): un escritorio Windows que corre en la nube y se usa desde cualquier dispositivo. **AppStream 2.0** (hoy Amazon WorkSpaces Applications) transmite aplicaciones individuales al navegador. En estos entornos, la máquina del usuario suele recrearse en cada sesión, así que su perfil tiene que vivir en un almacenamiento compartido. El libro no explica por qué se reducen los tiempos de inicio de sesión. En las soluciones habituales de perfiles, como FSLogix, el perfil completo se guarda como un disco virtual en la carpeta compartida y se monta al iniciar sesión, en lugar de copiarse archivo por archivo.

### Amazon FSx for OpenZFS

Amazon FSx for OpenZFS es un servicio de almacenamiento totalmente administrado que te permite operar y escalar sistemas de archivos OpenZFS en AWS. Combina las funciones y el rendimiento conocidos de OpenZFS con la escalabilidad y la simplicidad de AWS. Para entenderlo hacen falta dos conceptos previos:

- **ZFS** es un sistema de archivos que también hace de gestor de volúmenes (administra directamente los discos). Lo creó Sun Microsystems y se publicó a mediados de la década de 2000 para su sistema operativo Solaris. Sus rasgos distintivos son los siguientes:
  - **Copia en escritura** (*copy-on-write*): nunca sobrescribe un dato en su lugar, sino que escribe la versión nueva en otro sitio y luego actualiza los punteros. Eso lo protege de corrupciones si se corta la energía a mitad de una escritura.
  - **Sumas de verificación** en cada bloque, que detectan la corrupción silenciosa de datos.
  - **Snapshots** (copias instantáneas): copias de solo lectura del sistema de archivos en un instante, que se crean al momento y solo ocupan espacio por lo que cambie después.
  - **Clones**: copias escribibles creadas a partir de un snapshot, también al instante. Con ellos se puede tener en segundos una copia de 10 TB de una base de datos para hacer pruebas.
  - **Compresión** transparente: los programas no se enteran de que los datos están comprimidos.
- **OpenZFS** es la continuación de código abierto de ZFS que mantiene la comunidad, surgida después de que Oracle comprara Sun (2010) y cerrara el desarrollo de ZFS. Corre en Linux y FreeBSD, y es la base de muchos servidores y equipos NAS basados en Linux. Importa porque muchos equipos organizan su trabajo en torno a los snapshots y clones de ZFS.

Admite acceso desde instancias y contenedores con Linux, Windows y macOS mediante el protocolo NFS (v3, v4, v4.1 y v4.2). Ofrece más de un millón de IOPS y latencias por debajo del milisegundo, y aprovecha las tecnologías más recientes de cómputo, discos y redes de AWS para cargas de trabajo de alto rendimiento. Ofrecer tantas versiones de NFS amplía la compatibilidad (EFS, en comparación, solo admite 4.0 y 4.1). NFSv3 sigue siendo el preferido de muchas aplicaciones y herramientas antiguas. NFSv4.2 agrega funciones como la copia en el servidor, que copia un archivo sin que los datos viajen hasta el cliente y vuelvan. Windows y macOS acceden mediante sus clientes NFS; para permisos nativos de Windows, la opción natural sigue siendo FSx for Windows File Server. Para dar escala a las cifras, un millón de IOPS equivale a unos 5000 a 10 000 discos mecánicos, y una latencia de cientos de microsegundos está entre 0.1 y 0.9 ms, en el orden de un SSD local.

> [!note] Cifras y funciones vigentes (documentación consultada el 23-09-2026)
> La documentación actual habla de **hasta 2 millones de IOPS** con latencias de cientos de microsegundos. Da hasta 21 GB/s de throughput para los datos que se sirven desde la caché en memoria o NVMe, y hasta 400 000 IOPS y 10 GB/s (21 GB/s con compresión) para los datos que se leen del disco. También ofrece una clase de almacenamiento *Intelligent-Tiering*, despliegues en varias zonas y la posibilidad de adjuntar **S3 Access Points** a sus volúmenes para leer los mismos datos con la API de S3. Verifica los valores vigentes.

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de Amazon FSx for OpenZFS:

- **Migración de cargas de trabajo.** Como sugiere su nombre, el caso de uso principal de este servicio es migrar a AWS las cargas de trabajo que corren en ZFS (o en otros servidores de archivos basados en Linux) sin modificar el código de las aplicaciones ni las prácticas de gestión de datos. El razonamiento que el autor da por sentado es este. El nombre dice que el servicio **es** OpenZFS, así que un equipo que ya usa ZFS conserva las mismas funciones (snapshots, clones, compresión, cuotas por usuario) y, con ellas, sus procedimientos: los calendarios de snapshots, los entornos de prueba creados con clones y demás. Y sirve también para «otros servidores de archivos basados en Linux» porque su protocolo es NFS, la forma nativa de compartir archivos en Linux. Un servidor NFS Linux genérico se reemplaza cambiando la dirección que montan los clientes.
- **Aplicaciones intensivas en datos.** Gracias a las funciones avanzadas del sistema de archivos OpenZFS y a su amplio soporte de NFS, Amazon FSx for OpenZFS es el más adecuado para desarrollar aplicaciones intensivas en datos en AWS. Una aplicación intensiva en datos es aquella cuyo cuello de botella es leer y escribir datos, no calcular: analítica, preprocesamiento para ML, compilaciones de software con miles de archivos pequeños o servidores web. Las «funciones avanzadas» que el autor no enumera son las que las sirven: la caché en memoria y en NVMe, la compresión que multiplica el throughput efectivo, los clones para entornos de desarrollo y la latencia de cientos de microsegundos.

> [!warning] Nota de precisión: «sin modificar las prácticas de gestión de datos» tiene un matiz
> En FSx for OpenZFS, los snapshots y los clones se crean con la API, la consola o la CLI de Amazon FSx, no ejecutando comandos `zfs` en el servidor, al que no tienes acceso porque es un servicio administrado. Los scripts que hoy llaman a `zfs snapshot` o `zfs clone` tendrán que adaptarse a la API de FSx, aunque el concepto y el resultado sean los mismos.

### Amazon Elastic Block Storage (EBS)

Amazon EBS es un servicio de almacenamiento de bloques de alto rendimiento de AWS que puede usarse como una SAN en la nube. La comparación con una SAN se debe a que los volúmenes de EBS llegan a la instancia a través de la red de AWS, igual que los volúmenes de un arreglo llegan a los servidores por una SAN, y el sistema operativo los ve como discos propios.

> [!warning] Nota de precisión
> - El nombre oficial del servicio es **Amazon Elastic Block Store**, no *Elastic Block Storage*.
> - Hay dos diferencias con una SAN corporativa que el libro no menciona y que suelen decidir preguntas de examen. Un volumen EBS **vive en una sola zona de disponibilidad** y solo se conecta a instancias de esa zona. Y normalmente se conecta a **una sola instancia** a la vez. La excepción, *Multi-Attach*, solo existe en los volúmenes io1 e io2, para un número limitado de instancias de la misma zona, y exige un sistema de archivos o una aplicación que coordine las escrituras entre ellas.

El almacenamiento de Amazon EBS se presenta en forma de **volúmenes EBS**, que son similares a discos virtuales en la nube y pueden conectarse a instancias de Amazon EC2. Los volúmenes EBS pueden ser unidades de estado sólido (SSD) o discos duros (HDD). Los volúmenes basados en SSD están optimizados para cargas de trabajo transaccionales con operaciones de lectura y escritura frecuentes y de tamaño de E/S pequeño, donde la métrica de rendimiento clave son las IOPS. Los volúmenes basados en HDD están optimizados para cargas de trabajo grandes de *streaming*, donde la métrica clave es el throughput.

- Una **carga de trabajo transaccional** es la de una base de datos que registra operaciones (pedidos, pagos) una a una. Cada operación toca unos pocos KB en posiciones dispersas del disco, así que lo que limita es cuántas operaciones por segundo se pueden hacer (IOPS).
- Aquí «*streaming*» no significa flujo de eventos, como en Kinesis, sino leer o escribir **archivos grandes de forma secuencial**, de principio a fin, como un log enorme o un escaneo completo de una tabla.
- La razón física de la división es que un disco duro tiene un brazo mecánico y platos que giran. Saltar de una posición a otra es lento (unas 100 a 200 IOPS), pero leer seguido es rápido (cientos de MB/s). Por eso el HDD sirve para lo secuencial y el SSD, sin partes móviles, para lo aleatorio.

> [!note] Límites vigentes por volumen (documentación consultada el 23-09-2026)
> | Tipo | Medio | Máximo de IOPS | Máximo de throughput |
> | --- | --- | --- | --- |
> | gp3 (uso general) | SSD | 80 000 | 2000 MiB/s |
> | gp2 (uso general, generación anterior) | SSD | 16 000 | 250 MiB/s |
> | io2 Block Express (IOPS aprovisionadas) | SSD | 256 000 | 4000 MiB/s |
> | io1 (IOPS aprovisionadas) | SSD | 64 000 | 1000 MiB/s |
> | st1 (optimizado para throughput) | HDD | 500 | 500 MiB/s |
> | sc1 (en frío) | HDD | 250 | 250 MiB/s |
>
> Fíjate en el contraste: un volumen st1 da 500 IOPS pero 500 MiB/s, porque cada operación es de 1 MiB. Los máximos de gp3 aumentaron después de la edición del libro. Verifica los valores vigentes.

Con Amazon EBS también puedes tomar **snapshots**, que son copias de seguridad de volúmenes EBS en un momento dado y que pueden usarse para restaurar volúmenes nuevos. Los snapshots de EBS son **incrementales**: después del primero, cada snapshot guarda solo los bloques que cambiaron. AWS los almacena en S3, aunque no aparecen en tus buckets. Restaurar un snapshot en otra zona de disponibilidad es, además, la forma de «mover» un volumen de zona.

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de Amazon EBS:

- **Migración de almacenamiento a nivel de bloque.** Migra a AWS las cargas de trabajo de SAN on-premises de gama media (arreglos de almacenamiento de tamaño y precio intermedios). Conecta almacenamiento de bloques de alto rendimiento y alta disponibilidad a aplicaciones **de misión crítica**, aquellas cuya caída detiene el negocio.
- **Cargas de trabajo de RDBMS y NoSQL.** Despliega y escala las bases de datos que elijas, como SAP HANA, Oracle, Microsoft SQL Server, PostgreSQL, MySQL, Cassandra y MongoDB. Un **RDBMS** (*relational database management system*) es un sistema de gestión de bases de datos relacionales. **SAP HANA** es la base de datos en memoria de SAP, sobre la que corren sus sistemas de gestión empresarial, y **Cassandra** es una base de datos NoSQL distribuida. Lo que el libro da por sentado es que se trata de bases de datos **autoadministradas**: las instalas tú en instancias EC2 y guardas sus archivos de datos en volúmenes EBS. Es la alternativa a Amazon RDS (siguiente sección) cuando el motor no está disponible en RDS, como SAP HANA, Cassandra o MongoDB, o cuando necesitas control total sobre la configuración.
- **Cargas de trabajo de analítica de big data.** Redimensiona fácilmente clústeres de motores de analítica de big data, como Hadoop y Spark, y desconecta y vuelve a conectar volúmenes libremente. Por ejemplo, si reemplazas un nodo del clúster, puedes conectar su volumen con los datos al nodo nuevo (de la misma zona) en lugar de copiarlos. Además, un volumen puede agrandarse o cambiar de tipo sin desconectarlo.

### Amazon Relational Database Service (RDS)

Amazon RDS es un servicio administrado de bases de datos relacionales disponible con ocho **motores** distintos (el motor es el software de base de datos): Amazon Aurora PostgreSQL-Compatible Edition, Amazon Aurora MySQL-Compatible Edition, RDS for PostgreSQL, RDS for MySQL, RDS for MariaDB, RDS for SQL Server, RDS for Oracle y RDS for Db2. **MariaDB** es una bifurcación de MySQL mantenida por la comunidad, y **Db2** es la base de datos relacional de IBM. **Aurora** es el motor que diseñó AWS, compatible con MySQL o PostgreSQL. Las aplicaciones se conectan a él como si fuera uno de esos dos motores, pero su capa de almacenamiento está distribuida: guarda seis copias de los datos en tres zonas de disponibilidad.

Como eliges el motor de base de datos, puedes seguir usando con Amazon RDS el código, las aplicaciones y las herramientas que ya usas con tus bases de datos actuales. Por ejemplo, no hace falta refactorizar tus procedimientos almacenados de SQL Server ni recodificar tu aplicación (salvo la forma en que se conecta a la base de datos), ni usar otra herramienta de edición de SQL. Un **procedimiento almacenado** (*stored procedure*) es un programa que se guarda y ejecuta dentro de la propia base de datos, escrito en el dialecto SQL del motor, como T-SQL en SQL Server. **Refactorizar** es reestructurar código sin cambiar lo que hace. Todo esto es posible porque RDS ejecuta el mismo motor, con el mismo dialecto y las mismas funciones. Solo cambia la cadena de conexión (servidor, usuario y contraseña). La excepción son las funciones que requieren acceso de administrador al sistema operativo, que RDS no da; para esos casos existe RDS Custom, que se menciona más abajo.

Como servicio administrado, Amazon RDS se encarga por ti de las tareas de gestión de la base de datos, como el aprovisionamiento, los parches, las copias de seguridad, la recuperación, la detección de fallas y la reparación.

Amazon RDS ofrece tres entornos de despliegue: en la nube, con Amazon Aurora o Amazon RDS; para cargas de trabajo híbridas, con Amazon RDS on AWS Outposts (en tu propio centro de datos); y con acceso privilegiado, con **Amazon RDS Custom**. RDS Custom te da acceso de administrador al sistema operativo y a la base de datos para instalar software o configuraciones que una aplicación heredada exige. Está disponible para Oracle y SQL Server.

Como con todos los servicios de AWS, no se requieren inversiones iniciales y solo pagas por los recursos que usas. En RDS, «los recursos que usas» son principalmente las horas en que la instancia de base de datos está encendida, más el almacenamiento, no las consultas que ejecutas. También existen instancias reservadas, con pago anticipado opcional a cambio de un descuento.

#### Casos de uso (*Use Cases*)

Estos son casos de uso típicos de Amazon RDS:

- **Aplicaciones web y móviles modernas.** Amazon RDS es una excelente opción para construir aplicaciones bien diseñadas (*well-architected*), porque ofrece una solución de almacenamiento segura, con alta disponibilidad, rendimiento y escalabilidad, y con una administración limitada. Además, su modelo de pago por uso te permite gestionar eficazmente el costo de persistir tus datos en una base de datos relacional que corre en AWS.
- **Migración de bases de datos heredadas.** Migrar bases de datos heredadas on-premises a Amazon RDS es la opción natural para las organizaciones que buscan modernizar sus cargas de trabajo durante su transformación digital. Además de mejorar la escalabilidad y la confiabilidad, la reducción de costos es un factor crítico, por el ahorro importante en licencias y por el beneficio del modelo de pago por uso. El ahorro en licencias puede venir por dos caminos que el libro no distingue. El primero es cambiar de un motor comercial a uno de código abierto (por ejemplo, de Oracle a PostgreSQL o Aurora), lo que elimina las licencias, pero obliga a convertir el código SQL, en contra de lo que se dijo arriba sobre no refactorizar. El segundo es conservar el motor con la modalidad de «licencia incluida», que paga la licencia por hora de uso en lugar de comprarla por adelantado.

### Amazon DynamoDB

Amazon DynamoDB es un servicio de base de datos NoSQL serverless con un rendimiento constante de un solo dígito de milisegundos a cualquier escala. **NoSQL** agrupa las bases de datos no relacionales: no exigen el mismo esquema tabular en todos los registros, en general no hacen *joins* y están diseñadas para escalar horizontalmente, repartiendo los datos entre servidores según una clave.

Que sea serverless significa que no necesitas aprovisionar infraestructura ni parchear, administrar, instalar, mantener u operar software. DynamoDB también ofrece mantenimiento sin tiempo de inactividad (*zero-downtime*). En las bases de datos tradicionales, el mantenimiento suele exigir ventanas en las que el servicio se detiene.

Como base de datos NoSQL, DynamoDB está diseñada para ofrecer alto rendimiento, escalabilidad, facilidad de gestión y flexibilidad frente a las bases de datos relacionales. Para cubrir un amplio abanico de casos de uso, DynamoDB admite los modelos de datos de **clave-valor** y de **documentos**. En el modelo clave-valor, cada elemento se obtiene por una clave única, como en un diccionario de Python. En el de documentos, el valor es un documento anidado, parecido a un JSON, cuyos atributos pueden variar de un elemento a otro. Para ayudarte a construir aplicaciones de nivel empresarial, Amazon DynamoDB ofrece **consistencia fuerte en las lecturas** y soporte para **transacciones ACID** (atomicidad, consistencia, aislamiento y durabilidad). Por defecto, las lecturas de DynamoDB son *eventualmente consistentes*: durante una fracción de segundo tras una escritura pueden devolver el valor anterior. La lectura fuertemente consistente se pide de forma explícita en cada consulta. ACID resume las garantías de una transacción: se aplica toda o nada, respeta las reglas de los datos, no interfiere con otras transacciones simultáneas y, una vez confirmada, no se pierde.

Para lograr un rendimiento constante de un solo dígito de milisegundos, DynamoDB está optimizada para cargas de trabajo de alto rendimiento y ofrece API que fomentan un uso eficiente de la base de datos. Omite las funciones ineficientes y de bajo rendimiento a escala, como las operaciones JOIN. DynamoDB ofrece un rendimiento constante de un solo dígito de milisegundos a tus aplicaciones, tanto si hay 100 usuarios como si hay 100 millones. El razonamiento implícito es que un *join* combina filas de tablas que, a gran escala, están repartidas en servidores distintos, así que su costo crece con el volumen de datos. DynamoDB solo ofrece operaciones cuyo costo no depende del tamaño de la tabla, como obtener un elemento por su clave o leer los elementos que comparten una clave de partición. La consecuencia práctica es que las tablas se diseñan **a partir de los patrones de acceso**. El patrón de la Tabla 2.2 («buscar pedidos por ID de cliente e intervalo de tiempo») se resolvería con `customer_id` como clave de partición y la fecha como clave de ordenamiento.

#### Casos de uso (*Use Cases*)

DynamoDB es ideal para casos de uso que requieren un rendimiento constante a cualquier escala con una carga operativa mínima, entre ellos los siguientes:

- **Aplicaciones de servicios financieros.** Las transacciones de Amazon DynamoDB pueden usarse para lograr garantías ACID sobre una o más tablas con una sola petición. Las transacciones ACID son ideales para cargas de trabajo que procesan transacciones financieras o completan pedidos. Amazon DynamoDB se ajusta al instante a las cargas de trabajo cuando suben y bajan bruscamente, lo que te permite escalar tu base de datos de forma eficiente según las condiciones del mercado, como el horario de negociación de la bolsa. (El ajuste inmediato corresponde sobre todo al modo de capacidad *on-demand*, que también tiene límites ante picos muy bruscos, como los de Kinesis. Verifica su comportamiento en la documentación vigente.)
- **Aplicaciones de videojuegos.** Por su capacidad de reducir y ampliar su capacidad (*scale in* y *scale out*), su rendimiento constante y la facilidad de operación que da su arquitectura serverless, Amazon DynamoDB puede usarse para persistir de forma eficiente todos los datos de cualquier plataforma de juegos, como el estado del juego, los datos de los jugadores, el historial de sesiones y las tablas de clasificación (*leaderboards*). Esta escalabilidad optimiza la eficiencia de tu arquitectura, tanto cuando amplías capacidad para el tráfico pico como cuando la reduces porque hay pocos jugadores.
- **Aplicaciones de streaming de datos.** Las empresas de medios y entretenimiento usan mucho Amazon DynamoDB como índice de metadatos para servicios de gestión de contenidos, es decir, como el catálogo de cada título (nombre, duración, dónde está el archivo) que la aplicación consulta por ID, o para servir estadísticas deportivas casi en tiempo real. Amazon DynamoDB también se usa para servicios de listas de seguimiento y marcadores de los usuarios, y para procesar miles de millones de eventos diarios de clientes para generar recomendaciones. Estos clientes se benefician de la escalabilidad, el rendimiento y la resiliencia de DynamoDB. Su elasticidad integrada permite casos de uso de streaming de medios que soportan cualquier nivel de demanda.

## Resolución de problemas (*Troubleshooting*)

Resolver y depurar los problemas de ingesta y almacenamiento de datos relacionados con la capacidad y la escalabilidad en AWS implica varios pasos y buenas prácticas. Estas son algunas estrategias clave que necesitas conocer para el examen:

- **Monitoreo y registro.** Usa **CloudWatch** para monitorear tus recursos y aplicaciones de AWS. CloudWatch es el servicio de monitoreo de AWS: recoge **métricas** (series de tiempo numéricas, como el porcentaje de CPU o los bytes leídos por minuto) y logs. Configura **alarmas** que avisen a tu equipo de operaciones de datos ante cualquier anomalía en métricas como el uso de CPU, el uso de memoria y las operaciones de E/S; una alarma se dispara cuando una métrica cruza un umbral durante cierto tiempo. Activa **CloudTrail** para registrar las llamadas a la API y seguir los cambios en tus recursos de AWS. CloudTrail anota cada llamada a la API de tu cuenta: quién la hizo, qué hizo, cuándo y desde dónde. Esto ayuda a identificar la causa raíz de los problemas, por ejemplo, descubrir que alguien cambió la política de un bucket justo antes de que empezaran los errores.
- **Escalado y optimización del rendimiento.** Implementa el escalado automático (*auto scaling*) de tus instancias EC2 y otros recursos escalables para ajustar la capacidad automáticamente según la demanda; el escalado automático agrega o quita instancias según reglas sobre métricas. Usa **réplicas de lectura** y las capacidades de escalado automático de Aurora para manejar el aumento del tráfico de lectura y mejorar el rendimiento. Una réplica de lectura es una copia de solo lectura de una base de datos que recibe los cambios de la principal, de modo que las consultas de lectura se reparten entre varias copias. Aurora puede agregar o quitar réplicas automáticamente según la carga.
- **Optimización de la ingesta de datos.** Para optimizar el rendimiento de tus pipelines de datos, usa técnicas como el particionamiento de datos, el **almacenamiento en caché** (guardar en memoria rápida los resultados que se piden a menudo) y el procesamiento en paralelo. Para la ingesta de datos en tiempo real, aprovecha Amazon MSK, Amazon Manage Service for Apache Flink (sic; el nombre correcto es *Managed*), Amazon Kinesis Data Streams o Amazon Data Firehose para manejar grandes volúmenes de datos de streaming. Además, considera complementar estos servicios con funciones de AWS Lambda que procesen los datos en tiempo real y escalen automáticamente según el volumen de datos ingeridos.
- **Gestión del almacenamiento.** Usa Amazon S3 para el almacenamiento de objetos a escala empresarial. Implementa **políticas de ciclo de vida**, reglas que mueven los objetos a otras clases de almacenamiento (o los borran) tras cierto número de días según los patrones de acceso a los datos. Monitorea los volúmenes de Amazon EBS para vigilar su rendimiento y ajusta los tipos o tamaños de volumen según sea necesario para cumplir los requisitos de capacidad y rendimiento.
- **Optimización de bases de datos.** Asegúrate de que tus consultas estén optimizadas y de que los índices estén bien configurados para mejorar el rendimiento. Considera aplicar ***sharding*** a tu base de datos para repartir la carga entre varias instancias. El sharding divide horizontalmente una base de datos entre varios servidores según una clave (por ejemplo, clientes A–M en uno y N–Z en otro), de modo que cada servidor guarda y atiende solo una parte de las filas. Con Amazon DynamoDB, evita en lo posible las operaciones **Scan**, porque su rendimiento es ineficiente comparado con otras operaciones de consulta. Un Scan lee la tabla **completa**, elemento por elemento, así que su costo y su tiempo crecen con el tamaño de la tabla. Un **Query**, en cambio, lee solo los elementos de una clave de partición.
- **Gestión de costos.** Usa **AWS Cost Explorer**, la herramienta que desglosa el gasto por servicio, etiqueta y periodo, para monitorear y analizar tu gasto en AWS. Identifica las áreas donde puedes optimizar costos ajustando el uso de los recursos. Considera comprar **instancias reservadas** o **Savings Plans** para cargas de trabajo predecibles y reducir costos; ambos son compromisos de uso durante uno o tres años a cambio de un descuento. Aprovecha las **instancias spot** para cargas de trabajo elásticas y efímeras y así minimizar costos. Las instancias spot usan capacidad sobrante de EC2 con descuentos grandes (AWS habla de hasta un 90 %), pero AWS puede recuperarlas con un aviso de dos minutos. Por eso sirven para trabajos que toleran interrupciones, como procesos por lotes o entrenamientos que guardan *checkpoints*.

Al implementar estas estrategias, puedes resolver y depurar eficazmente los problemas de ingesta y almacenamiento de datos relacionados con la capacidad y la escalabilidad en AWS.

## Escenarios donde este servicio es la opción obligada

Este capítulo no trata de un solo servicio, sino de muchos. Por eso cada escenario se centra en uno distinto, elegido porque, dadas las restricciones del caso, es la única opción razonable dentro de AWS. Los dos primeros son de almacenamiento, donde las variantes de FSx más se confunden entre sí, y el tercero es de ingesta. Las empresas son ficticias; las restricciones son las que aparecen en proyectos reales. Las capacidades de los servicios que se citan se verificaron en la documentación de AWS el 23 de septiembre de 2026 y deben volver a verificarse antes de usarlas en una decisión real.

### Escenario 1: análisis genómico masivo en un instituto de investigación → Amazon FSx for Lustre

**Contexto.** Un instituto de investigación en genómica recibe cada semana unos 2000 genomas humanos secuenciados, del orden de 200 TB de archivos que los secuenciadores depositan en un bucket de S3. Su pipeline hace el alineamiento y el llamado de variantes (identificar en qué posiciones el ADN de cada paciente difiere del genoma de referencia), e incluye un llamador de variantes basado en deep learning que corre en GPU. Usa herramientas bioinformáticas de terceros, validadas por la comunidad científica. El pipeline corre los fines de semana en un clúster de unas 400 instancias EC2 Linux, lanzado con AWS ParallelCluster o AWS Batch, que existe solo durante la corrida. Los resultados deben volver a S3, tanto para archivarse como para que el equipo de ML entrene modelos con ellos.

**Restricciones clave.**

1. Las herramientas leen y escriben archivos por ruta, con acceso aleatorio dentro de archivos de decenas de GB y con archivos intermedios. El instituto no puede modificarlas: son de terceros, y cambiarlas obligaría a revalidar científicamente los resultados.
2. Para terminar dentro del fin de semana, el pipeline necesita sostener, sumando todos los nodos, del orden de 100 GB/s de lectura y decenas de GB/s de escritura.
3. Los 400 nodos deben ver el mismo espacio de archivos, porque lo que un paso escribe en un nodo lo lee el paso siguiente en otro.
4. Los datos de entrada y los de salida viven en S3, y el equipo no quiere un paso de copia manual antes y después de cada corrida.
5. El almacenamiento de trabajo solo hace falta durante la corrida. Los originales ya están a salvo en S3, así que no quieren pagar almacenamiento de alto rendimiento toda la semana.

**Por qué este servicio.**

- (1) → Según la documentación, FSx for Lustre es compatible con POSIX, con bloqueo de archivos y consistencia de lectura tras escritura, y las aplicaciones Linux existentes lo usan sin cambios, como un disco local.
- (2) → Es un sistema de archivos paralelo, con throughput documentado de hasta varios TB/s y latencia por debajo del milisegundo. El requisito cabe holgadamente.
- (3) → Todos los nodos Linux montan a la vez el mismo sistema de archivos.
- (4) → La asociación con un repositorio de datos presenta los objetos del bucket como archivos, carga su contenido al primer acceso y permite escribir los resultados de vuelta a S3.
- (5) → Un despliegue *scratch* no replica los datos y está pensado justamente para procesamiento de corto plazo: se crea el viernes y se elimina el lunes. Además, FSx for Lustre se integra con AWS ParallelCluster y AWS Batch.

**Por qué no las alternativas.**

- **Amazon S3** no es un sistema de archivos: las herramientas no pueden abrir rutas ni escribir en medio de un archivo. Existe un cliente de AWS que monta un bucket como carpeta (*Mountpoint for Amazon S3*), pero, hasta donde sé, solo admite escrituras secuenciales (no puede modificar un archivo en posiciones arbitrarias) y no ofrece bloqueos, lo que choca con la restricción 1. Verifica sus capacidades vigentes en su documentación.
- **Amazon EFS** lo pueden montar todos los nodos por NFS, pero su escritura documentada por sistema de archivos (de 1 a 5 GiB/s según la región) queda muy por debajo de las decenas de GB/s que exige la restricción 2, y no se vincula con un bucket de S3.
- **Amazon EBS** conecta cada volumen a una instancia; Multi-Attach solo cubre unas pocas instancias de una zona y exige software de clúster, así que no sirve para 400 nodos que comparten archivos.
- **Amazon FSx for OpenZFS** ofrece NFS de alto rendimiento, pero su techo documentado (hasta 21 GB/s desde caché y 10 GB/s desde disco) no alcanza la restricción 2, y no importa los objetos de un bucket como archivos.
- **Amazon FSx for NetApp ONTAP** llega, según su documentación, a «decenas de GB/s» por sistema de archivos, en el límite o por debajo del requisito, y sus puntos fuertes (multiprotocolo, SnapMirror) no aportan nada aquí.
- **Amazon FSx for Windows File Server** está pensado para clientes Windows que acceden por SMB, no para un clúster Linux de HPC que espera semántica POSIX.

### Escenario 2: fabricante industrial que cierra un centro de datos con NetApp → Amazon FSx for NetApp ONTAP

**Contexto.** Un fabricante de autopartes cierra uno de sus dos centros de datos y traslada su contenido a AWS. En ese centro, un arreglo NetApp sirve tres cosas. Primero, los archivos de diseño (CAD) y los datos de pruebas de laboratorio, que usan a la vez las estaciones de trabajo Windows de los ingenieros por SMB y los servidores Linux de simulación de esfuerzos por NFS, **sobre las mismas carpetas**. Segundo, los volúmenes de bloques por iSCSI de la base de datos de control de calidad. Tercero, la réplica SnapMirror hacia el NetApp del otro centro de datos, que se queda on-premises como sitio de recuperación ante desastres. Además, el equipo de ciencia de datos quiere entrenar modelos de calidad predictiva con diez años de datos de pruebas.

**Restricciones clave.**

1. Los mismos datos deben ser accesibles simultáneamente por SMB, con los permisos de Active Directory, y por NFS, desde Linux.
2. El proveedor de la aplicación de control de calidad solo certifica almacenamiento de bloques por iSCSI para su base de datos.
3. Los procedimientos de recuperación ante desastres (auditados por los clientes del sector automotriz) recuperan la base de datos y los archivos juntos a partir de réplicas SnapMirror hacia el NetApp que queda on-premises. La empresa no quiere reescribirlos ni volver a certificarlos.
4. El plazo es de tres meses y es un *lift and shift*: no se modifica ninguna aplicación. El equipo de almacenamiento conoce ONTAP y no otros sistemas.
5. Diez años de datos de pruebas ocupan mucho y casi no se consultan, así que el costo por GB importa.

**Por qué este servicio.**

- (1) → Es la única variante de FSx con acceso multiprotocolo (NFS, SMB, iSCSI y NVMe) sobre los mismos datos, con integración con Active Directory.
- (2) → Ofrece volúmenes de bloques por iSCSI en el mismo sistema.
- (3) → Admite **SnapMirror**, la replicación nativa de NetApp, compatible con los equipos NetApp on-premises, así que la relación de réplica y los procedimientos se conservan.
- (4) → Se administra también con la CLI y la API REST de ONTAP y con las herramientas de NetApp, de modo que el equipo reutiliza sus conocimientos y scripts.
- (5) → La deduplicación, la compresión y el paso automático de los datos poco usados a un nivel más barato reducen el costo del histórico.
- Para el equipo de ML, DataSync admite FSx for ONTAP como origen, así que se puede copiar el histórico a S3 y entrenar desde allí con la integración nativa de SageMaker.

**Por qué no las alternativas.**

- **Amazon FSx for Windows File Server** cubre SMB y Active Directory, pero no ofrece NFS ni iSCSI ni SnapMirror. Los servidores Linux tendrían que pasarse a SMB, con otra semántica de permisos, y los procedimientos de recuperación ante desastres dejarían de valer.
- **Amazon FSx for OpenZFS** solo habla NFS: deja fuera a los usuarios Windows por SMB, a la base de datos por iSCSI y la réplica SnapMirror.
- **Amazon FSx for Lustre** solo tiene cliente Linux, sin SMB ni iSCSI.
- **Amazon EFS** solo ofrece NFS y no admite clientes Windows.
- **Amazon EBS** podría alojar la base de datos, pero no comparte archivos ni replica con SnapMirror hacia el NetApp on-premises, así que rompe la restricción 3.
- **Amazon S3** no ofrece SMB, NFS ni iSCSI, así que habría que reescribir las aplicaciones.

### Escenario 3: plataforma de logística con un ecosistema Kafka → Amazon MSK

**Contexto.** Una empresa de reparto de última milla para comercio electrónico opera on-premises un clúster de Apache Kafka con unos 150 topics. Sesenta microservicios producen y consumen eventos con las bibliotecas cliente de Kafka en Java y Python. Kafka Connect, con conectores de CDC, captura los cambios de su base de datos PostgreSQL de pedidos, y varias aplicaciones de Kafka Streams calculan en vivo el tiempo estimado de entrega. El equipo de ML entrena modelos de tiempo de entrega y de demanda. Cuando cambia la definición de una característica, necesita recalcularla releyendo 30 días de eventos GPS de los vehículos. La empresa migra a AWS, las dos personas que operaban Kafka dejan la empresa y la dirección prohíbe operar clústeres propios.

**Restricciones clave.**

1. No se puede modificar el código de los 60 servicios ni el de las aplicaciones de Kafka Streams; solo su configuración de conexión.
2. Hay que reutilizar los conectores de Kafka Connect que ya existen.
3. Hay que poder releer (*replay*) 30 días de historia, con varios grupos de consumidores independientes que leen a ritmos distintos.
4. Nadie debe operar servidores, aplicar parches ni reemplazar nodos caídos.
5. Hay que preservar el orden de los eventos de cada vehículo.

**Por qué este servicio.**

- (1) → MSK ejecuta Apache Kafka de código abierto y habla su mismo protocolo, así que las aplicaciones y herramientas existentes funcionan sin cambios de código; solo cambian la dirección del clúster y la autenticación.
- (2) → Los conectores funcionan contra MSK como contra cualquier Kafka, y AWS ofrece además un servicio administrado para ejecutarlos, *MSK Connect*.
- (3) → Los topics retienen los eventos según la retención que configures, y cada grupo de consumidores guarda su propio offset y puede rebobinarlo.
- (4) → AWS opera los brokers, los parchea y los reemplaza cuando fallan. Los controladores KRaft vienen incluidos, y MSK Serverless elimina incluso el dimensionamiento.
- (5) → Kafka garantiza el orden dentro de cada partición, y usar el ID del vehículo como clave envía todos sus eventos a la misma partición.

**Por qué no las alternativas.**

- **Amazon Kinesis Data Streams** tiene conceptos equivalentes (shards, retención de hasta 365 días), pero una API propia distinta del protocolo de Kafka. Habría que reescribir los 60 servicios, las aplicaciones de Kafka Streams y los conectores, lo que viola las restricciones 1 y 2.
- **Amazon Data Firehose** es un conducto de entrega: no guarda un flujo que las aplicaciones puedan consumir ni releer. Puede leer de MSK, pero no reemplazarlo.
- **Amazon Managed Service for Apache Flink** procesa flujos, pero no los almacena. Podría complementar a MSK más adelante (por ejemplo, reemplazando a Kafka Streams), pero no ser la columna vertebral de eventos.
- **Amazon SNS** reparte mensajes a sus suscriptores, pero no los retiene para releerlos 30 días después.
- **Kafka autoadministrado en EC2** sería compatible, pero viola la restricción 4.

## Resumen (*Summary*)

En este capítulo cubrimos la ingesta y el almacenamiento de datos, los dos componentes clave de la fase de recolección del ciclo de vida de machine learning.

Con la ingesta, reúnes los datos de tu solución de ML desde distintas fuentes y los envías a AWS en su formato crudo. La ingesta puede ocurrir por lotes o en tiempo real. Amazon Data Firehose, Amazon Kinesis Data Streams, Amazon MSK y Amazon Managed Service for Apache Flink se usan para ingerir datos en tiempo real. AWS DataSync y AWS Glue se usan para ingerir datos por lotes.

Con el almacenamiento, persistes los datos ingeridos en un almacén de datos adecuado de AWS, donde permanecerán hasta que estén listos para procesarse. El almacenamiento es de tres tipos: de objetos, de archivos y de bloques. Amazon S3 se usa para guardar datos de objetos, Amazon EBS para datos de bloques, y Amazon EFS y la familia de servicios Amazon FSx para datos de archivos.

Los factores para seleccionar el servicio de ingesta y de almacenamiento adecuado dependen de las particularidades de tu caso de uso e incluyen la escalabilidad, la resiliencia, la seguridad y el costo.

El capítulo también cubrió los distintos formatos de datos que necesitas conocer para el examen, que pueden usarse para optimizar aún más el rendimiento, mejorar la escalabilidad y reducir el tiempo de procesamiento. Un factor importante al seleccionar un formato de datos para entrenar tu modelo es si el algoritmo de ML que piensas usar para tu problema lo admite.

## Puntos esenciales para el examen (*Exam Essentials*)

**Conoce la diferencia entre la ingesta y el almacenamiento de datos.** La ingesta de datos consiste en reunir datos de distintas fuentes, mientras que el almacenamiento consiste en persistir los datos en un almacén de datos alojado en AWS.

**Entiende los distintos formatos de datos.** CSV se usa para guardar datos estructurados en formato tabular, mientras que JSON se usa para datos semiestructurados basados en documentos. Apache Parquet y Apache ORC son formatos de datos columnares, mientras que Apache Avro es un formato por filas. RecordIO es un formato de datos que usa principalmente Apache MXNet, un framework de deep learning.

**Entiende los servicios de ingesta para datos de streaming.** Entre ellos están Amazon Data Firehose, Amazon Kinesis Data Streams, Amazon MSK y Amazon Streaming Service for Apache Flink. Amazon Kinesis Data Streams es más adecuado para el procesamiento de datos personalizado en tiempo real, mientras que Amazon Data Firehose es ideal para cargar eficientemente datos de streaming en almacenes de datos de AWS seleccionados, con una configuración y una gestión mínimas. Amazon MSK es el más adecuado para gestionar y distribuir datos de streaming (p. ej., agregación de logs, analítica en tiempo real y *event sourcing*), mientras que Amazon Managed Service for Apache Flink está diseñado para procesar y analizar datos en tiempo real. La **agregación de logs** consiste en reunir en un solo lugar los logs que generan muchos servidores. El ***event sourcing*** es un patrón de diseño que guarda el estado de un sistema como la secuencia completa de eventos que lo produjeron, en lugar de guardar solo el estado actual; es la idea detrás del caso de uso de MSK como sistema de registro.

> [!warning] Nota de precisión
> «Amazon Streaming Service for Apache Flink» es una errata del original: el servicio se llama **Amazon Managed Service for Apache Flink**, como en el resto del capítulo.

**Entiende los servicios de ingesta para migración de datos, datos por lotes y ETL.** Entre ellos están AWS DataSync y AWS Glue. El primero es ideal para transferir o migrar datos a AWS, y el segundo para ETL y para desarrollar pipelines de datos.

**Entiende los servicios de almacenamiento para datos de objetos.** Amazon S3 es el servicio de almacenamiento de objetos más versátil y más usado de AWS. Ofrece escalabilidad, disponibilidad de datos, seguridad y rendimiento líderes en la industria. Un dato de objeto contiene el dato en sí, sus metadatos y un identificador único. Al almacenamiento de objetos se accede mediante API, lo que lo hace adecuado para aplicaciones nativas de la nube. Amazon SageMaker admite de forma nativa Amazon S3 como origen de datos para entrenar modelos de ML.

**Entiende los servicios de almacenamiento para datos de archivos.** Amazon EFS y la familia de servicios Amazon FSx (Amazon FSx for Lustre, for NetApp ONTAP, for Windows File Server y for OpenZFS) se usan para guardar datos con el tipo de almacenamiento de archivos. El almacenamiento de archivos organiza los datos de forma jerárquica (carpetas dentro de carpetas) y es ideal para persistir datos compartidos, como las carpetas compartidas de una empresa, el almacenamiento de medios y los sistemas de gestión de contenidos. Amazon EFS y Amazon FSx for Lustre son servicios de almacenamiento de archivos que Amazon SageMaker admite de forma nativa como orígenes de datos para entrenar modelos de ML.

**Entiende los servicios de almacenamiento para datos de bloques.** Amazon EBS es el servicio de almacenamiento de bloques de AWS. El almacenamiento de bloques divide los datos en bloques de tamaño fijo, cada uno con una dirección única, pero sin metadatos. Es el sistema de archivos que se instala encima el que agrega los nombres, las carpetas y los permisos. El almacenamiento de bloques ofrece alto rendimiento y baja latencia, lo que lo hace ideal para cargas de trabajo transaccionales.

**Entiende los servicios de almacenamiento que se integran de forma nativa con Amazon SageMaker.** Amazon SageMaker se integra de forma nativa con Amazon S3 para el almacenamiento de objetos, y con Amazon EFS y Amazon FSx for Lustre para el almacenamiento de sistemas de archivos. Así facilita el acceso a los datos para el entrenamiento y el aprendizaje, y la gestión eficiente de los flujos de trabajo de ML.

## Preguntas de repaso (*Review Questions*)

1. Necesitas guardar datos sin procesar de dispositivos IoT para un nuevo pipeline de machine learning. La solución de almacenamiento debe ser un repositorio centralizado y de alta disponibilidad. ¿Qué servicio de almacenamiento de AWS eliges para guardar los datos sin procesar?
   - A. Amazon Elastic File System (EFS)
   - B. Amazon S3
   - C. Amazon DynamoDB
   - D. Amazon Relational Database Service (RDS)

2. Estás diseñando un repositorio de datos muy escalable para tu pipeline de machine learning. Necesitas acceso inmediato a los datos procesados de tu pipeline durante 6 meses. Tus datos sin procesar deben ser accesibles en un plazo de 12 horas y conservarse durante 6 años. La solución de almacenamiento debe admitir consultas SQL. ¿Cuál es la solución de almacenamiento más económica?
   - A. Amazon S3 y Amazon Athena
   - B. Amazon S3
   - C. Amazon DynamoDB
   - D. Amazon Redshift

3. Usas un flujo de entrega de Amazon Data Firehose para ingerir registros de datos comprimidos con GZIP desde una aplicación on-premises. Necesitas configurar una solución para que tu científico de datos ejecute consultas SQL sobre el flujo de datos y obtenga información en tiempo real. ¿Qué solución cumple estos requisitos?
   - A. Amazon S3 y Amazon Athena
   - B. Amazon Managed Service for Apache Flink y una función Lambda
   - C. Amazon Managed Streaming for Apache Kafka y una función Lambda
   - D. Amazon Redshift y Amazon Athena

4. Eres ingeniero de machine learning y necesitas procesar una gran cantidad de datos de clientes, analizarlos y obtener información para que los analistas puedan tomar decisiones. Para lograrlo, necesitas guardar los datos en una estructura que pueda manejar grandes volúmenes y recuperarlos de la forma más rápida posible. ¿Qué solución cumple estos requisitos?
   - A. Amazon EMR con HDFS
   - B. Amazon S3 y una función Lambda
   - C. Amazon DynamoDB y una función Lambda
   - D. Amazon Redshift y Amazon Athena

5. Te pidieron rediseñar una solución para reducir la carga operativa y usar servicios de AWS que detecten anomalías en datos de transacciones y asignen puntajes de anomalía a los registros maliciosos. Los registros se transmiten en tiempo real y se guardan en un data lake de Amazon S3 para su procesamiento y análisis. ¿Cuál es la solución más eficiente?
   - A. Amazon Data Firehose para transmitir los datos de transacciones y la función RANDOM_CUT_FOREST de Amazon Managed Service for Apache Flink para detectar anomalías
   - B. Amazon Data Firehose para transmitir los datos de transacciones a Amazon S3, con la función RANDOM_CUT_FOREST de SageMaker para detectar anomalías
   - C. Amazon Kinesis Data Stream para transmitir los datos de transacciones y la función RANDOM_CUT_FOREST de Amazon Managed Service for Apache Flink para detectar anomalías
   - D. Amazon Kinesis Data Stream para transmitir los datos de transacciones a Amazon S3, con la función RANDOM_CUT_FOREST de SageMaker para detectar anomalías

6. Te pidieron mejorar el tiempo de ingesta y almacenamiento de datos de geolocalización en Amazon Redshift para hacer analítica casi en tiempo real. ¿Cuál es la solución más económica?
   - A. Amazon Kinesis Data Stream para ingerir los datos de geolocalización. Cargar los datos de streaming en el clúster de Amazon Redshift con Amazon Redshift Streaming Ingestion.
   - B. Amazon Managed Streaming for Apache Kafka para ingerir los datos de geolocalización. Cargar los datos de streaming en el clúster de Amazon Redshift con Amazon Redshift Spectrum.
   - C. Amazon Data Firehose para ingerir los datos de geolocalización. Cargar los datos de streaming en el clúster de Amazon Redshift con Amazon Redshift Streaming Ingestion.
   - D. Amazon Managed Service for Apache Flink para ingerir los datos de geolocalización. Cargar los datos de streaming en el clúster de Amazon Redshift con Amazon Redshift Streaming Ingestion.

7. Estás migrando a AWS una solución de análisis de datos. La aplicación produce los datos como archivos CSV casi en tiempo real. Necesitas una solución que convierta los datos a Apache Parquet antes de guardarlos en un bucket de S3. ¿Cuál es la solución más eficiente?
   - A. Amazon Kinesis Data Streams y crear un job de ETL de streaming de AWS Glue que convierta los datos a Apache Parquet
   - B. Amazon Managed Streaming for Apache Kafka y una función Lambda
   - C. Amazon Data Firehose y una función Lambda
   - D. Amazon Managed Service for Apache Flink y una función Lambda

8. Usas Amazon Data Firehose para ingerir registros de datos desde on-premises. Los registros están comprimidos con GZIP. ¿Cómo puedes ejecutar eficientemente consultas SQL sobre el flujo de datos para obtener información en tiempo real y reducir la latencia de las consultas?
   - A. Amazon Managed Service for Apache Flink y una función Lambda
   - B. Amazon Kinesis Data Streams, una función Lambda y Amazon OpenSearch
   - C. Amazon Managed Streaming for Apache Kafka y una función Lambda
   - D. Amazon Kinesis Data Streams, una función Lambda y Amazon Redshift

9. Tu equipo está entrenando un modelo de reconocimiento de imágenes a gran escala que requiere alto throughput y acceso de baja latencia a un dataset guardado en Amazon S3. ¿Qué servicio de almacenamiento optimizaría mejor el rendimiento del entrenamiento en Amazon SageMaker?
   - A. Amazon S3
   - B. Amazon EFS
   - C. Amazon FSx for Lustre
   - D. Amazon FSx for Windows File Server

10. Necesitas una solución económica para guardar, y consultar con frecuencia, una gran cantidad de datos de sensores para un proyecto de analítica de IoT en Amazon SageMaker. ¿Qué servicio de almacenamiento deberías elegir?
    - A. Amazon FSx for Lustre
    - B. Amazon EFS
    - C. Amazon S3
    - D. Amazon FSx for OpenZFS

> [!info] Términos de las preguntas que no aparecen en el capítulo
> - **Amazon EMR** (pregunta 4) es el servicio administrado de AWS para ejecutar clústeres de Hadoop y Spark; con **HDFS** (ver la sección de DataSync), los datos viven en los discos de los nodos del clúster.
> - **Amazon Redshift Streaming Ingestion** (pregunta 6) permite que Redshift lea directamente de Kinesis Data Streams o de MSK, sin pasar por S3. **Amazon Redshift Spectrum** permite que Redshift consulte archivos en S3 sin cargarlos.
> - **RANDOM_CUT_FOREST** (pregunta 5) es un algoritmo de detección de anomalías. Existe como algoritmo integrado de SageMaker (Tabla 2.1) y existía como función SQL de Kinesis Data Analytics.

> [!warning] Estado del servicio en las preguntas 3, 5 y 8
> Estas preguntas se escribieron cuando existía **Amazon Kinesis Data Analytics for SQL**, el servicio que ejecutaba SQL sobre flujos y ofrecía la función SQL `RANDOM_CUT_FOREST`. Ese servicio dejó de funcionar el 27 de enero de 2026. Si aparecen en el examen tal como están redactadas, respóndelas con la lógica del libro. En un diseño real de hoy, el SQL sobre flujos se hace con Flink SQL en Amazon Managed Service for Apache Flink, y la función `RANDOM_CUT_FOREST` de SQL ya no existe.

## Glosario

- **ACID.** Garantías de una transacción: atomicidad (todo o nada), consistencia, aislamiento entre transacciones simultáneas y durabilidad de lo confirmado.
- **ACL (lista de control de acceso).** Lista asociada a un archivo o carpeta que indica qué usuarios o grupos pueden leerlo o modificarlo.
- **Active Directory (AD).** Servicio de directorio de Microsoft que centraliza los usuarios, grupos y equipos de una organización y los autentica.
- **Agente de DataSync.** Máquina virtual de AWS que se instala dentro de la red de origen para leer los datos y enviarlos a AWS.
- **Almacenamiento de archivos (file storage).** Sistema de archivos que vive en un servidor y que varias máquinas usan a la vez por la red (EFS, FSx).
- **Almacenamiento de bloques (block storage).** Disco «en crudo», dividido en bloques de tamaño fijo, que una máquina formatea con su propio sistema de archivos (EBS).
- **Almacenamiento de objetos (object storage).** Espacio plano de claves, cada una con su objeto completo y metadatos, al que se accede por HTTP (S3).
- **Almacenamiento de streaming.** Registro ordenado de eventos, solo de escritura al final, que los conserva un tiempo para que varios consumidores los lean.
- **Alta disponibilidad.** Capacidad de seguir funcionando aunque falle un servidor o una zona, gracias a copias y servidores de reserva.
- **Amazon Athena.** Motor SQL serverless que consulta archivos en S3 y cobra por datos escaneados.
- **Amazon Aurora.** Motor relacional de AWS, compatible con MySQL y PostgreSQL, con almacenamiento distribuido en tres zonas.
- **Amazon Data Firehose.** Servicio totalmente administrado que recibe flujos de datos y los entrega, con búfer y transformaciones opcionales, a un destino.
- **Amazon DynamoDB.** Base de datos NoSQL serverless de clave-valor y documentos, con latencia de un solo dígito de milisegundos.
- **Amazon EBS (Elastic Block Store).** Almacenamiento de bloques de AWS: volúmenes que se conectan a instancias EC2 de una misma zona.
- **Amazon EFS (Elastic File System).** Sistema de archivos NFS serverless y elástico que muchas máquinas montan a la vez.
- **Amazon EMR.** Servicio administrado para ejecutar clústeres de Hadoop y Spark.
- **Amazon FSx.** Familia de sistemas de archivos administrados construidos sobre tecnologías de terceros: Lustre, Windows File Server, NetApp ONTAP y OpenZFS.
- **Amazon Kinesis Data Streams.** Servicio de streaming que almacena eventos en shards durante un periodo de retención para que varios consumidores los lean en tiempo real.
- **Amazon Managed Service for Apache Flink.** Servicio administrado para ejecutar aplicaciones de Apache Flink (antes, Kinesis Data Analytics).
- **Amazon MSK.** Servicio administrado de Apache Kafka, en modalidad aprovisionada o serverless.
- **Amazon RDS.** Servicio administrado de bases de datos relacionales con ocho motores.
- **Amazon Redshift.** Data warehouse de AWS, optimizado para consultas analíticas SQL.
- **Amazon WorkSpaces.** Servicio de escritorios virtuales de AWS.
- **Apache Avro.** Formato binario por filas que guarda el esquema junto con los datos y admite evolución de esquema.
- **Apache Flink.** Framework de código abierto para procesar flujos con estado y garantías exactly-once.
- **Apache Kafka.** Plataforma distribuida de código abierto para almacenar y transmitir flujos de eventos en topics particionados.
- **Arquitectura orientada a eventos.** Diseño en que los componentes publican hechos y otros reaccionan a ellos, sin llamarse directamente.
- **Arreglo de almacenamiento.** Equipo empresarial con muchos discos y software propio que sirve almacenamiento a los servidores de un centro de datos.
- **Artefacto de modelo.** Archivos resultantes del entrenamiento (pesos, parámetros) que se usan para desplegar el modelo.
- **Autoadministrado (self-managed).** Software que instalas y operas tú mismo, por ejemplo en EC2, en lugar de un servicio administrado.
- **Autosupervisado (aprendizaje).** Aprendizaje a partir de datos sin etiquetar prediciendo partes del propio dato, como la siguiente palabra.
- **AWS Backup.** Servicio centralizado de copias de seguridad de AWS.
- **AWS DataSync.** Servicio de transferencia de archivos y objetos entre sistemas de almacenamiento on-premises, otras nubes y AWS.
- **AWS Glue.** Servicio serverless de integración de datos: catálogo, crawlers y trabajos ETL con Spark.
- **AWS Glue Data Catalog.** Registro central de esquemas y ubicaciones que convierte archivos de S3 en tablas consultables.
- **AWS Lambda.** Servicio de funciones serverless que ejecuta tu código ante cada evento y cobra por milisegundo.
- **AWS Outposts.** Bastidores de hardware de AWS, administrados por AWS, instalados en el centro de datos del cliente.
- **Bloqueo de archivos (file locking).** Mecanismo que marca un archivo como «en uso» para evitar escrituras simultáneas que lo corrompan.
- **Broker.** Servidor de Kafka que guarda particiones y atiende a productores y consumidores.
- **Bucket.** Contenedor de objetos de S3 con nombre único.
- **Búfer (Firehose).** Acumulación de datos hasta un tamaño o tiempo máximo antes de entregarlos al destino.
- **Caché intermedia (buffer cache).** Almacenamiento rápido y temporal que guarda cerca del consumidor los datos usados recientemente.
- **Capa de abstracción.** Capa intermedia que oculta los detalles de la que está debajo.
- **Captura de cambios de datos (CDC).** Técnica que publica como evento cada inserción, actualización o borrado de una base de datos.
- **Carga de trabajo (workload).** Aplicación o proceso que se ejecuta en la nube, con su patrón de uso de recursos.
- **Carga diferida (lazy loading).** Traer cada dato desde su origen solo la primera vez que se lee.
- **Centro de datos.** Instalación con servidores, red, energía y refrigeración propios.
- **Checkpoint / offset.** Marca de hasta dónde se procesó un flujo, para reanudar sin perder ni duplicar datos.
- **Ciclo de vida (políticas o gestión).** Reglas que mueven datos a clases más baratas, o los borran, tras cierto tiempo sin uso.
- **Clase de almacenamiento (S3).** Nivel de precio y de acceso que se asigna a cada objeto según la frecuencia con que se lee.
- **Clave de partición.** Valor de cada registro que decide en qué shard o partición se guarda.
- **Clickstream.** Secuencia de clics y páginas vistas de cada visitante de un sitio web.
- **Clon (ZFS).** Copia escribible creada al instante a partir de un snapshot.
- **CloudTrail (AWS CloudTrail).** Registro de todas las llamadas a la API de una cuenta de AWS: quién, qué, cuándo y desde dónde.
- **CloudWatch (Amazon CloudWatch).** Servicio de monitoreo de métricas, logs y alarmas de AWS.
- **Clúster.** Grupo de servidores que funcionan juntos como un solo sistema.
- **CMS (sistema de gestión de contenidos).** Software para publicar y administrar el contenido de un sitio web, como WordPress.
- **Codificación (columnar).** Representación compacta de los valores de una columna, como la codificación por diccionario o por longitud de corrida.
- **Consistencia eventual.** Modelo en que una lectura hecha justo después de una escritura puede devolver, por un instante, el valor anterior.
- **Consistencia fuerte.** Garantía de que, tras una escritura, toda lectura posterior ve el dato nuevo.
- **Consulta continua.** Consulta sobre un flujo que no termina y actualiza su resultado con cada evento.
- **Contenedor.** Aplicación empaquetada con todas sus dependencias (por ejemplo, con Docker).
- **Continuidad del negocio y recuperación ante desastres (BCDR).** Capacidad de seguir operando durante un incidente y de restaurar los sistemas después de una catástrofe.
- **Controlador (Kafka).** Nodo que administra el estado de particiones y réplicas del clúster.
- **Coordinación distribuida.** Lograr que varias máquinas acuerden hechos compartidos (quién es líder, qué nodos están vivos) pese a las fallas.
- **Copia en escritura (copy-on-write).** Técnica de ZFS que escribe cada versión nueva en otro sitio en lugar de sobrescribir.
- **Cost Explorer (AWS Cost Explorer).** Herramienta para analizar el gasto de AWS por servicio, etiqueta y periodo.
- **Costo total de propiedad (TCO).** Suma de todos los costos de una solución a lo largo del tiempo, no solo el precio por GB.
- **Crawler (AWS Glue).** Proceso que recorre archivos, deduce su esquema y lo registra en el catálogo.
- **Data lake.** Repositorio central de datos crudos en su formato original.
- **Data warehouse.** Base de datos analítica con esquema definido.
- **Dataset acotado / flujo no acotado.** Datos finitos (un archivo) frente a un flujo de eventos que no termina.
- **Datos fríos (cold data).** Datos que casi nunca se consultan pero deben conservarse.
- **Deduplicación.** Guardar una sola copia de bloques o fragmentos idénticos y referencias a ella.
- **DevOps.** Prácticas y rol que unen el desarrollo de software con la operación de la infraestructura.
- **Direct PUT (Firehose).** Envío de registros a Firehose directamente desde tu aplicación mediante la API.
- **Directorio personal (home directory).** Carpeta de red propia de cada empleado, disponible desde cualquier equipo.
- **Disponibilidad.** Porcentaje del tiempo en que un servicio puede usarse; 99.99 % equivale a unos 53 minutos al año sin servicio.
- **Durabilidad.** Probabilidad de no perder un dato guardado; S3 está diseñado para once nueves.
- **E/S (entrada/salida, I/O).** Operaciones de lectura y escritura sobre el almacenamiento.
- **ECS / EKS.** Servicios de AWS para ejecutar contenedores; EKS es Kubernetes administrado.
- **EFA (Elastic Fabric Adapter).** Interfaz de red de AWS para cargas de HPC y ML con muy alto throughput entre nodos.
- **Endpoint de VPC.** Puerta de entrada privada desde tu VPC a un servicio de AWS, sin pasar por internet.
- **Endpoint HTTP.** Dirección de un servicio que recibe datos mediante peticiones HTTP.
- **Escalado automático (auto scaling).** Ajuste automático de la capacidad (instancias, réplicas, workers) según métricas de carga.
- **ETL / ETL de streaming.** Extraer, transformar y cargar datos; en streaming, evento a evento y de forma continua.
- **Event sourcing.** Patrón que guarda el estado como la secuencia completa de eventos que lo produjeron.
- **Evolución de esquema.** Capacidad de leer datos escritos con un esquema distinto del que espera el lector.
- **Exabyte.** Un millón de terabytes.
- **Exactly-once.** Garantía de que cada evento afecta al resultado una sola vez, aunque haya reprocesamientos.
- **Flash.** Almacenamiento en chips de memoria, como los SSD.
- **Formato columnar.** Formato que guarda juntos los valores de cada columna (Parquet, ORC); eficiente para leer pocas columnas.
- **Formato por filas.** Formato que guarda juntos los valores de cada registro (Avro, CSV); eficiente para escribir registro a registro.
- **FSx for Lustre.** Sistema de archivos paralelo administrado, para Linux, vinculable a S3 e integrado con SageMaker.
- **FSx for NetApp ONTAP.** Almacenamiento administrado de NetApp con NFS, SMB, iSCSI y NVMe, y replicación SnapMirror.
- **FSx for OpenZFS.** Sistema de archivos OpenZFS administrado, accesible por NFS v3 a v4.2, con snapshots y clones.
- **FSx for Windows File Server.** Servidor de archivos Windows administrado, accesible por SMB e integrado con Active Directory.
- **Geo-fencing.** Perímetro virtual en un mapa que dispara un evento cuando un dispositivo entra o sale.
- **Grupo de seguridad.** Firewall virtual de un recurso de AWS que define qué tráfico se permite.
- **Hadoop.** Plataforma de código abierto de big data, anterior a Spark, de la que forma parte HDFS.
- **HDFS.** Sistema de archivos distribuido de Hadoop que reparte bloques de 128 MB, con tres copias, entre los nodos.
- **HIPAA.** Ley estadounidense que protege la información de salud de los pacientes.
- **HPC (computación de alto rendimiento).** Muchos servidores trabajando en paralelo sobre un mismo problema de cálculo.
- **IAM (rol de IAM).** Servicio de permisos de AWS; un rol es una identidad con permisos que un servicio o persona asume temporalmente.
- **Inferencia (en ML).** Uso de un modelo entrenado para predecir sobre datos nuevos.
- **Intelligent-Tiering (S3).** Clase que mueve cada objeto entre niveles de precio según su acceso.
- **IOPS.** Operaciones de lectura o escritura por segundo; un disco mecánico da unas 100 a 200.
- **IoT (Internet de las cosas).** Dispositivos físicos con sensores y conexión a la red que envían datos continuamente.
- **iSCSI.** Protocolo que transporta comandos de disco por una red IP, de modo que un servidor ve un volumen remoto como disco local.
- **JDBC.** Estándar de Java para conectarse a bases de datos mediante controladores específicos de cada motor.
- **JSON Lines.** Archivo de texto con un objeto JSON por línea.
- **KRaft.** Modo de Kafka que gestiona los metadatos con controladores propios (algoritmo Raft), sin ZooKeeper.
- **Kubernetes.** Orquestador de contenedores que decide dónde corre cada uno y lo reinicia si falla.
- **Latencia.** Tiempo entre pedir un dato y empezar a recibirlo.
- **Legacy (sistema heredado).** Sistema antiguo que sigue en uso aunque esté desfasado.
- **LibSVM.** Formato de texto para datos dispersos: `etiqueta índice:valor ...`.
- **Líder-seguidor (réplicas).** Esquema en que una réplica recibe las escrituras y las demás la copian para tomar el relevo si falla.
- **Lift and shift.** Migrar una aplicación a la nube sin modificarla, cambiando solo dónde corre.
- **LLM (modelo de lenguaje grande).** Red neuronal con miles de millones de parámetros que genera texto.
- **Lustre.** Sistema de archivos paralelo de código abierto usado en supercómputo.
- **Metadatos.** Datos sobre los datos: nombres, tipos, ubicaciones, permisos.
- **Microlote (micro-batch).** Procesamiento de un flujo en pequeños lotes cada pocos segundos o minutos.
- **Migración de datos.** Traslado a gran escala de datos de un sistema de almacenamiento a otro.
- **Modelo clave-valor / de documentos.** Acceso a cada elemento por una clave única / elementos con estructura anidada tipo JSON que puede variar.
- **Montar.** Conectar un sistema de archivos remoto en una carpeta local para usarlo como si fuera un disco propio.
- **Motor de base de datos.** Software que implementa la base de datos, como PostgreSQL, MySQL u Oracle.
- **MSK Serverless.** Modalidad de MSK que aprovisiona y escala la capacidad automáticamente.
- **NAS (almacenamiento conectado a la red).** Equipo que sirve archivos por la red mediante NFS o SMB.
- **Nativo de la nube (cloud native).** Aplicación diseñada desde el principio para aprovechar servicios administrados y escalar agregando máquinas.
- **NetApp.** Fabricante estadounidense de almacenamiento empresarial.
- **NFS (Network File System).** Protocolo estándar de Linux y Unix para compartir archivos; v3 es sin estado, v4.x con estado y con bloqueos integrados.
- **Nodo.** Cada servidor de un clúster.
- **NoSQL.** Bases de datos no relacionales, sin esquema tabular fijo ni joins, que escalan repartiendo datos por clave.
- **Nube híbrida.** Arquitectura en que una parte corre on-premises y otra en la nube.
- **NVMe / NVMe-over-TCP.** Protocolo moderno para SSD de baja latencia; su variante sobre TCP lo lleva por una red Ethernet estándar.
- **Observabilidad.** Capacidad de entender el estado de un sistema a partir de sus logs, métricas y trazas.
- **On-premises.** Infraestructura que la empresa opera en sus propios centros de datos.
- **ONTAP.** Sistema operativo de los equipos de almacenamiento de NetApp.
- **OpenZFS.** Continuación de código abierto de ZFS, usada en Linux y FreeBSD.
- **Pago por uso (pay-as-you-go).** Modelo de precios de la nube en que se paga por lo consumido, sin inversión inicial.
- **Partición (Kafka).** Subdivisión ordenada de un topic que permite repartir la carga entre brokers.
- **Particionar.** Dividir los datos por una clave para que cada consulta lea solo lo necesario.
- **Patrón de acceso a los datos.** Descripción de cómo se escriben, consultan y recuperan los datos (filtros, orden, frecuencia).
- **PCI DSS.** Estándar de seguridad obligatorio para quien guarda o procesa datos de tarjetas de pago.
- **Perfil de usuario (Windows).** Carpeta con el escritorio, la configuración y los documentos de un usuario.
- **Periodo de retención.** Tiempo durante el que un servicio de streaming conserva los eventos antes de que expiren.
- **Petabyte.** Mil terabytes.
- **Pipeline.** Cadena automatizada de pasos por la que pasan los datos.
- **Plano de control / plano de datos.** Operaciones que administran la infraestructura frente a operaciones sobre los datos mismos.
- **POSIX.** Estándar de interfaces de los sistemas tipo Unix, incluidas las operaciones y los permisos de archivos.
- **Procedimiento almacenado.** Programa guardado y ejecutado dentro de la base de datos.
- **Productor / consumidor.** Programa que escribe en un flujo / programa que lee de él.
- **Protocol Buffers (protobuf).** Formato binario de serialización, compacto y tipado, creado por Google.
- **Protocolo.** Reglas con que dos computadoras se comunican por la red.
- **Puntos de montaje (EFS mount targets).** Interfaces de red que EFS coloca en tus subredes para que las instancias lo monten.
- **RDS Custom.** Variante de RDS con acceso de administrador al sistema operativo, para Oracle y SQL Server.
- **RecordIO / RecordIO-protobuf.** Formato de secuencia de registros con su longitud antepuesta; en SageMaker, con contenido protobuf.
- **Recuperación (retrieval).** Restauración previa necesaria para leer objetos de las clases de archivo de S3 Glacier.
- **Refactorizar.** Reestructurar código sin cambiar lo que hace.
- **Replay.** Volver a leer eventos pasados de un flujo mientras sigan retenidos.
- **Réplica de lectura.** Copia de solo lectura de una base de datos que reparte la carga de consultas.
- **Residencia de datos.** Obligación de que ciertos datos permanezcan en un país o región.
- **Retraso put-to-get.** Tiempo entre que un registro entra a un stream y puede leerse; en Kinesis Data Streams, normalmente menos de un segundo.
- **Reverse ETL.** Devolver datos procesados a los sistemas operativos de la empresa, como el CRM.
- **Right-sizing.** Elegir el número y el tamaño correctos de servidores para una carga.
- **RTO / RPO.** Tiempo máximo aceptable para restaurar el servicio / cantidad máxima de datos que se acepta perder, medida en tiempo.
- **S3 (Amazon Simple Storage Service).** Almacenamiento de objetos de AWS.
- **S3 Express One Zone.** Clase de S3 de latencia de un solo dígito de milisegundos, en una sola zona.
- **SAN (red de área de almacenamiento).** Red dedicada por la que los servidores acceden a almacenamiento de bloques remoto.
- **Savings Plans / instancias reservadas.** Compromisos de uso de uno o tres años a cambio de descuento.
- **Scale-out / scale-up.** Crecer agregando servidores / reemplazando un servidor por uno más grande.
- **Scan / Query (DynamoDB).** Leer la tabla completa / leer solo los elementos de una clave de partición.
- **Scratch / persistente (FSx for Lustre).** Sistema sin replicación para trabajos cortos / sistema replicado para largo plazo.
- **Semántica de sistema de archivos.** Operaciones que un programa espera de los archivos: modificar en el lugar, renombrar, bloquear, permisos.
- **Semisupervisado (aprendizaje).** Combinación de pocos datos etiquetados con muchos sin etiquetar.
- **Serialización.** Conversión de un objeto en memoria a bytes para guardarlo o enviarlo.
- **Serverless.** Modelo en que no se ven ni administran servidores; el servicio asigna recursos y cobra por uso.
- **Servidor de archivos (file server).** Computadora dedicada a guardar archivos y compartirlos por la red.
- **Shard.** Unidad de capacidad de un stream de Kinesis.
- **Sharding.** División horizontal de una base de datos entre varios servidores según una clave.
- **SIEM.** Plataforma que centraliza y correlaciona registros de seguridad para detectar ataques.
- **Sistema de archivos.** Capa del sistema operativo que organiza los bloques de un disco en carpetas y archivos con nombre y permisos.
- **Sistema de archivos distribuido.** Sistema de archivos cuyos datos están repartidos entre muchos servidores pero se ven como un solo árbol.
- **Sistema de registro (system of record).** Fuente autorizada de un dato.
- **Sistema de reserva (standby).** Copia lista para tomar el relevo si falla el sistema principal.
- **SMB (Server Message Block).** Protocolo de carpetas compartidas de Windows (`\\servidor\carpeta`).
- **SnapMirror.** Replicación nativa de NetApp entre sistemas ONTAP.
- **Snapshot.** Copia de un volumen o sistema de archivos en un instante dado; en EBS, incremental.
- **Snow (familia).** Dispositivos físicos de AWS para transferir datos por mensajería; Snowcone descontinuado y Snowball Edge solo para clientes existentes.
- **Spark (Apache Spark).** Motor de procesamiento distribuido de big data; PySpark es su API en Python.
- **Spot (instancias).** Capacidad sobrante de EC2 con gran descuento que AWS puede recuperar con dos minutos de aviso.
- **Subred.** Rango de direcciones de una VPC que vive en una sola zona de disponibilidad.
- **Throughput.** Cantidad de datos transferidos por segundo; un SSD NVMe de laptop lee de 3 a 7 GB/s.
- **Tolerancia a fallas.** Capacidad de seguir dando resultados correctos aunque fallen máquinas.
- **Topic (Kafka).** Registro de eventos con nombre, ordenado y persistente.
- **Totalmente administrado (fully managed).** Servicio cuya infraestructura opera AWS; tú lo configuras y pagas por uso.
- **Transaccional (carga de trabajo).** Muchas operaciones pequeñas en posiciones dispersas, como las de una base de datos de pedidos; limitada por IOPS.
- **Transcodificación / renderizado.** Convertir un video a otro formato o resolución / generar los fotogramas finales de una escena 3D.
- **URI.** Identificador único de un recurso, como `s3://bucket/clave`.
- **Validación de integridad.** Comparación de sumas de verificación del origen y el destino para confirmar que una copia es idéntica.
- **VPC (nube privada virtual).** Red privada y aislada dentro de AWS.
- **WAF (AWS Web Application Firewall).** Firewall que filtra el tráfico web según reglas agrupadas en web ACL.
- **Well-Architected Framework.** Guía de buenas prácticas de AWS organizada en seis pilares, entre ellos costo y seguridad.
- **ZFS.** Sistema de archivos y gestor de volúmenes con copia en escritura, sumas de verificación, snapshots, clones y compresión.
- **Zona de disponibilidad (AZ).** Uno o varios centros de datos independientes dentro de una región, aislados de las demás zonas.
- **ZooKeeper (Apache ZooKeeper).** Servicio de coordinación distribuida que Kafka usaba para sus metadatos antes de KRaft.
