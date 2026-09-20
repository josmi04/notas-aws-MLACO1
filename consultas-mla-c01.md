# Paquete de consultas para las notas del MLA-C01

Este archivo tiene cuatro partes. Las tres primeras se pegan **una sola vez** en las instrucciones del proyecto de Claude. La cuarta son las 29 consultas: pegas una por conversación, en orden.

Cómo se usa, paso a paso:

1. Copia la **Parte 1**, la **Parte 2** y la **Parte 3** en las instrucciones del proyecto.
2. Crea un archivo vacío llamado `glosario-mla.md` y súbelo al conocimiento del proyecto.
3. Abre una conversación nueva y pega la consulta de la nota 1.
4. Claude te devuelve primero un esquema. Contesta «ok, redacta» o pide cambios.
5. Al terminar la nota, pega el mensaje de cierre de la Parte 3. Reemplaza `glosario-mla.md` en el proyecto con la versión que te devuelva.
6. Repite con la siguiente consulta.

No necesitas entender el contenido técnico de las consultas para usarlas. Están escritas para Claude, no para ti.

---

## Parte 1 — Instrucciones del proyecto

```text
PROYECTO: notas de estudio para AWS Certified Machine Learning Engineer – Associate (MLA-C01).

OBJETIVO
Aprobar el MLA-C01 y, además, adquirir oficio real de Machine Learning Engineer. Las notas se
ajustan al alcance del temario MLA-C01 (guía de examen versión 1.0), pero enseñan el uso práctico
de los servicios con AWS CLI y boto3 (y el SageMaker Python SDK cuando sea lo natural).

SKILL
Usa el skill notas-de-programacion. Donde este documento contradiga al skill, manda este documento.
En particular, el perfil de lector por defecto del skill NO aplica; aplica el de abajo.

LECTOR
- Domina estadística y Machine Learning a nivel avanzado. No definas algoritmos, hiperparámetros,
  métricas, regularización ni validación. Usa esa teoría como palanca.
- En AWS tiene nivel teórico de Cloud Practitioner: conoce regiones, zonas de disponibilidad, IAM,
  EC2 y S3 como conceptos, el modelo de responsabilidad compartida y los principios de
  escalabilidad y alta disponibilidad. NO ha operado AWS de forma extensa: no sabe escribir una
  política IAM, configurar una VPC ni lanzar un job de SageMaker.
- Todo lo definido en notas anteriores (ver glosario-mla.md) se da por sabido. Se recuerda en una
  línea cuando haga falta, sin redefinirlo.

RIGOR
- Prohibido usar un término, servicio, componente o abreviatura antes de definirlo o sin definirlo
  en su primera aparición. Si un término pertenece a una nota posterior, aparece solo como
  [[puntero]], nunca como uso.
- Las notas posteriores a la actual (ver la secuencia en el glosario) están fuera del corte.
- Nunca inventes nombres de parámetros, métodos, campos de API ni límites numéricos. Verifícalos
  con búsqueda en la documentación oficial de AWS. Si no puedes verificar algo, omítelo y dilo.
- Fija en el frontmatter las versiones de boto3, AWS CLI y SageMaker Python SDK usadas.

CÓDIGO
- Ejemplos realistas, dentro del caso conductor (Parte 2), en un flujo plausible de ML/MLOps.
- Cada ejemplo dice antes del código: qué problema resuelve, desde dónde se ejecuta (laptop con
  CLI configurada, notebook de Studio, CodeBuild, Lambda...), con qué identidad y qué recursos
  deben existir ya. Después del código: glosa de cada parte relevante y resultado esperado
  (salida real o su forma, estado final del recurso).
- Incluye los fallos más probables (AccessDenied, ValidationException, ResourceLimitExceeded,
  throttling...) con el mensaje o la forma del error y su causa.
- CLI y boto3 cuando ambos aporten; no dupliques mecánicamente un ejemplo en los dos.

DECISIONES
Toda alternativa entre servicios o configuraciones se explica con: razón práctica, ventajas,
limitaciones y escenario donde conviene. Prioriza las distinciones que el examen evalúa.

ALCANCE Y PROFUNDIDAD
- Profundidad de ML Engineer. No te desvíes hacia detalles de Data Engineer, Solutions Architect
  o administrador de infraestructura, salvo que expliquen el comportamiento del servicio o una
  pregunta de examen.
- Sin teoría de ML conocida. La extensión se reparte según la dificultad real del tema en AWS.
- Unas 15 páginas por nota, incluidas las preguntas, salvo que la consulta indique otra cifra.

SERVICIOS QUE CAMBIARON
Enseña lo que evalúa MLA-C01. Si un servicio, nombre o API cambió después, añade un aviso breve
con el estado actual verificado por búsqueda (por ejemplo: Data Wrangler dentro de SageMaker
Canvas, Studio Classic vs Studio, Experiments y MLflow gestionado, CodeCommit sin clientes nuevos,
Kinesis Data Firehose renombrado a Amazon Data Firehose, Kinesis Data Analytics renombrado a
Managed Service for Apache Flink, interfaces nuevas del SageMaker Python SDK). No introduzcas
contenido propio de MLA-C02.

PREGUNTAS
Al final de cada nota, 10 preguntas de práctica al estilo AWS con solución razonada:
- 5 de dificultad media, comparables a preguntas normales del examen.
- 5 de dificultad alta: alternativas plausibles, escenarios, errores de configuración que detectar,
  elección de la solución más adecuada.
Mezcla los formatos que usa el examen: opción múltiple (1 correcta de 4), respuesta múltiple
(2+ correctas de 5+), ordenamiento, emparejamiento y al menos un mini caso de estudio con dos
preguntas sobre el mismo escenario. Cada solución explica por qué la correcta gana y por qué cada
distractor falla. Las preguntas solo usan lo definido hasta esta nota.

FLUJO EN DOS PASOS
Cuando te pida una nota, NO la redactes de inmediato. Primero entrega:
1. Encabezados propuestos (nombrados por su contenido).
2. Por sección: términos que define y ejemplos de código que tendrá.
3. Decisiones o distinciones de examen que cubrirás (añade las que la consulta no mencione).
4. Reparto aproximado de páginas.
5. Cualquier término que la consulta necesite y que no esté en el glosario ni en la nota
   (conflicto de corte), con tu propuesta para resolverlo.
Espera mi «ok, redacta». Luego escribe la nota como archivo .md.
```

---

## Parte 2 — Caso conductor

```text
CASO CONDUCTOR (todos los ejemplos ocurren aquí; reutiliza nombres y recursos entre notas)

Empresa: Kanan Financiera, fintech que emite tarjetas y opera una red de cajeros automáticos.
Región principal: us-east-1.
Cuentas AWS:
- kanan-ml-dev  (111111111111): experimentación y entrenamiento.
- kanan-ml-prod (222222222222): endpoints y monitoreo de producción.
Convención de buckets: kanan-ml-<entorno>-<propósito>-us-east-1
  (p. ej. kanan-ml-dev-raw-us-east-1, kanan-ml-dev-curated-us-east-1,
  kanan-ml-dev-artifacts-us-east-1).
Convención de roles: KananSageMakerExecutionRole-<entorno>, KananGlueRole-<entorno>, etc.
Equipo: dos ML engineers, un data engineer, un responsable de cumplimiento.

Problemas de ML:
1. FRAUDE (eje principal). Clasificar transacciones de tarjeta como fraude o no.
   Datos: transacciones que llegan en streaming (~2,000/s en hora pico), historial en una base
   RDS PostgreSQL, perfil de cliente en DynamoDB. Contiene PII. Clases muy desbalanceadas.
   Requisito: respuesta en < 100 ms por transacción.
2. DOCUMENTOS KYC. Clasificar y extraer datos de identificaciones y comprobantes de domicilio
   subidos como imágenes/PDF. Contiene PII. Requiere revisión humana en casos dudosos.
3. EFECTIVO EN CAJEROS. Pronosticar retiros diarios por cajero (≈3,000 series).
   Se calcula una vez por noche; nadie espera la respuesta en línea.

Restricción regulatoria: los datos de clientes no pueden salir de la región elegida y todo debe
estar cifrado con llaves administradas por Kanan.

No es necesario usar los tres problemas en cada nota: usa el que haga el ejemplo más natural.
```

---

## Parte 3 — Glosario y mensaje de cierre

Formato inicial de `glosario-mla.md` (súbelo así, vacío):

```markdown
# Glosario acumulado MLA-C01

## Secuencia de notas
1 IAM y acceso programático · 2 Anatomía de un job de SageMaker · 3 Almacenamiento y formatos ·
4 Ingesta en streaming · 5 Catálogo y transformación a escala · 6 Preparación y calidad ·
7 Feature Store · 8 Etiquetado · 9 Protección de datos · 10 Servicios de IA · 11 Algoritmos
integrados · 12 Modelos fundacionales · 13 Script mode · 14 Entrenar más rápido y barato ·
15 AMT · 16 Debugger · 17 Clarify · 18 Experimentos y Model Registry · 19 Opciones de inferencia ·
20 Contenedores y hosting múltiple · 21 Optimización y edge · 22 Escalado y despliegue ·
23 Red · 24 IaC · 25 Orquestación · 26 CI/CD · 27 Model Monitor · 28 Observabilidad · 29 Costos

## Términos
| Término | Definición mínima | Nota |
|---|---|---|

## Recursos del caso conductor ya creados
| Recurso | Nombre | Nota |
|---|---|---|
```

Mensaje de cierre, que pegas al terminar cada nota:

```text
Actualiza glosario-mla.md: añade cada término, servicio, recurso o abreviatura que esta nota
definió (una línea de definición mínima cada uno) y cada recurso del caso conductor que se creó
o nombró. No borres nada de lo existente. Entrégame el archivo completo.
```

---

## Parte 4 — Las 29 consultas

Cada bloque se pega tal cual en una conversación nueva del proyecto.

### Bloque A — Fundamentos operativos

Este bloque va primero aunque el examen lo pone en el dominio 4, porque todo lo demás lo necesita.

```text
Nota 1/29 — Acceso programático a AWS e IAM para trabajo de ML

Tareas de la guía que cubre:
- 4.3 K: IAM roles, policies, and groups that control access to AWS services (IAM, bucket
  policies, SageMaker Role Manager)
- 4.3 S: Configuring least privilege access to ML artifacts
- 4.3 S: Configuring IAM policies and roles for users and applications that interact with ML systems
- 4.3 S: Troubleshooting and debugging security issues (solo la parte IAM)

Pregunta central: cómo se autentica y autoriza cada llamada que un ML engineer hace (desde su
laptop, desde un notebook y desde un servicio como SageMaker), y cómo se escriben y depuran los
permisos mínimos para eso.

Debe quedar claro:
- AWS CLI y boto3 desde cero: instalación, perfiles, credenciales, región, sts get-caller-identity;
  client vs resource en boto3; paginators y waiters; manejo de ClientError.
- Usuario vs grupo vs rol; política de identidad vs política de recurso (bucket policy);
  trust policy vs permission policy; asumir un rol (sts assume-role) y cuentas distintas
  (dev y prod del caso).
- Qué es un rol de ejecución de un servicio y por qué existe iam:PassRole; el error típico
  cuando falta.
- Evaluación de políticas: deny explícito, allow, deny implícito; por qué un bucket policy
  puede bloquear aunque IAM permita.
- Condiciones en políticas y cómo restringir a prefijos de S3 concretos.
- Managed policies de AWS vs políticas propias; AmazonSageMakerFullAccess y por qué no es
  mínimo privilegio. SageMaker Role Manager (personas) como atajo.
- Diagnóstico de AccessDenied: leer el mensaje, IAM Policy Simulator, CloudTrail como puntero.

Corte: ninguna nota previa. SageMaker solo se menciona como "servicio que necesita un rol";
su funcionamiento es la nota 2. KMS, VPC y CloudTrail en detalle quedan como punteros.
Fuera de alcance: IAM Identity Center en detalle, federación SAML, Organizations y SCPs
(solo una mención de qué son si aparece en una pregunta).
Caso: Kanan, roles para ML engineers en dev y el rol de ejecución de SageMaker en dev.
Peso: mucho a políticas, PassRole y diagnóstico; poco a instalación.
```

```text
Nota 2/29 — Anatomía de un job de SageMaker: qué ocurre cuando lanzas Processing o Training

Tareas de la guía que cubre:
- 2.2 S: Using SageMaker built-in algorithms and common ML libraries to develop ML models
  (solo la mecánica de lanzar el job)
- 1.2 K/S: Transforming data by using AWS tools (SageMaker Processing como herramienta)
- 3.2 S: Deploying and hosting models by using the SageMaker SDK (solo la idea de SDK vs boto3)

Pregunta central: qué es un job de SageMaker, qué recursos consume y produce, y cómo se lanza,
se sigue y se diagnostica desde boto3 y desde el SageMaker Python SDK.

Debe quedar claro:
- Qué es SageMaker como plataforma y qué es un dominio de Studio (solo lo necesario para saber
  desde dónde se ejecuta el código).
- Qué es un contenedor y una imagen, y que SageMaker ejecuta imágenes guardadas en ECR
  (ECR solo como registro; el detalle es la nota 20). Cómo obtener la URI de una imagen de AWS.
- Ciclo de vida de un job: create -> InProgress -> Completed/Failed/Stopped; describe-* y
  FailureReason; waiters.
- Estructura de CreateProcessingJob y CreateTrainingJob: rol, imagen, recursos (tipo y número
  de instancias, volumen), canales de entrada, salida a S3, StoppingCondition.
- Rutas /opt/ml/... dentro del contenedor y cómo se mapean a S3.
- Logs en CloudWatch Logs: grupo, stream, cómo leerlos por CLI/boto3.
- SDK (Estimator/Processor, o las interfaces nuevas si aplican) vs boto3: qué hace el SDK por ti
  y cuándo conviene cada uno.
- Errores típicos: ResourceLimitExceeded (cuotas), AccessDenied al leer S3, imagen en otra región.

Corte: nota 1. Los input modes (File/FastFile/Pipe) solo como puntero a la nota 3.
Script mode como puntero a la nota 13.
Caso: un Processing job que limpia un CSV de transacciones y un Training job de XGBoost
integrado sobre el resultado (sin explicar el algoritmo ni su configuración de datos en detalle).
Peso: mucho al ciclo de vida, a la estructura de la petición y al diagnóstico.
```

### Bloque B — Datos (dominio 1)

```text
Nota 3/29 — Almacenamiento y formatos de datos para entrenar en AWS

Tareas de la guía que cubre:
- 1.1 K: Data formats and ingestion mechanisms (validated and non-validated formats, Parquet,
  JSON, CSV, ORC, Avro, RecordIO)
- 1.1 K: How to use the core AWS data sources (S3, EFS, FSx for NetApp ONTAP)
- 1.1 K: AWS storage options, including use cases and tradeoffs
- 1.1 S: Extracting data from storage (S3, EBS, EFS, RDS, DynamoDB) by using relevant AWS
  service options (S3 Transfer Acceleration, EBS Provisioned IOPS)
- 1.1 S: Choosing appropriate data formats based on data access patterns
- 1.1 S: Troubleshooting data ingestion and storage issues that involve capacity and scalability
- 1.1 S: Making initial storage decisions based on cost, performance, and data structure
- 1.3 S: Configuring data to load into the model training resource (EFS, FSx)

Pregunta central: dónde viven los datos de entrenamiento, en qué formato, y cómo llegan al
contenedor del job con el menor tiempo y costo.

Debe quedar claro:
- S3 para ML: prefijos y rendimiento por prefijo, clases de almacenamiento (incl. Glacier) y
  cuándo importan, multipart upload, Transfer Acceleration (cuándo sí y cuándo no), sync con CLI.
- EBS (gp3 vs io2/Provisioned IOPS), EFS, FSx for Lustre (vinculado a S3) y FSx for NetApp ONTAP:
  qué problema resuelve cada uno en ML.
- Input modes de SageMaker: File, FastFile, Pipe, y EFS/FSx for Lustre como fuente del canal;
  qué requiere cada uno (VPC como puntero a la nota 23), tiempo de arranque y tamaño de disco.
- Formatos: columnar vs fila, compresión, esquema; RecordIO-protobuf y por qué existe en los
  algoritmos integrados; formatos "validados" vs "no validados".
- Extraer datos de RDS y DynamoDB hacia S3 para entrenar (exportaciones nativas vs consulta).
- DataSync y Storage Gateway: solo cuándo aparecen (datos on-premises).

Corte: notas 1–2. Glue/Athena como punteros a la nota 5.
Caso: histórico de transacciones de fraude (~500 GB) y consultas repetidas por columnas.
Peso: mucho a input modes y a elección de formato; poco a clases de S3.
```

```text
Nota 4/29 — Ingesta de datos en streaming para ML

Tareas de la guía que cubre:
- 1.1 K: How to use AWS streaming data sources to ingest data (Kinesis, Apache Flink, Apache Kafka)
- 1.2 K: Services that transform streaming data (Lambda, Spark)
- 1.1 S: Troubleshooting data ingestion issues that involve capacity and scalability
- 3.3 K: Automation and integration of data ingestion with orchestration services (solo la parte
  de ingesta)

Pregunta central: cómo entran eventos continuos (transacciones) a AWS, cómo se transforman al
vuelo y cómo terminan en S3 listos para entrenar o en un servicio que los consume en línea.

Debe quedar claro:
- Kinesis Data Streams: shards, capacidad por shard, modo on-demand vs provisioned, partition
  key y hot shards, ProvisionedThroughputExceededException, retención, consumidores.
- Amazon Data Firehose: entrega gestionada a S3 (y otros destinos), buffering, transformación con
  Lambda, conversión de formato a Parquet/ORC (requiere esquema; puntero a la nota 5).
- Managed Service for Apache Flink: cuándo hace falta procesamiento con estado/ventanas.
- Amazon MSK (Kafka): cuándo se elige frente a Kinesis.
- Lambda como transformador: límites relevantes, lotes, errores.
- Kinesis Video Streams: solo qué es y cuándo aparece.
- Tabla de decisión Data Streams vs Firehose vs Flink vs MSK.
- Productor de ejemplo con boto3 (put_records) y su manejo de fallos parciales.

Corte: notas 1–3.
Caso: ~2,000 transacciones/s de fraude hacia S3 particionado por fecha, más un consumidor en línea.
Peso: mucho a capacidad y diagnóstico de Data Streams y a la decisión entre servicios.
```

```text
Nota 5/29 — Catálogo y transformación de datos a escala: Glue, Athena, EMR y Lake Formation

Tareas de la guía que cubre:
- 1.1 S: Merging data from multiple sources (programming techniques, AWS Glue, Apache Spark)
- 1.2 K: Tools to explore, visualize, or transform data (AWS Glue)
- 1.2 S: Transforming data by using AWS tools (AWS Glue, Spark running on Amazon EMR)

Pregunta central: cómo se describen, consultan y transforman datasets grandes que viven en S3,
y qué servicio elegir para cada tipo de transformación.

Debe quedar claro:
- Glue Data Catalog (bases, tablas, particiones) y crawlers: qué hacen, cuándo fallan o infieren
  mal el esquema.
- Athena: SQL sobre S3, costo por datos escaneados, particiones, CTAS para convertir a Parquet.
- Glue ETL jobs (Spark gestionado): DynamicFrame vs DataFrame, tipos de worker, job bookmarks,
  lanzar y seguir un job con CLI/boto3.
- EMR: cuándo se elige frente a Glue (control, costo en cargas grandes y continuas, librerías).
- Unir fuentes: transacciones (S3), historial (RDS), perfil (DynamoDB) en un dataset de
  entrenamiento.
- Lake Formation: permisos a nivel de tabla/columna sobre el catálogo; cuándo aparece en una
  pregunta de ML.
- Redshift y OpenSearch: solo su papel si aparecen como fuente o destino.

Corte: notas 1–4.
Caso: construir el dataset de entrenamiento de fraude uniendo las tres fuentes.
Peso: mucho a Catálogo + Athena + Glue jobs; poco a EMR y Lake Formation.
```

```text
Nota 6/29 — Preparación y calidad de datos: DataBrew, Glue Data Quality y Data Wrangler

Tareas de la guía que cubre:
- 1.2 K: Tools to explore, visualize, or transform data and features (Data Wrangler, Glue
  DataBrew)
- 1.2 S: Transforming data by using AWS tools (DataBrew, Data Wrangler)
- 1.1 S: Ingesting data into SageMaker Data Wrangler (y su salida hacia Feature Store, como
  puntero a la nota 7)
- 1.3 S: Validating data quality (DataBrew and AWS Glue Data Quality)
- 1.2 K: Data cleaning, feature engineering and encoding techniques (solo cómo se implementan en
  estas herramientas, no la teoría)

Pregunta central: qué herramienta usar para explorar, limpiar, transformar y validar datos antes
de entrenar, y cómo se automatiza lo que se hizo de forma visual.

Debe quedar claro:
- DataBrew: datasets, recipes, profile jobs, recipe jobs; ejecutarlos por CLI/boto3.
- Glue Data Quality: reglas (DQDL), dónde se evalúan (catálogo o dentro de un job), qué hacer
  cuando fallan.
- Data Wrangler: flows, transformaciones, reportes de calidad/insights, exportación a Processing
  job, pipeline o Feature Store; estado actual del producto (dentro de Canvas).
- Tabla de decisión DataBrew vs Data Wrangler vs Glue job vs Processing job con código propio.
- Cómo se implementan imputación, outliers, encoding y escalado en cada herramienta
  (sin explicar las técnicas).

Corte: notas 1–5.
Caso: limpiar y validar el dataset de fraude antes de entrenar; bloquear si falla una regla.
Peso: mucho a la decisión entre herramientas y a Data Quality.
```

```text
Nota 7/29 — SageMaker Feature Store

Tareas de la guía que cubre:
- 1.1 S: Ingesting data into SageMaker Feature Store
- 1.2 S: Creating and managing features by using AWS tools (SageMaker Feature Store)

Pregunta central: cómo guardar features una vez y servirlas igual para entrenar (histórico) y
para inferir (baja latencia), sin fuga de información temporal.

Debe quedar claro:
- Feature group: definiciones, record identifier, event time; online store vs offline store y
  cuándo habilitar cada uno.
- Ingesta: PutRecord con boto3, ingesta por lotes desde un DataFrame con el SDK, desde Data
  Wrangler o Spark; costos y límites relevantes.
- Lectura en línea: GetRecord / BatchGetRecord y latencia.
- Offline store: formato en S3, tabla en el Glue Data Catalog, consultas con Athena,
  point-in-time correct join para armar datasets de entrenamiento.
- Borrado, TTL y versiones de registros.
- Cuándo NO usar Feature Store (features que no se reutilizan, sin inferencia en línea).

Corte: notas 1–6.
Caso: features de comportamiento del cliente para fraude, usadas en entrenamiento y en inferencia.
Peso: mucho a online vs offline y a point-in-time.
```

```text
Nota 8/29 — Etiquetado de datos: Ground Truth, Mechanical Turk y Augmented AI

Tareas de la guía que cubre:
- 1.2 K: Data annotation and labeling services that create high-quality labeled datasets
- 1.2 S: Validating and labeling data by using AWS services (SageMaker Ground Truth, Amazon
  Mechanical Turk)

Pregunta central: cómo obtener etiquetas confiables a costo razonable y cómo meter revisión
humana en el proceso.

Debe quedar claro:
- Ground Truth: labeling job, tipos de tarea integrados, manifiesto de entrada/salida,
  consolidación de anotaciones, etiquetado automático (active learning) y cuándo compensa.
- Workforces: pública (Mechanical Turk), privada, vendor; implicaciones con datos con PII.
- Ground Truth Plus: cuándo se elige.
- Amazon A2I: human review workflows sobre predicciones (Textract/Rekognition nativos o modelo
  propio); en qué difiere de Ground Truth.

Corte: notas 1–7. Textract y Rekognition solo se mencionan como "servicios que producen
predicciones"; su detalle es la nota 10.
Caso: etiquetar documentos KYC con PII.
Extensión: ~10 páginas incluidas las preguntas.
```

```text
Nota 9/29 — Protección de datos y cumplimiento en cargas de ML

Tareas de la guía que cubre:
- 1.3 K: Techniques to encrypt data
- 1.3 K: Data classification, anonymization, and masking
- 1.3 K: Implications of compliance requirements (PII, PHI, data residency)
- 4.3 K: SageMaker security and compliance features (parte de cifrado)

Pregunta central: cómo se cifran, clasifican y enmascaran los datos de ML en AWS, y qué
configuración concreta lo garantiza en S3 y en los jobs de SageMaker.

Debe quedar claro:
- KMS: llaves administradas por AWS vs por el cliente, key policy vs política IAM, cifrado en
  sobre (envelope) solo al nivel necesario, por qué un AccessDenied puede venir de KMS.
- S3: SSE-S3 vs SSE-KMS vs cifrado del lado del cliente, cifrado por defecto del bucket,
  bucket keys, forzar cifrado con bucket policy.
- SageMaker: llave para el volumen, llave para la salida, cifrado del tráfico entre contenedores
  en entrenamiento distribuido; cifrado en tránsito.
- Macie para descubrir PII en S3; Comprehend (y Comprehend Medical para PHI) para detectar y
  redactar PII en texto; enmascaramiento en DataBrew/Glue.
- Residencia de datos: región, replicación, qué servicios pueden sacar datos de la región.
- Secrets Manager para credenciales de fuentes (p. ej. RDS) usadas por jobs.

Corte: notas 1–8. Aislamiento de red como puntero a la nota 23.
Caso: restricción regulatoria de Kanan (llaves propias, datos en la región).
Peso: mucho a KMS en S3/SageMaker y a diagnóstico; poco a teoría de criptografía.
```

### Bloque C — Modelado (dominio 2)

```text
Nota 10/29 — Servicios de IA gestionados: cuándo no entrenar un modelo propio

Tareas de la guía que cubre:
- 2.1 K: How to use AWS AI services (Translate, Transcribe, Rekognition, Bedrock) to solve
  specific business problems
- 2.1 S: Selecting AI services to solve common business needs
- 2.1 S: Assessing available data and problem complexity to determine the feasibility of an ML
  solution

Pregunta central: dado un problema de negocio, cuándo basta un servicio de IA gestionado, cuál
elegir y cuándo hace falta construir un modelo en SageMaker.

Debe quedar claro (una llamada boto3 realista para los principales):
- Visión y documentos: Rekognition (incl. Custom Labels), Textract (y cuándo AnalyzeDocument vs
  AnalyzeID vs asíncrono), Lookout for Vision.
- Lenguaje y voz: Comprehend (incl. clasificación personalizada), Comprehend Medical, Translate,
  Transcribe, Polly, Lex, Kendra.
- Negocio: Personalize, Fraud Detector, Lookout for Metrics, Lookout for Equipment.
- Otros del temario: HealthLake, CodeGuru, DevOps Guru, Amazon Q (una línea cada uno).
- Estado actual: servicios cerrados a clientes nuevos o descontinuados (verificar Lookout, Fraud
  Detector y otros).
- Criterios de decisión: datos disponibles, necesidad de personalización, costo, latencia,
  interpretabilidad.
- Bedrock solo como "servicio de modelos fundacionales"; su detalle es la nota 12.

Corte: notas 1–9.
Caso: pipeline KYC con Textract + Comprehend; comparar Fraud Detector vs modelo propio.
Peso: mucho a la tabla de decisión y a distractores típicos (servicio parecido pero equivocado).
```

```text
Nota 11/29 — Algoritmos integrados de SageMaker

Tareas de la guía que cubre:
- 2.1 K: SageMaker built-in algorithms and when to apply them
- 2.1 S: Choosing built-in algorithms
- 2.1 S: Comparing and selecting appropriate ML models or algorithms to solve specific problems
- 2.1 K: How to consider interpretability during model selection
- 2.1 S: Selecting models or algorithms based on costs
- 2.2 S: Using SageMaker built-in algorithms to develop ML models

Pregunta central: qué algoritmo integrado corresponde a cada tipo de problema y qué exige cada
uno en formato de datos, content type, instancias y modo de entrada.

Debe quedar claro (sin explicar la teoría de los algoritmos):
- Mapa problema -> algoritmo: XGBoost, Linear Learner, KNN, Factorization Machines, DeepAR,
  Random Cut Forest, IP Insights, K-Means, PCA, LDA, NTM, BlazingText, Seq2Seq, Object2Vec,
  Image Classification, Object Detection, Semantic Segmentation, y los tabulares más nuevos
  (LightGBM, CatBoost, TabTransformer, AutoGluon-Tabular) con su estado actual.
- Para cada uno: formatos aceptados, CPU/GPU, entrenamiento distribuido sí/no, supervisado o no.
- Detalles que el examen usa como trampa: target en la primera columna sin encabezado para CSV,
  recordIO-protobuf obligatorio en algunos, canales requeridos.
- Hiperparámetros solo cuando su nombre en SageMaker no sea obvio.
- Ejemplo completo: XGBoost integrado para fraude con boto3/SDK y DeepAR para cajeros.

Corte: notas 1–10.
Caso: fraude (XGBoost/Linear Learner), cajeros (DeepAR), anomalías (RCF).
Peso: mucho a la tabla y a los requisitos de datos.
```

```text
Nota 12/29 — Modelos fundacionales: Amazon Bedrock y SageMaker JumpStart

Tareas de la guía que cubre:
- 2.1 S: Choosing built-in algorithms, foundation models, and solution templates (JumpStart,
  Bedrock)
- 2.2 S: Using custom datasets to fine-tune pre-trained models (Bedrock, JumpStart)
- 2.2 S: Preventing catastrophic forgetting (en el contexto de fine-tuning)

Pregunta central: cuándo usar un modelo fundacional en Bedrock, cuándo desplegarlo o ajustarlo
en SageMaker con JumpStart, y cómo se hace el fine-tuning en cada caso.

Debe quedar claro:
- Bedrock: acceso a modelos, invocación con boto3 (bedrock-runtime), inferencia on-demand vs
  batch vs Provisioned Throughput (y cuándo es obligatorio), personalización (fine-tuning,
  continued pre-training) con datos en S3, costo.
- JumpStart: modelos preentrenados y solution templates, desplegar y hacer fine-tuning con el
  SDK; qué infraestructura queda bajo tu responsabilidad.
- Bedrock vs JumpStart: control, costo, gestión, datos y cumplimiento.
- Catastrophic forgetting en fine-tuning y cómo mitigarlo en estas herramientas.
- Knowledge Bases y Guardrails: solo qué son, en un párrafo, si ayudan a responder preguntas del
  examen; nada de agentes.

Corte: notas 1–11. Endpoints de SageMaker como puntero a la nota 19.
Caso: clasificar el tipo de documento KYC con un FM ajustado.
Peso: mucho a la decisión Bedrock vs JumpStart y al proceso de fine-tuning.
```

```text
Nota 13/29 — Script mode y modelos entrenados fuera de SageMaker

Tareas de la guía que cubre:
- 2.2 S: Using SageMaker script mode with SageMaker supported frameworks (TensorFlow, PyTorch)
- 2.2 K: Methods to integrate models that were built outside SageMaker into SageMaker
- 2.2 S: Using common ML libraries to develop ML models

Pregunta central: cómo llevar tu propio código de entrenamiento (scikit-learn, PyTorch,
TensorFlow, XGBoost de código abierto) a un training job, y cómo traer un modelo ya entrenado
en otro lugar.

Debe quedar claro:
- Contenedores de framework de AWS y script mode: entry_point, source_dir, requirements.txt,
  hiperparámetros como argumentos, variables de entorno SM_* (canales, modelo, salida).
- Qué debe guardar el script y dónde para que SageMaker produzca model.tar.gz.
- Métricas en logs y metric_definitions (regex) para verlas en CloudWatch.
- Probar localmente (local mode) antes de gastar instancias.
- Modelo externo: empaquetar model.tar.gz, subirlo a S3 y registrarlo como Model de SageMaker;
  qué pasa si la versión del framework no coincide.
- Script mode vs extender un contenedor vs BYOC (BYOC como puntero a la nota 20).
- Interfaces actuales del SDK (Estimator de framework vs ModelTrainer, si aplica) con aviso.

Corte: notas 1–12.
Caso: modelo de fraude en PyTorch con script mode; modelo scikit-learn entrenado en la laptop
de un data scientist que hay que llevar a SageMaker.
Peso: mucho a la estructura del script y a los errores típicos.
```

```text
Nota 14/29 — Entrenar más rápido y más barato en SageMaker

Tareas de la guía que cubre:
- 2.2 K: Methods to reduce model training time (early stopping, distributed training)
- 3.1 S: Choosing the appropriate compute environment for training based on requirements (GPU or
  CPU, processor family, networking bandwidth)
- 3.2 S: dynamically adding Spot Instances
- 3.2 K: Difference between on-demand and provisioned resources (en entrenamiento)
- 4.2 S: Optimizing infrastructure costs by selecting purchasing options (solo entrenamiento;
  el resto en la nota 29)

Pregunta central: con un training job que ya funciona, qué palancas reducen tiempo y costo,
cómo se configuran y qué rompe cada una.

Debe quedar claro:
- Familias de instancias para entrenar (CPU, GPU, Trainium) y ancho de banda de red; cuándo una
  instancia grande vs varias.
- Entrenamiento distribuido: data parallel vs model parallel, librerías de SageMaker y soporte
  nativo de los frameworks, qué cambia en el script; tráfico entre nodos.
- Managed Spot Training: MaxWaitTimeInSeconds vs MaxRuntimeInSeconds, checkpoints en S3,
  qué pasa sin checkpoint, cómo leer el ahorro en la descripción del job.
- Warm pools (KeepAlivePeriodInSeconds): cuándo ahorran tiempo.
- Early stopping dentro del script vs el de AMT (puntero a la nota 15).
- Heterogeneous clusters: solo si ayuda a una pregunta.

Corte: notas 1–13.
Caso: modelo de fraude en PyTorch con dataset de ~40 GB en Parquet.
Peso: mucho a Spot + checkpoints y a distribuido; poco a warm pools.
```

```text
Nota 15/29 — SageMaker Automatic Model Tuning (AMT)

Tareas de la guía que cubre:
- 2.2 K: Hyperparameter tuning techniques (random search, Bayesian optimization)
- 2.2 K: Model hyperparameters and their effects on model performance
- 2.2 S: Performing hyperparameter tuning (SageMaker AMT)
- 2.2 S: Integrating automated hyperparameter optimization capabilities

Pregunta central: cómo se configura, lanza, vigila y analiza un tuning job, y qué decisiones de
configuración cambian costo y calidad del resultado.

Debe quedar claro (sin teoría de búsqueda de hiperparámetros):
- Estructura de un tuning job: métrica objetivo, rangos (continuo, entero, categórico), tipo de
  escala, límites de jobs totales y en paralelo, relación con la definición del training job.
- Estrategias disponibles en AMT (Bayesian, Random, Grid, Hyperband) y cuándo elegir cada una.
- Paralelismo vs eficiencia de la búsqueda bayesiana.
- Early stopping de AMT; warm start (tipos) y reutilización de tuning jobs anteriores.
- Métrica objetivo que no aparece: metric_definitions y regex (conexión con la nota 13).
- Leer resultados: mejor training job, análisis de los jobs con boto3/SDK.
- AMT con Spot (puntero a lo visto en la nota 14).
- Autopilot: solo qué es y cuándo aparece como alternativa.

Corte: notas 1–14.
Caso: tuning de XGBoost para fraude optimizando AUC en validación.
Peso: mucho a configuración y a errores (métrica mal definida, rangos mal escalados).
```

```text
Nota 16/29 — SageMaker Debugger y análisis de convergencia

Tareas de la guía que cubre:
- 2.3 K: Convergence issues
- 2.3 S: Using SageMaker Model Debugger to debug model convergence
- 2.3 K: Methods to identify model overfitting and underfitting (vía Debugger)

Pregunta central: cómo detectar durante el entrenamiento que un modelo no converge o que la
infraestructura está mal aprovechada, y cómo actuar automáticamente.

Debe quedar claro:
- Qué captura Debugger (tensores, métricas) y dónde lo guarda.
- Reglas integradas (pérdida que no baja, gradientes que desaparecen/explotan, sobreajuste...),
  cómo se adjuntan a un training job y cómo se leen sus estados.
- Acciones de regla (detener el entrenamiento, notificar).
- Profiling de uso de sistema: estado actual de Debugger Profiler vs SageMaker Profiler.
- Diagnóstico con CloudWatch como alternativa ligera.

Corte: notas 1–15.
Caso: el modelo PyTorch de fraude cuya pérdida se estanca.
Extensión: ~10 páginas incluidas las preguntas.
```

```text
Nota 17/29 — SageMaker Clarify: sesgo en datos y modelos, y explicabilidad

Tareas de la guía que cubre:
- 1.3 K: Pre-training bias metrics for numeric, text, and image data (CI, DPL)
- 1.3 K: Strategies to address class imbalance (synthetic data generation, resampling)
- 1.3 S: Identifying and mitigating sources of bias in data (selection bias, measurement bias)
  by using SageMaker Clarify
- 1.3 S: Preparing data to reduce prediction bias (dataset splitting, shuffling, augmentation)
- 2.3 K: Metrics available in SageMaker Clarify to gain insights into ML training data and models
- 2.3 S: Selecting and interpreting evaluation metrics and detecting model bias
- 2.3 S: Using SageMaker Clarify to interpret model outputs

Pregunta central: cómo se configura y ejecuta Clarify para medir sesgo antes y después de
entrenar y para explicar predicciones, y cómo se interpretan sus salidas.

Debe quedar claro (sin definir las métricas estadísticamente más allá de lo necesario):
- Clarify como processing job: DataConfig, BiasConfig (faceta, grupo, etiqueta positiva),
  ModelConfig, SHAPConfig; qué necesita cada análisis.
- Métricas pre-entrenamiento (CI, DPL, KL, JS, etc.) y post-entrenamiento (DPPL, DI, etc.):
  cuál responde a qué pregunta de negocio.
- Explicabilidad: SHAP (baseline, número de muestras, costo), PDP; texto e imágenes.
- Dónde quedan los reportes y cómo leerlos (S3, Studio).
- Mitigación del desbalance y del sesgo en AWS (remuestreo, datos sintéticos, aumento) y qué
  herramienta lo hace.
- Uso de Clarify dentro de Data Wrangler y en monitoreo (punteros a notas 6 y 27).

Corte: notas 1–16.
Caso: modelo de fraude y posible sesgo por región o edad del cliente.
Peso: mucho a configuración y a elegir la métrica correcta para el escenario.
```

```text
Nota 18/29 — Experimentos reproducibles, versionado y SageMaker Model Registry

Tareas de la guía que cubre:
- 2.2 S: Managing model versions for repeatability and audits (SageMaker Model Registry)
- 2.3 S: Performing reproducible experiments by using AWS services
- 2.3 K: Methods to create performance baselines
- 2.3 S: Assessing tradeoffs between model performance, training time, and cost
- 2.2 S: Combining multiple training models to improve performance (ensembling, stacking,
  boosting): solo cómo se organiza en SageMaker
- 2.2 K: Factors that influence model size; 2.2 S: Reducing model size (puntero a nota 21)

Pregunta central: cómo dejar rastro de cada experimento y de cada modelo candidato para poder
compararlos, reproducirlos, aprobarlos y auditarlos.

Debe quedar claro:
- SageMaker Experiments (runs, parámetros, métricas) y su estado actual frente a MLflow
  gestionado en SageMaker; qué evalúa MLA-C01.
- Model Registry: model package groups, model packages (versiones), estados de aprobación,
  métricas y metadatos adjuntos, registrar con boto3/SDK, aprobar/rechazar.
- Linaje (ML Lineage Tracking) y Model Cards: para auditoría.
- Baselines de desempeño: modelo simple de referencia, cómo guardarlos junto al candidato.
- Reproducibilidad real: semillas, versiones de imagen, datos versionados en S3 (versioning).

Corte: notas 1–17.
Caso: tres candidatos de fraude (XGBoost, Linear Learner, PyTorch); uno se aprueba.
Peso: mucho a Model Registry; menos a Experiments.
```

### Bloque D — Despliegue y orquestación (dominio 3)

```text
Nota 19/29 — Opciones de inferencia en SageMaker: real-time, serverless, asíncrona y batch

Tareas de la guía que cubre:
- 3.1 K: AWS deployment services (SageMaker)
- 3.1 K: Methods to serve ML models in real time and in batches
- 3.1 K: Model and endpoint requirements (serverless, real-time, asynchronous endpoints,
  batch inference)
- 3.1 S: Choosing model deployment strategies (real time, batch)
- 3.1 S: Evaluating performance, cost, and latency tradeoffs
- 3.1 K: How to provision compute resources in production and test environments (CPU, GPU)

Pregunta central: dado un requisito de latencia, tamaño de payload, patrón de tráfico y costo,
qué modo de inferencia se elige y cómo se crea con boto3/SDK.

Debe quedar claro:
- Model -> EndpointConfig -> Endpoint: qué es cada recurso, por qué están separados, cómo se
  actualiza un endpoint.
- Invocación: sagemaker-runtime invoke_endpoint, content types, errores típicos (ModelError,
  timeouts).
- Real-time vs serverless (cold start, memoria, concurrencia, sin GPU) vs asíncrona (payloads
  grandes, cola, notificaciones SNS, escalar a cero) vs batch transform (archivos en S3,
  split/join, filtros de entrada/salida).
- Límites verificados de payload y tiempo de cada modo.
- Entornos de prueba vs producción; local mode.
- Tabla de decisión con costo.

Corte: notas 1–18.
Caso: fraude (real-time), KYC (asíncrona con PDFs grandes), cajeros (batch nocturno).
Peso: mucho a la decisión y a los límites.
```

```text
Nota 20/29 — Contenedores de inferencia y hosting de varios modelos

Tareas de la guía que cubre:
- 3.1 K: How to choose appropriate containers (provided or customized)
- 3.1 S: Selecting multi-model or multi-container deployments
- 3.1 S: Selecting the correct deployment target (SageMaker endpoints, Kubernetes, ECS, EKS,
  Lambda)
- 3.2 K: Containerization concepts and AWS container services
- 3.2 S: Building and maintaining containers (ECR, EKS, ECS, BYOC with SageMaker)

Pregunta central: qué contenedor sirve el modelo, cómo se construye cuando no basta el de AWS y
cómo alojar varios modelos o contenedores detrás de un mismo endpoint.

Debe quedar claro:
- Contenedor provisto, extendido (FROM imagen de AWS) y BYOC; contrato de un contenedor de
  inferencia (/ping, /invocations, puerto, serve) y de entrenamiento (train, /opt/ml).
- inference.py en contenedores de framework (model_fn, input_fn, predict_fn, output_fn).
- ECR: repositorio, build, push, autenticación, escaneo, política de ciclo de vida; permisos
  para que SageMaker extraiga la imagen.
- Multi-model endpoints (TargetModel, carga dinámica desde S3, mismo contenedor),
  multi-container endpoints (invocación directa o serial), inference pipelines: diferencias.
- Destinos alternativos: ECS, EKS, Lambda; cuándo tiene sentido salir de SageMaker.

Corte: notas 1–19.
Caso: un modelo de fraude por país en un multi-model endpoint; preprocesamiento + modelo en
inference pipeline.
Peso: mucho a MME vs MCE vs pipeline y al contrato BYOC.
```

```text
Nota 21/29 — Optimización de modelos para inferencia y dispositivos edge

Tareas de la guía que cubre:
- 3.1 K: Methods to optimize models on edge devices (SageMaker Neo)
- 2.2 S: Reducing model size (altering data types, pruning, updating feature selection,
  compression)
- 2.2 K: Factors that influence model size
- 4.2 K: Differences between instance types and how they affect performance (memory optimized,
  compute optimized, general purpose, inference optimized)
- 4.2 S: Rightsizing instance families and sizes (SageMaker Inference Recommender)

Pregunta central: cómo hacer el modelo más pequeño y rápido y cómo elegir la instancia de
inferencia correcta con evidencia, no a ojo.

Debe quedar claro:
- Neo: compilación para un destino (instancia o dispositivo), frameworks soportados, cómo se
  lanza un compilation job, qué se despliega después.
- Estado actual del lado edge (Edge Manager descontinuado; verificar).
- Cuantización, poda, destilación como técnicas y dónde se aplican en AWS.
- Familias de instancias para inferencia: CPU, GPU, Inferentia, Graviton.
- Inference Recommender: jobs por defecto vs avanzados (prueba de carga), lectura de resultados.

Corte: notas 1–20.
Caso: reducir latencia y costo del endpoint de fraude.
Extensión: ~12 páginas incluidas las preguntas.
```

```text
Nota 22/29 — Escalado automático y estrategias de despliegue de endpoints

Tareas de la guía que cubre:
- 3.1 K: Deployment best practices (versioning, rollback strategies)
- 3.2 K: How to compare scaling policies
- 3.2 K: How to use SageMaker endpoint auto scaling policies (based on demand, time)
- 3.2 S: Choosing specific metrics for auto scaling (model latency, CPU utilization,
  invocations per instance)
- 3.2 S: Applying best practices (automatic scaling on SageMaker endpoints)
- 3.3 K: Deployment strategies and rollback actions (blue/green, canary, linear)
- 2.3 S: Comparing the performance of a shadow variant to a production variant
- 4.1 S: Monitoring model performance in production by using A/B testing

Pregunta central: cómo hacer que un endpoint aguante la demanda sin pagar de más y cómo
cambiar el modelo en producción sin romperla.

Debe quedar claro:
- Application Auto Scaling para endpoints: registrar el target, target tracking vs step
  scaling vs scheduled, métricas predefinidas y personalizadas, cooldowns, mínimo cero en
  asíncronos; ejemplo completo con CLI.
- Production variants y pesos de tráfico (A/B), invocar una variante concreta.
- Deployment guardrails: blue/green con all-at-once, canary y linear; baking period;
  rollback automático con alarmas de CloudWatch; rolling deployments.
- Shadow tests / variantes shadow: qué se compara y cómo.
- Diagnóstico: escala tarde, escala de más, el despliegue se revierte.

Corte: notas 1–21. Alarmas de CloudWatch explicadas lo mínimo; detalle en la nota 28.
Caso: endpoint de fraude con picos en quincena y un modelo nuevo por liberar.
Peso: mucho a métricas de escalado y a guardrails.
```

```text
Nota 23/29 — Aislamiento de red para cargas de ML

Tareas de la guía que cubre:
- 4.3 K: Controls for network access to ML resources
- 4.3 S: Building VPCs, subnets, and security groups to securely isolate ML systems
- 3.2 S: Configuring SageMaker endpoints within the VPC network

Pregunta central: cómo se mete SageMaker (jobs, endpoints, Studio) en una red privada sin romper
su acceso a S3, ECR y demás servicios.

Debe quedar claro (lo justo de redes, desde cero):
- VPC, subredes públicas/privadas, tablas de ruteo, NAT, security groups: solo lo necesario.
- VpcConfig en jobs y modelos: qué crea SageMaker (ENIs), qué debe permitir el security group.
- VPC endpoints: gateway (S3) vs interface (API de SageMaker, runtime, ECR, CloudWatch, STS...);
  qué falla cuando falta cada uno y cómo se ve el error.
- Network isolation (EnableNetworkIsolation) vs VPC; tráfico entre contenedores cifrado.
- Studio en modo solo VPC; políticas de endpoint y bucket policies con aws:SourceVpce.
- Condiciones IAM para obligar a usar VPC en los jobs.
- PrivateLink/Direct Connect: solo cuándo aparecen.

Corte: notas 1–22.
Caso: cuenta de prod de Kanan sin acceso a internet.
Peso: mucho a VPC endpoints y a diagnóstico de jobs que se cuelgan o fallan por red.
```

```text
Nota 24/29 — Infraestructura como código para ML: CloudFormation y CDK

Tareas de la guía que cubre:
- 3.2 K: Tradeoffs and use cases of IaC options (CloudFormation, CDK)
- 3.2 S: Automating the provisioning of compute resources, including communication between
  stacks (CloudFormation, CDK)

Pregunta central: cómo describir en código los recursos de ML (roles, buckets, modelo, endpoint)
para crearlos, actualizarlos y replicarlos entre dev y prod.

Debe quedar claro:
- CloudFormation: plantilla, stack, parámetros, outputs, change sets, rollback; recursos
  AWS::SageMaker::* y qué pasa al actualizar una EndpointConfig.
- Comunicación entre stacks: exports/ImportValue, nested stacks, SSM Parameter Store.
- CDK: constructs, synth, deploy, bootstrap; referencias entre stacks.
- CloudFormation vs CDK vs SDK directo: cuándo cada uno.
- Service Catalog: solo cómo aparece junto a SageMaker Projects (puntero a la nota 26).

Corte: notas 1–23.
Caso: stack de red + stack del endpoint de fraude, desplegados en dev y prod.
Peso: mucho a comunicación entre stacks y a actualización de endpoints.
```

```text
Nota 25/29 — Orquestación de flujos de ML: SageMaker Pipelines, EventBridge, Step Functions y MWAA

Tareas de la guía que cubre:
- 3.1 S: Selecting the correct deployment orchestrator (Apache Airflow, SageMaker Pipelines)
- 3.3 S: Using AWS services to automate orchestration (deploy ML models, automate model
  building)
- 3.3 S: Configuring training and inference jobs (EventBridge rules, SageMaker Pipelines)
- 3.3 S: Building and integrating mechanisms to retrain models
- 3.3 K: Automation and integration of data ingestion with orchestration services

Pregunta central: cómo encadenar procesamiento, entrenamiento, evaluación, registro y despliegue
en un flujo automático que se dispara solo, y qué orquestador conviene.

Debe quedar claro:
- SageMaker Pipelines: definición, parámetros, pasos (processing, training, tuning, condition,
  register model, transform, lambda, callback, fail, quality/clarify check), propiedades entre
  pasos, caching, ejecución e inspección con boto3.
- EventBridge: reglas por evento (p. ej. modelo aprobado en Model Registry, archivo nuevo en S3)
  y por horario; targets.
- Step Functions: integraciones con SageMaker; cuándo preferirlo.
- MWAA (Airflow): cuándo preferirlo.
- SNS/SQS en estos flujos; AWS Batch: solo cuándo aparece.
- Reentrenamiento: disparadores (horario, datos nuevos, drift como puntero a la nota 27).

Corte: notas 1–24.
Caso: pipeline de fraude de datos a modelo registrado, disparado cada semana y al llegar datos.
Peso: mucho a Pipelines y a la decisión entre orquestadores.
```

```text
Nota 26/29 — CI/CD para ML con CodePipeline, CodeBuild y CodeDeploy

Tareas de la guía que cubre:
- 3.3 K: Capabilities and quotas for CodePipeline, CodeBuild, and CodeDeploy
- 3.3 K: Version control systems and basic usage (Git)
- 3.3 K: CI/CD principles and how they fit into ML workflows
- 3.3 K: How code repositories and pipelines work together
- 3.3 S: Configuring and troubleshooting CodeBuild, CodeDeploy, and CodePipeline, including stages
- 3.3 S: Applying continuous deployment flow structures (Gitflow, GitHub Flow)
- 3.3 S: Creating automated tests in CI/CD pipelines (integration, unit, end-to-end tests)
- 4.3 K: Security best practices for CI/CD pipelines

Pregunta central: cómo se conecta un repositorio Git con la construcción, prueba y despliegue
automático del código y los modelos de ML, en dos cuentas.

Debe quedar claro:
- Git lo mínimo (ramas, merge, pull request) y Gitflow vs GitHub Flow.
- CodePipeline: stages, actions, artefactos, fuentes (GitHub vía conexión; estado de CodeCommit),
  aprobación manual, despliegue entre cuentas.
- CodeBuild: buildspec.yml, fases, rol, variables y secretos, logs, errores típicos.
- CodeDeploy: qué despliega (EC2, ECS, Lambda) y por qué los endpoints de SageMaker normalmente
  se despliegan con CloudFormation o SDK desde CodeBuild (trampa frecuente).
- SageMaker Projects y plantillas MLOps: build pipeline + deploy pipeline.
- Tests en cada fase; seguridad del pipeline (roles mínimos, secretos, artefactos cifrados).
- CodeArtifact: para dependencias.

Corte: notas 1–25.
Caso: repositorio del modelo de fraude con despliegue a dev automático y a prod con aprobación.
Peso: mucho a la estructura del pipeline y al diagnóstico de stages que fallan.
```

### Bloque E — Operación (dominio 4)

```text
Nota 27/29 — Monitoreo de modelos en producción con SageMaker Model Monitor

Tareas de la guía que cubre:
- 4.1 K: Drift in ML models
- 4.1 K: Techniques to monitor data quality and model performance
- 4.1 K: Design principles for ML lenses relevant to monitoring
- 4.1 S: Monitoring models in production (SageMaker Model Monitor)
- 4.1 S: Detecting changes in the distribution of data (SageMaker Clarify)
- 4.1 S: Monitoring workflows to detect anomalies or errors in data processing or model inference

Pregunta central: cómo se detecta que el modelo en producción se degradó y cómo esa detección
se convierte en alerta o reentrenamiento.

Debe quedar claro (sin teoría de drift):
- Data capture en endpoints y batch transform: configuración, porcentaje de muestreo, formato
  en S3.
- Los cuatro monitores: calidad de datos, calidad del modelo (con ground truth y su unión por
  inference ID), bias drift y feature attribution drift (Clarify por debajo).
- Baselines (statistics y constraints), monitoring schedules, reportes de violaciones,
  métricas en CloudWatch.
- Conectar violaciones con alarmas y con el pipeline de reentrenamiento.
- ML Lens del Well-Architected: principios relevantes en un párrafo.
- A2I para revisar predicciones dudosas como fuente de ground truth (conexión con la nota 8).

Corte: notas 1–26.
Caso: el patrón de fraude cambia en diciembre; las etiquetas llegan con 30 días de retraso.
Peso: mucho a configuración y a elegir el monitor correcto.
```

```text
Nota 28/29 — Observabilidad y auditoría de sistemas de ML

Tareas de la guía que cubre:
- 4.2 K: Key performance metrics for ML infrastructure (utilization, throughput, availability,
  scalability, fault tolerance)
- 4.2 K: Monitoring and observability tools (X-Ray, CloudWatch Lambda Insights, CloudWatch Logs
  Insights)
- 4.2 K: How to use CloudTrail to log, monitor, and invoke re-training activities
- 4.2 S: Configuring and using tools to troubleshoot and analyze resources (CloudWatch Logs,
  CloudWatch alarms)
- 4.2 S: Creating CloudTrail trails
- 4.2 S: Setting up dashboards (QuickSight, CloudWatch dashboards)
- 4.2 S: Monitoring infrastructure (EventBridge events)
- 4.2 S: Monitoring and resolving latency and scaling issues
- 4.3 S: Monitoring, auditing, and logging ML systems to ensure continued security and compliance

Pregunta central: qué señales emite un sistema de ML en AWS, dónde se leen y cómo se usan para
diagnosticar latencia, errores y accesos indebidos.

Debe quedar claro:
- Métricas de endpoints (ModelLatency vs OverheadLatency, Invocations, 4XX/5XX, uso de CPU/GPU
  /memoria) y dónde viven; métricas personalizadas.
- Alarmas: umbral, periodos, acciones; alarmas compuestas.
- Logs Insights: consultas para errores de inferencia y de jobs.
- X-Ray y Lambda Insights: cuándo aplican (Lambda/API Gateway delante del endpoint).
- CloudTrail: eventos de administración vs de datos, trails, qué llamadas de SageMaker quedan
  registradas (verificar), usarlo para auditoría y para disparar acciones con EventBridge.
- AWS Config: reglas de cumplimiento sobre recursos de ML.
- Dashboards en CloudWatch vs QuickSight: para quién es cada uno.

Corte: notas 1–27.
Caso: la latencia del endpoint de fraude sube en hora pico; auditoría pide quién aprobó un modelo.
Peso: mucho a métricas de latencia y diagnóstico, y a CloudTrail.
```

```text
Nota 29/29 — Costos y capacidad de cargas de ML

Tareas de la guía que cubre:
- 4.2 K: Capabilities of cost analysis tools (Cost Explorer, Billing and Cost Management,
  Trusted Advisor)
- 4.2 K: Cost tracking and allocation techniques (resource tagging)
- 4.2 S: Preparing infrastructure for cost monitoring (tagging strategy)
- 4.2 S: Troubleshooting capacity concerns that involve cost and performance (provisioned
  concurrency, service quotas, auto scaling)
- 4.2 S: Optimizing costs and setting cost quotas (Cost Explorer, Trusted Advisor, Budgets)
- 4.2 S: Optimizing infrastructure costs by selecting purchasing options (Spot, On-Demand,
  Reserved Instances, SageMaker Savings Plans)
- 4.2 S: Rightsizing instance families and sizes (AWS Compute Optimizer)
- 3.2 S: Applying best practices for cost-effective ML solutions (Spot, Lambda behind endpoints)

Pregunta central: cómo saber cuánto cuesta cada parte del sistema de ML, cómo ponerle límites
y cómo pagar menos por la misma capacidad.

Debe quedar claro:
- Etiquetado: estrategia, cost allocation tags (activarlos), etiquetas en recursos de SageMaker.
- Cost Explorer, Budgets (alertas y acciones), Billing, Trusted Advisor, Compute Optimizer:
  qué pregunta responde cada uno.
- Opciones de compra: On-Demand, Spot, Reserved Instances (aplican a EC2, no a SageMaker),
  SageMaker Savings Plans (qué cubren); trampa frecuente.
- Service quotas: ver y pedir aumentos; errores de cuota en jobs y endpoints.
- Lambda: concurrencia reservada vs provisioned concurrency.
- Resumen de palancas de costo por fase (datos, entrenamiento, inferencia) con punteros a las
  notas donde se explicaron.

Corte: notas 1–28.
Caso: la factura de ML de Kanan se duplicó; hay que encontrar el responsable y poner límites.
Peso: mucho a opciones de compra, etiquetado y la decisión entre herramientas.
```
