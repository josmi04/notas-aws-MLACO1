# Repaso final · AWS Certified Machine Learning Engineer – Associate (MLA-C01)

Síntesis de las notas de la guía oficial (capítulos 2–8, dominios 1–4). Supone que los temas ya se estudiaron. Conserva servicios, límites, parámetros y las distinciones que suelen aparecer como distractores; omite la teoría de ML, el código y los ejemplos.

**Marcas:** ⚠️ = dato que corrige un error del libro (corrección ya verificada en las notas originales) · 🕒 = cambio de estado del servicio posterior al libro.

> **Vigencia (septiembre de 2026)**
> - Amazon SageMaker se llama **Amazon SageMaker AI** desde diciembre de 2024. Aquí, «SageMaker» = SageMaker AI.
> - Desde el **30-07-2026** no admiten clientes nuevos (los existentes siguen usándolos): SageMaker **Model Monitor**, **Clarify**, **Ground Truth**, **Augmented AI (A2I)** y **Role Manager**, y **Amazon Mechanical Turk**. El MLA-C01 los trata como vigentes: estúdialos y elígelos como tales cuando el escenario los pida.

**Contenido**

1. Ingesta y almacenamiento de datos
2. Transformación de datos, feature engineering e integridad
3. Servicios de IA de AWS y Amazon Bedrock
4. Algoritmos integrados de SageMaker AI y selección de modelos
5. Entrenamiento, ajuste de hiperparámetros y evaluación
6. Despliegue e infraestructura de inferencia
7. Orquestación, CI/CD e infraestructura como código
8. Monitoreo de modelos en producción
9. Monitoreo de infraestructura y optimización de costos
10. Seguridad de soluciones de ML

---

## 1. Ingesta y almacenamiento de datos

*Dominio 1 · Tarea 1.1*

### 1.1 Criterios de selección

La **ingesta** lleva los datos crudos de las fuentes a AWS (batch o tiempo real); el **almacenamiento** los persiste hasta su procesamiento. Criterios de ingesta: escalabilidad (velocidad y volumen), resiliencia (reanudar desde el punto de fallo), seguridad y cumplimiento (PCI DSS, HIPAA, residencia de datos), costo (un stream no se detiene y su costo crece rápido) y flexibilidad. Criterios de almacenamiento: **durabilidad** (cuánto tiempo conservar los datos), **disponibilidad** (qué tan pronto se necesitan), tipo (objeto, archivo o bloque), costo y protección en reposo. Durabilidad ≠ disponibilidad.

El **patrón de acceso** (cómo se consultan, almacenan y recuperan los datos) determina, junto con el algoritmo, el formato y el servicio; su tamaño y su velocidad (picos de consulta) guían el particionamiento.

### 1.2 Formatos de datos

| Formato | Orientación | Cuándo |
|---|---|---|
| CSV | Filas | Tabular estructurado |
| JSON | Documento | Semiestructurado |
| Parquet / ORC | **Columnar** | Consultas analíticas sobre pocas columnas: compresión por columna y lectura de columnas sin leer filas completas |
| Avro | **Filas** | Serialización compacta con el esquema del escritor incluido en los datos; evolución de esquema (lector ≠ escritor) |
| RecordIO | Registros con su longitud antepuesta | Apache MXNet |

El primer criterio es que el formato lo acepte el algoritmo. Trampa: Avro es por filas, no columnar.

### 1.3 Formatos de entrenamiento de los algoritmos integrados

| Formatos aceptados | Algoritmos |
|---|---|
| RecordIO-protobuf o CSV | K-means, k-NN, Linear Learner, LDA, NTM, PCA, Random Cut Forest |
| Solo RecordIO-protobuf | Factorization Machines, Seq2Seq (tokens enteros) |
| Solo CSV | IP Insights |
| JSON Lines o Parquet | DeepAR |
| CSV, LibSVM o Parquet | XGBoost |
| Texto (una oración por línea, tokens separados por espacios) | BlazingText |
| RecordIO (MXNet) o .jpg/.png | Image Classification, Object Detection |
| Solo archivos de imagen | Semantic Segmentation |

⚠️ El libro (tabla 2.1) atribuye CSV a Factorization Machines, texto plano a Seq2Seq y RecordIO a Semantic Segmentation; ninguno los acepta.

- «RecordIO» (imágenes, MXNet) ≠ «RecordIO-protobuf» (tensores y datos tabulares). DeepAR es el único con JSON Lines; XGBoost, el único con LibSVM.
- CSV en un algoritmo integrado: sin cabecera y declarado como `text/csv` (`text/csv;label_size=0` si no hay columna de etiqueta; en XGBoost la etiqueta va en la primera columna). Si no se declara `text/csv`, integrados como K-means o PCA esperan RecordIO-protobuf.

### 1.4 Tipos de almacenamiento y servicios por tipo de dato

| Tipo | Servicio | Modelo | Ideal para |
|---|---|---|---|
| Objeto | S3 | Dato + metadatos + ID único; acceso por API | Data lakes, cloud-native, gran escala a bajo costo |
| Archivo | EFS, familia FSx | Jerárquico y compartido | Directorios compartidos, medios, CMS |
| Bloque | EBS | Bloques de tamaño fijo, sin metadatos | Transaccional, baja latencia |

Por tipo de dato: estructurado → RDS, Aurora, Redshift, S3, Athena; semiestructurado → DynamoDB, DocumentDB, Athena, S3; no estructurado → S3. Rekognition, Transcribe y Comprehend **procesan** datos no estructurados, pero no son almacenamiento. S3 es el único servicio presente en las tres categorías.

**Fuentes de datos de entrenamiento con integración nativa en SageMaker: S3, EFS y FSx for Lustre. EBS no lo es.**

### 1.5 Streaming: Firehose, Kinesis Data Streams, MSK y Flink

**Amazon Data Firehose** (antes Kinesis Data Firehose) captura, transforma y **entrega** streams a destinos en **near real time (segundos)**. Es totalmente administrado, escala solo y no requiere código consumidor.
- Fuentes: Direct PUT (API), Kinesis Data Streams, MSK y más de 20 integraciones (CloudWatch Logs, logs de WAF y de Network Firewall, SNS, AWS IoT).
- Destinos: S3, Redshift, OpenSearch, Splunk, Snowflake y endpoints HTTP.
- Opcionales: conversión a Parquet/ORC, descompresión, transformación con Lambda y particionamiento dinámico por atributos del registro.
- ⚠️ La conversión de formato exige entrada **JSON** y toma el esquema de una tabla del **Glue Data Catalog**. Si llega CSV, primero una Lambda lo convierte a JSON.
- Casos: cargar streams en data lakes y warehouses, observabilidad de seguridad (SIEM), enriquecer streams con modelos de ML en tránsito.

**Amazon Kinesis Data Streams (KDS)** ingesta **y almacena** streams para procesamiento personalizado en **tiempo real** (retardo put-to-get < 1 s). Necesita consumidores: Managed Service for Apache Flink, Spark, aplicaciones en EC2 o Lambda, y admite varios en paralelo sobre el mismo stream. Los productores escriben directamente al stream, así que la caída de un servidor de aplicaciones no pierde datos. Casos: logs y feeds, métricas en tiempo real, clickstream, combinar streams en otros nuevos.

**Amazon MSK** es Apache Kafka administrado y de alta disponibilidad: las aplicaciones, herramientas y plugins de Kafka funcionan **sin cambios de código**. Tiene brokers (al menos uno por AZ, cada AZ con su subred) y controladores; los metadatos viven en ZooKeeper o en **KRaft** (en controladores de Kafka, sin costo ni gestión adicional). El **plano de control** (consola, CLI o SDK de AWS) crea y borra clústeres y cambia el número o tipo de brokers; el **plano de datos** (APIs de Kafka) crea topics, produce y consume. **MSK Serverless** aprovisiona y escala la capacidad, gestiona las particiones y cobra por uso. MSK **gestiona y distribuye** streams (event sourcing, agregación de logs, system of record); el análisis en tiempo real es tarea de Flink.

**Amazon Managed Service for Apache Flink** (antes Kinesis Data Analytics) procesa streams en tiempo real de forma **stateful**, con garantía **exactly-once** y tolerancia a fallos; es serverless y se paga por uso. **No tiene almacenamiento propio**: el estado y los datos van en S3, MSK, KDS u otros. Admite consultas continuas sobre streams (no acotados) y batch sobre datos acotados, y **Flink Studio** permite consultas interactivas. Casos: ETL en streaming, feature engineering, pipelines de ML, detección de fraude, geofencing, facturación.

🕒 **Kinesis Data Analytics for SQL** (con la función SQL `RANDOM_CUT_FOREST` para anomalías) está discontinuado. Si aparece en una pregunta, se refiere a ese servicio legado; no lo confundas con el algoritmo Random Cut Forest de SageMaker.

| Necesidad | Servicio |
|---|---|
| Cargar un stream en S3, Redshift, OpenSearch o Splunk con mínima gestión y conversión de formato | Firehose |
| Procesamiento personalizado en tiempo real (< 1 s) con varios consumidores | KDS |
| Kafka existente, event sourcing, system of record de streaming | MSK (Serverless si no se quiere dimensionar) |
| Analítica o ETL stateful en tiempo real, SQL sobre streams, exactly-once | Managed Service for Apache Flink |
| Procesamiento ligero de cada registro sin servidores | Lambda, como complemento de cualquiera |

**Firehose vs. KDS:** Firehose entrega a destinos en segundos sin código consumidor; KDS retiene el stream en tiempo real (< 1 s) para que lo lean tus consumidores. **MSK vs. Flink:** MSK transporta y distribuye eventos (Kafka); Flink los procesa y analiza.

### 1.6 Transferencia y ETL: DataSync y Glue

**AWS DataSync** transfiere archivos u objetos **online** hacia, desde y entre servicios de almacenamiento de AWS, otras nubes y on-premises, con un protocolo propio paralelo y multihilo y una **tarifa plana por GB**.
- Orígenes: NFS, SMB, HDFS y almacenamiento de objetos (Google Cloud Storage, Azure Blob, compatibles con la API de S3).
- Destinos: S3 (cualquier clase, incluidas Glacier Flexible Retrieval y Deep Archive), EFS y las cuatro variantes de FSx.
- Seguridad: cifrado y **validación de integridad** de extremo a extremo, roles IAM y **VPC endpoints** para no atravesar internet.
- Casos: migrar, replicar o archivar datos por red sin scripts propios. Los datos fríos on-premises se archivan directamente en Glacier Flexible Retrieval o Deep Archive; un file system en standby, en EFS o FSx.
- 🕒 DataSync Discovery (analizaba el almacenamiento on-premises y recomendaba la migración) se retiró en 2025.

**AWS Glue** es ETL **serverless** con auto scaling. Sus **crawlers** infieren esquemas y tipos de archivo y los registran en el **Data Catalog**. Genera scripts ETL automáticamente, trae conectores integrados más JDBC personalizado (más de 70 fuentes), usa Spark, PySpark, Scala o **Ray**, admite batch, micro-batch y **streaming**, y ofrece **interactive sessions** desde tu IDE o notebook. Conviene para ETL con demanda de cómputo irregular, volumen impredecible y muchas fuentes, y para descubrir y catalogar datos antes de consultarlos.

**DataSync vs. Glue:** DataSync mueve o migra datos; Glue los transforma e integra.

### 1.7 Amazon S3, sus clases y Athena

S3 es almacenamiento de objetos con escala prácticamente ilimitada, replicado en varios dispositivos y AZ. Es la opción más costo-eficiente y se integra de forma nativa con SageMaker tanto para los **datos de entrenamiento** como para los **artefactos del modelo**. Se elige para data lakes (datos crudos centralizados y de alta disponibilidad), backup y restore con RTO/RPO (replicación y protección de datos), retención por cumplimiento (clases Glacier) y entrenamiento de LLM a gran escala. Los costos se controlan con **políticas de lifecycle** que mueven los objetos entre clases según su patrón de acceso.

| Clase | Clave para el examen |
|---|---|
| Standard | Acceso frecuente |
| Intelligent-Tiering | Patrón de acceso desconocido o cambiante. ⚠️ Tres niveles automáticos: Frequent → Infrequent Access (30 días sin acceso) → Archive Instant Access (90 días), más dos opcionales asíncronos: Archive Access (≥ 90 días) y Deep Archive Access (≥ 180 días). No cobra recuperación |
| Express One Zone | Una sola AZ; latencia consistente de milisegundos de un dígito |
| Standard-IA | Acceso infrecuente pero rápido; multi-AZ |
| One Zone-IA | Acceso infrecuente, sin resiliencia multi-AZ, más barata; datos recreables |
| Glacier Instant Retrieval | Datos de larga vida, raramente accedidos, con recuperación **inmediata** |
| Glacier Flexible Retrieval | Archivo; recuperación de minutos a horas |
| Glacier Deep Archive | La más barata; ⚠️ recuperación Standard ≤ 12 h y Bulk ≤ 48 h |
| S3 on Outposts | S3 on-premises (residencia de datos) |

**Glacier Instant vs. Flexible Retrieval:** «poco acceso pero inmediato» es Instant Retrieval; minutos u horas de espera, Flexible Retrieval.

**Amazon Athena** ejecuta SQL estándar, serverless e interactivo, directamente sobre S3 (CSV, JSON, Parquet, ORC), sin cargar datos ni gestionar infraestructura. «Almacenamiento barato + consultas SQL» → S3 + Athena, no Redshift ni DynamoDB.

### 1.8 Sistemas de archivos y bloque: EFS, FSx y EBS

**Amazon EFS** es un file system NFS (v4.0/4.1) serverless y elástico (petabytes, GB/s), montable en EC2, EKS, ECS y Lambda. Tiene lifecycle hacia las clases Infrequent Access y Archive, protección con AWS Backup y replicación, **consistencia fuerte** y **bloqueo de archivos**. Se elige para datos compartidos entre muchos consumidores con volumen impredecible, o cuando los datos de entrenamiento **ya están** en EFS. Con SageMaker exige trabajo de red (VPC, security groups, IAM), que S3 no necesita, y cuesta más que S3: se elige por la semántica de file system compartido, no por precio.

**Amazon FSx for Lustre** es un file system paralelo de alto rendimiento (latencia sub-ms, cientos de GB/s, millones de IOPS) que actúa como capa sobre un bucket de S3: las instancias de entrenamiento no descargan los datos, así que el arranque y el entrenamiento son más rápidos.
- Carga desde S3: **completa** (dataset disponible pronto, mejor rendimiento y mayor costo inicial) o **lazy loading** (carga al acceder, menor costo inicial y primer acceso más lento).
- Despliegue: **scratch** (efímero, procesamiento de corto plazo) o **persistent** (largo plazo, orientado a throughput).
- Alto throughput + baja latencia + datos en S3 + entrenamiento en SageMaker → FSx for Lustre, desplegado en la misma AZ que el cómputo para no pagar transferencia entre AZ (§8.3).

| Como fuente de entrenamiento | S3 | EFS | FSx for Lustre |
|---|---|---|---|
| Carga y entrenamiento | El más lento | Intermedio | El más rápido |
| Costo | El más barato | Intermedio | El más caro |
| Setup | Mínimo | VPC, security groups, IAM | VPC; vinculado a un bucket S3 |
| Cuándo | Gran escala, largo plazo, costo crítico | Datos ya en EFS, file system compartido | Entrenamiento intensivo, latencia sub-ms |

Otras variantes de FSx:

| Servicio | Base | Caso típico |
|---|---|---|
| FSx for NetApp ONTAP | NFS, SMB, iSCSI, NVMe-over-TCP (SAN + NAS); compresión y deduplicación | Migrar NetApp u otros servidores NFS/SMB/iSCSI sin cambiar código; DR entre regiones; bases de datos sub-ms |
| FSx for Windows File Server | **SMB + Active Directory**; deduplicación | Migrar file servers Windows; SQL Server en alta disponibilidad sin licencia Enterprise; perfiles de WorkSpaces |
| FSx for OpenZFS | NFS v3–v4.2 (clientes Linux, Windows, macOS); más de 1 M de IOPS | Migrar ZFS o file servers Linux sin cambios |

**FSx for Windows vs. ONTAP:** Active Directory o SMB nativo de Windows → Windows File Server; SAN (iSCSI) y NAS en el mismo servicio → ONTAP.

**Amazon EBS** es almacenamiento en bloque para EC2 (una SAN en la nube). Los volúmenes **SSD** sirven para cargas transaccionales (métrica clave: **IOPS**) y los **HDD** para streaming grande (métrica clave: **throughput**). Los **snapshots** son backups point-in-time desde los que se restauran volúmenes nuevos. Casos: bases de datos, migración de SAN, clústeres Hadoop/Spark. No es fuente nativa de entrenamiento en SageMaker.

### 1.9 Bases de datos: RDS y DynamoDB

**Amazon RDS** es una base de datos relacional administrada con 8 motores (Aurora PostgreSQL, Aurora MySQL, PostgreSQL, MySQL, MariaDB, SQL Server, Oracle y Db2): AWS aprovisiona, parchea, respalda, recupera y repara. Variantes: **RDS on Outposts** (híbrido) y **RDS Custom** (acceso privilegiado al SO y la base de datos). Migrar una base de datos legada sin refactorizar (solo cambia la conexión; se mantienen los stored procedures) → RDS con el mismo motor, con ahorro de licencias y pago por uso.

**Amazon DynamoDB** es NoSQL serverless (clave-valor y documento) con latencia de milisegundos de un dígito a cualquier escala. Admite **lecturas fuertemente consistentes** y **transacciones ACID** en una o varias tablas con una sola solicitud, **no tiene JOINs** y conviene usar Query en lugar de Scan. Casos: finanzas y pedidos, gaming (sesiones, leaderboards), media (índices de metadatos, watchlists). Se clasifica como servicio de datos semiestructurados.

### 1.10 Troubleshooting y escenarios tipo

| Problema u objetivo | Acción |
|---|---|
| Anomalías de CPU, memoria o I/O | CloudWatch: métricas y alarmas |
| Causa raíz: quién cambió qué | CloudTrail: registro de llamadas a la API |
| Más tráfico de lectura en la base de datos | Read replicas; auto scaling de Aurora |
| Base de datos lenta | Optimizar queries e índices, sharding; en DynamoDB, evitar Scan |
| Costo de S3 | Políticas de lifecycle entre clases |
| Costo de cómputo | Cost Explorer; Reserved Instances o Savings Plans (carga predecible); Spot (cargas elásticas y efímeras) |

**CloudWatch vs. CloudTrail:** métricas y alarmas frente a auditoría de llamadas a la API.

| Escenario | Respuesta |
|---|---|
| Procesados con acceso inmediato 6 meses; crudos recuperables en ≤ 12 h durante 6 años; consultas SQL; costo mínimo | S3 con lifecycle a Deep Archive + Athena |
| SQL en tiempo real sobre un stream de Firehose con registros comprimidos en GZIP | Managed Service for Apache Flink + Lambda para preprocesar |
| Anomalías en transacciones que llegan en streaming a un data lake en S3, con mínimo overhead | Firehose + `RANDOM_CUT_FOREST` de KDA for SQL 🕒 |
| Datos de geolocalización near real time en Redshift, costo-eficiente | KDS + **Redshift Streaming Ingestion** |
| CSV near real time → Parquet en S3, lo más eficiente | Firehose + Lambda (CSV → JSON) + conversión nativa a Parquet |
| Entrenamiento de imágenes a gran escala con datos en S3, alto throughput y baja latencia | FSx for Lustre |
| Muchos datos de sensores, acceso frecuente, costo-eficiente, para SageMaker | S3 |

⚠️ **Redshift Streaming Ingestion** solo acepta como fuente **KDS o MSK** (y Kafka externo), no Firehose, y no descomprime registros. **Redshift Spectrum** consulta datos en S3; no es ingesta de streaming.

> **Para recordar en el examen**
> - **Firehose** entrega en segundos a S3, Redshift, OpenSearch, Splunk o HTTP sin consumidores; convierte a Parquet/ORC solo desde JSON con esquema del Glue Data Catalog. **KDS** es tiempo real (< 1 s), almacena el stream y requiere consumidores propios.
> - **MSK** = Kafka gestionado sin cambios de código (Serverless si no se quiere dimensionar); **Flink** = análisis o ETL stateful y exactly-once sobre streams, sin almacenamiento propio.
> - **DataSync** mueve datos por red (tarifa por GB, validación de integridad); **Glue** transforma (ETL serverless) y cataloga (crawlers → Data Catalog).
> - Fuentes nativas de entrenamiento: S3 (la más barata), EFS (datos ya ahí o file system compartido; requiere VPC) y FSx for Lustre (la más rápida, capa sobre S3). EBS no.
> - Clases de S3: acceso raro pero inmediato → Glacier Instant Retrieval; ≤ 12 h y años al menor costo → Deep Archive; patrón desconocido → Intelligent-Tiering; datos recreables sin multi-AZ → One Zone-IA.
> - Redshift Streaming Ingestion: solo KDS o MSK, nunca Firehose. «Barato + SQL sobre S3» → Athena.
> - Formatos: Parquet/ORC columnares y Avro por filas; solo RecordIO-protobuf: Factorization Machines y Seq2Seq; DeepAR usa JSON Lines o Parquet; XGBoost, CSV, LibSVM o Parquet.
> - DynamoDB: ACID y lecturas consistentes, pero sin JOINs (Query, no Scan). RDS con el mismo motor: migrar sin refactorizar.

---

## 2. Transformación de datos, feature engineering e integridad

*Dominio 1 · Tareas 1.2 y 1.3*

### 2.1 Data lake y servicios de preparación

Un data lake en AWS combina **S3** (almacenamiento; 11 nueves de durabilidad; *single source of truth* de la mayoría de los servicios de ML), **Glue** (catálogo), **Athena** (consultas) y **Lake Formation** (permisos, seguridad y gobernanza centralizados). Acepta datos crudos en cualquier formato y los protege con IAM y cifrado en reposo, en tránsito y en uso. Acceso desconocido → S3 Intelligent-Tiering; permisos centralizados, monitoreo de accesos y cumplimiento a escala → Lake Formation.

| Necesidad | Servicio |
|---|---|
| Limpieza visual sin código: duplicados, faltantes y outliers (detecta con Z-score o IQR y luego reemplaza, elimina, reescala o marca) | **AWS Glue DataBrew** |
| Feature engineering low-code, series temporales, EDA y visualizaciones, división en splits | **SageMaker Data Wrangler** |
| Reformatear o transformar a escala (ETL) | **Job de AWS Glue** |
| Código propio (pandas, scikit-learn) | Notebooks de SageMaker |
| Almacenar, compartir y reutilizar features | **SageMaker Feature Store** |
| Etiquetado | **SageMaker Ground Truth** 🕒 |
| Sesgo en los datos antes de entrenar | **SageMaker Clarify** 🕒 |
| Gobernanza y permisos del data lake | **Lake Formation** |

**SageMaker Feature Store** es un repositorio totalmente administrado para almacenar, compartir y gestionar features entre equipos durante todo el ciclo de vida. Sincroniza las features offline (entrenamiento batch) con las online (inferencia en tiempo real) y permite reutilizarlas entre modelos y proyectos, con consistencia. **Feature Store vs. Data Wrangler:** Feature Store almacena features; Data Wrangler las transforma. **DataBrew vs. Data Wrangler:** DataBrew limpia datos de forma visual y sin código; Data Wrangler es la opción low-code para crear features, tratar series temporales, explorar y dividir.

Orden habitual del proceso: limpieza → feature engineering → etiquetado → tratamiento del desbalance → split → entrenamiento.

### 2.2 Técnicas de feature engineering

El feature engineering no agrega datos nuevos: hace más útiles los existentes, a menudo con conocimiento del dominio. A más features, más difícil es entrenar bien el modelo.

| Técnica | ¿Reduce dimensionalidad? | Qué hace |
|---|---|---|
| Extracción | Sí | Deriva automáticamente features nuevas de las existentes (común en imagen, audio y texto); en datos estructurados deben poder replicarse en producción |
| Selección | Sí | Conserva las features con más poder predictivo y elimina las irrelevantes y las redundantes (correlacionadas) |
| Creación / transformación | **No** | Genera features a partir de las existentes (una fecha → día, mes, año, día de la semana) |

### 2.3 Faltantes, outliers, duplicados y ruido

**Valores faltantes.** Se **recolectan** más datos si faltan muchos en varias features (es caro y lento: se compara con el costo de un modelo peor), se **imputan** si son pocos y aleatorios (típico de un fallo de ingesta) o se **elimina la feature** si faltan muchos solo en ella. Imputación: media si la distribución es normal; mediana o valor más frecuente si no; valor más frecuente en categóricas. La mayoría de los algoritmos no maneja faltantes por sí sola. Herramientas: `SimpleImputer` en SageMaker y Glue DataBrew.

**Outliers.** Por convención, un outlier está a más de 3σ de la media. Puede ser natural (refleja algo real) o artificial (un error), y no es lo mismo que el ruido (datos erróneos). Se eliminan si son ruido o error, se aplica una transformación logarítmica a toda la feature para acercarlos sin perder su significado, o se imputan con media o mediana si provienen de un error. Para detectarlos:

| Método | Distribución | Regla |
|---|---|---|
| **IQR** | **Sesgada** (robusto al sesgo) | Outlier si < Q1 − 1.5·IQR o > Q3 + 1.5·IQR |
| **Z-score** | **Normal** | Outlier si el valor absoluto de z supera 3 |

Primero se mira el histograma: en datos sesgados el Z-score puede no detectar outliers. Servicios: Data Wrangler y Glue DataBrew.

**Duplicados** → sesgan los resultados y causan overfitting y sesgo si unas entradas se repiten más que otras; la opción más simple, visual y sin código es DataBrew. **Ruido** → el modelo aprende rarezas del training set (overfitting). **Reformatear** = tipos de dato, formatos (fechas) y unidades uniformes, por ejemplo con un job de Glue.

### 2.4 Scaling y transformaciones contra el sesgo

**Normalización vs. estandarización:** la normalización (MinMax) lleva los datos a un rango común, normalmente [0, 1]; se centra en la **escala** y sirve para algoritmos de distancia (k-NN), redes neuronales y una convergencia más rápida del gradient descent. La estandarización (Z-score) deja media 0 y desviación estándar 1; se centra en la **distribución** y conviene con datos aproximadamente normales (regresión lineal o logística). «Que ninguna feature domine por su magnitud» → normalización. **Ninguna de las dos corrige el sesgo.**

| Técnica (scikit-learn) | Cuándo |
|---|---|
| `MinMaxScaler` | Rango fijo [0, 1]; preserva las relaciones |
| `StandardScaler` | Datos normales; μ y σ se ven afectados por outliers |
| `RobustScaler` (mediana e IQR) | **Outliers** y datos sesgados |
| `MaxAbsScaler` ([−1, 1]) | Datos **dispersos** o con muchos ceros; preserva los ceros |
| `PowerTransformer` | Estabilizar la varianza y acercar a la normal: **Box-Cox solo admite valores positivos**; **Yeo-Johnson admite positivos y negativos** |

La **transformación logarítmica** reduce el sesgo a la derecha, limita el impacto de los outliers sin eliminarlos y linealiza relaciones multiplicativas, pero no admite 0 ni negativos (hay que desplazar los datos o usar raíz cúbica). La **raíz cuadrada** (valores ≥ 0) y la **raíz cúbica** (cualquier real) tienen un efecto menor que el log.
- Corrigen el sesgo: log, raíz cuadrada, raíz cúbica, Box-Cox y Yeo-Johnson. No lo corrigen: MinMax y Z-score.
- Manejan outliers: eliminarlos, log, imputación, Robust scaling y **binning** (convierte un numérico continuo en intervalos categóricos y hace el modelo más robusto a outliers y ruido).

### 2.5 Encoding de variables categóricas

| Técnica | Úsala para | Riesgo |
|---|---|---|
| Label | Datos **ordinales**; modelos basados en árboles | Impone un orden artificial a las nominales |
| One-hot | Nominales de **baja cardinalidad**; modelos no basados en árboles (lineales, k-NN, redes) | Explosión de dimensionalidad |
| Binary | Reducir las columnas del one-hot (≈ log₂ k columnas) | Difumina las distinciones entre categorías |
| Feature hashing | **Alta cardinalidad**, datasets grandes, memoria acotada, costo-eficiente (vector de tamaño fijo n) | Colisiones, mitigadas con suficientes buckets |

Para mitigar la dimensionalidad del one-hot → binary encoding o PCA después del one-hot. «Convertir categóricas en numéricas para mejorar la precisión» o «convertir una columna en valores binarios» → one-hot.

### 2.6 Series temporales, imágenes y texto

**Series temporales en Data Wrangler** (low-code): primero se usan las visualizaciones para entender los patrones. Luego, **Featurize datetime** como primer paso (desagrega el timestamp en `date_month`, `date_day`, `date_week_of_year`, `date_day_of_year` y `date_quarter`), one-hot de los componentes de fecha, **lag features** (valores previos del target, que capturan la autocorrelación) y **rolling window features** (estadísticas por ventana; extracción automática con el paquete **tsfresh**). Los lags y las ventanas solo aplican a series temporales.

**Imágenes**: la vía principal para extraer features son los modelos de visión preentrenados de **SageMaker JumpStart** («mejorar las features de un modelo que reconoce autos» → JumpStart). **Rekognition** hace el análisis básico (objetos, escenas, texto, rostros; etiquetas y bounding boxes sin trabajo manual), Data Wrangler transforma (normalización de píxeles, PCA) y Feature Store almacena.

**Texto**: **Comprehend** extrae features de alto nivel (entidades, frases clave, sentimiento, idioma); **Textract** extrae el texto de documentos; los word embeddings (Word2Vec) y el modelado de temas (LDA) personalizados se hacen con algoritmos de SageMaker. **Comprehend vs. Textract:** analizar texto frente a extraerlo de documentos.
- Técnicas: tokenización (primer paso), eliminación de stop words, n-grams y **word embeddings** (vectores continuos con significado semántico, como Word2Vec o GloVe). «Palabras → vectores con semántica» → embeddings, no one-hot ni tokenización.
- **Stemming vs. lematización:** recorte mecánico de prefijos y sufijos frente a reducción a la forma base con reglas lingüísticas.
- Los FMs de Bedrock tokenizan el prompt y se cobran por **tokens de entrada + tokens de salida**: un prompt más largo cuesta más.

### 2.7 Etiquetado con SageMaker Ground Truth 🕒

Flujo: datos crudos en S3 → labeling job (tarea, instrucciones y tipo de tarea: clasificación de imagen o texto, detección de objetos…) → etiquetado automático opcional → revisión humana → etiquetas versionadas en S3 o Feature Store → entrenamiento. Fuerza laboral: **Mechanical Turk** (crowdsourcing global), privada o proveedores externos. Control de calidad: **consensus labeling** (varios anotadores etiquetan el mismo dato) y **audit sampling** (expertos revisan un subconjunto).

⚠️ **Automated data labeling** es **active learning**, no «etiquetar con modelos preentrenados»: una muestra aleatoria va a humanos → con esas etiquetas se entrena y valida un modelo → el modelo etiqueta solo lo que supera un **umbral de confianza** → lo de baja confianza vuelve a humanos y el ciclo se repite. Reduce costo y tiempo, pero genera costos de entrenamiento e inferencia, y solo existe para algunos tipos de tarea integrados.

### 2.8 Desbalance de clases y sesgo previo al entrenamiento

| Técnica | Cómo | Contra |
|---|---|---|
| **Data augmentation** (la más común y recomendada) | Más ejemplos de la minoritaria derivados de los existentes: rotaciones, flips o color en imágenes; sinónimos o inserción aleatoria en texto | Costo, cómputo y tiempo |
| Oversampling | Duplicar o sintetizar ejemplos de la minoritaria; **SMOTE** interpola entre ellos | — |
| Undersampling | Eliminar al azar ejemplos de la mayoritaria | **Pérdida de información** |
| Class weighting | Más peso en la función de pérdida a los errores sobre la minoritaria | — |

Primero se intenta data augmentation y, si no es viable, las demás. **Data augmentation vs. datos sintéticos:** la primera deriva datos del training set; los segundos se generan sin usar el dataset original.

**SageMaker Clarify: métricas de sesgo pre-entrenamiento** 🕒. Se calculan sobre los datos antes de entrenar, son agnósticas al modelo y se configuran con `BiasConfig`. El **facet** es la feature que se analiza: *a* es el valor favorecido y *d* el desfavorecido. En todas las métricas, un valor cercano a 0 indica ausencia de sesgo.
- **Class Imbalance (CI)** = (nₐ − n_d)/(nₐ + n_d), en [−1, +1]: desbalance en el **número de miembros** de cada valor del facet (+1 = solo miembros de *a*).
- **Difference in Proportions of Labels (DPL)** = qₐ − q_d: diferencia en la proporción de **etiquetas positivas** entre facets (0 = paridad demográfica).
- **CI vs. DPL:** representación de los grupos frente a resultados de la etiqueta.
- La métrica orienta la acción: oversampling, undersampling o class weighting. Ejemplo: 99.9 % de transacciones no fraudulentas → CI ≈ +1 → rebalancear.
- ⚠️ EOD y PPD no son métricas pre-entrenamiento (requieren predicciones del modelo). Las pre-entrenamiento son CI, DPL, KL, JS, LP, TVD, KS y CDD.

### 2.9 Splits y data leakage

**Training** (60–80 %) ajusta los parámetros; **validation** (10–20 %) ajusta hiperparámetros y detecta overfitting; **test** (10–20 %) da la evaluación final imparcial con datos nunca vistos. El propósito del split es **prevenir el overfitting y evaluar el rendimiento**. Hay **data leakage** cuando información del test se filtra al entrenamiento e infla las métricas. Se evita con separación estricta, validación cruzada, vigilancia del preprocesamiento y **escalado después del split** (el scaler se ajusta solo con train y se aplica a validation y test).

La transformación **Split data** de Data Wrangler crea 2 o 3 subconjuntos con porcentajes configurables (p. ej., 70/20/10):

| Método | Garantiza | Cuándo |
|---|---|---|
| Random | Muestras aleatorias sin solapamiento; ⚠️ no preserva las proporciones de clase | El orden no importa |
| Ordered | Conserva el orden (el primer X % va a train) | **Series temporales** |
| Stratified | La misma proporción de clases en cada split | **Clasificación desbalanceada** |
| Split by key | Ninguna combinación de claves aparece en más de un split (p. ej., un `customer_id`) | Evitar leakage en datos no ordenados manteniendo juntos los registros relacionados |

Escenarios tipo: faltantes en features categóricas sin distorsionar los datos → **imputación** (no eliminar la feature, no recolectar, no Clarify); magnitudes muy distintas → **normalización**; alta cardinalidad con eficiencia y bajo costo → **feature hashing**; técnica más común contra el desbalance → **data augmentation**; etiquetado → **Ground Truth** (no Comprehend, Rekognition ni Clarify).

> **Para recordar en el examen**
> - Glue DataBrew (limpieza visual sin código) vs. Data Wrangler (features low-code, series temporales, EDA, split) vs. job de Glue (ETL a escala) vs. Feature Store (guardar y compartir features online/offline; no transforma).
> - Distribución sesgada → IQR, `RobustScaler`, log, raíces, Box-Cox (solo positivos) o Yeo-Johnson (admite negativos). MinMax y Z-score no corrigen el sesgo.
> - Normalización = escala [0, 1] (k-NN, redes); estandarización = media 0 y σ 1 (datos normales). Datos dispersos → `MaxAbsScaler`.
> - Encoding: ordinal o árboles → label; nominal de baja cardinalidad → one-hot; alta cardinalidad + costo → feature hashing; reducir las columnas del one-hot → binary o PCA.
> - Desbalance: primero data augmentation; después oversampling (SMOTE), undersampling (pierde información) o class weighting.
> - Clarify pre-entrenamiento: CI (cuántos miembros por grupo) vs. DPL (proporción de etiquetas positivas); ~0 = sin sesgo; EOD y PPD son post-entrenamiento.
> - Split: escalar después del split; series temporales → ordered; misma entidad → split by key; desbalance → stratified (random no preserva proporciones).
> - Ground Truth: automated labeling = active learning con umbral de confianza; calidad con consensus labeling y audit sampling.

---

## 3. Servicios de IA de AWS y Amazon Bedrock

*Dominio 2 · Tarea 2.1 (y su consumo por API en el Dominio 3)*

Los servicios de IA son modelos **preentrenados y totalmente gestionados** que se consumen mediante APIs, sin experiencia profunda en ML, y escalan automáticamente. AWS posee y opera el modelo y su infraestructura: no lo ves ni lo despliegas, simplemente llamas a una API, y tu acceso al modelo es limitado. Son rápidos de adoptar pero poco flexibles para casos especializados (para eso están los algoritmos de SageMaker, §4). La excepción es **Personalize, que no ofrece modelos preentrenados**: entrena uno con tus datos y lo expone mediante una **campaña**.

| Servicio | Qué resuelve | Detalles que desempatan |
|---|---|---|
| **Rekognition** | Imágenes y vídeo: objetos, escenas, actividades, texto, celebridades y **análisis facial** (detección, comparación, reconocimiento, emociones), en tiempo real o sobre contenido almacenado | Moderación de contenido, verificación de identidad, catalogación de medios, alertas en hogares conectados |
| **Textract** | Texto, **formularios (pares clave-valor) y tablas** de documentos escaneados, conservando estructura y relaciones (no es OCR tradicional) | Síncrono (`analyze_document`, `detect_document_text`) solo para imágenes y PDF de una página; los multipágina van en asíncrono (`start_document_analysis` + `get_document_analysis`). Resultados a S3, Athena o SageMaker |
| **Polly** | Texto → voz (TTS), en tiempo real y con baja latencia | **SSML** controla tono, velocidad y pronunciación; IVR, audiolibros, accesibilidad, e-learning |
| **Transcribe** | Voz → texto (ASR), por lotes o en streaming | Identificación de hablantes, puntuación automática y **vocabularios personalizados**; subtítulos, call centers (análisis y cumplimiento); el batch es un job asíncrono |
| **Translate** | Traducción automática neuronal, en tiempo real o por lotes | Conserva el contexto |
| **Comprehend** | NLP sobre texto no estructurado: sentimiento, frases clave, entidades, idioma y organización por temas | **Modelos personalizados** de clasificación y de reconocimiento de entidades; ⚠️ entidades médicas en historiales clínicos → **Comprehend Medical** |
| **Lex** | Chatbots de voz y texto (ASR + NLU, la tecnología de Alexa) | *Intents*, *slots* y respuestas definidos en consola; lógica en **Lambda**; conversaciones multiturno; integración nativa con **Amazon Connect** (agentes automáticos 24/7) |
| **Personalize** | Recomendaciones personalizadas en tiempo real | Ver abajo |
| **Bedrock** | IA generativa con modelos fundacionales (FMs) | Ver §3.1 |

**Rekognition vs. Textract:** analizar el contenido visual de imágenes y vídeo frente a extraer texto, formularios y tablas de documentos. **Polly vs. Transcribe:** texto a voz frente a voz a texto.

**Amazon Personalize** entrena con **interacciones** usuario-ítem (clics, compras), **metadatos de ítems** y **metadatos de usuarios**, y se actualiza con eventos en tiempo real. Eliges la *recipe* según el caso: filtrado colaborativo, basado en contenido o híbrido. La gestión usa el cliente `personalize` (p. ej., `create_campaign`) y las recomendaciones, `personalize-runtime` (`get_recommendations`, `get_personalized_ranking`). ⚠️ No genera datos sintéticos para «rellenar» datos dispersos: el *cold start* se resuelve con los metadatos y la exploración de ítems nuevos. Sus funciones generativas son Content Generator (temas descriptivos para recomendaciones batch), la devolución de metadatos de las recomendaciones para enriquecer prompts de un LLM y la integración con LangChain.

### 3.1 Amazon Bedrock

Bedrock es un servicio **serverless** y totalmente gestionado para construir y escalar aplicaciones de IA generativa con FMs de AI21 Labs, Anthropic, Cohere, Meta, Mistral AI, Stability AI y Amazon (Titan y Nova), todos mediante **una sola API**. Permite personalizar los FMs con datos propios (**fine-tuning** y **RAG**), y **Bedrock Marketplace** agrega FMs especializados por dominio (salud, finanzas…).

| Forma de uso | Cuándo |
|---|---|
| `InvokeModel` (cliente `bedrock-runtime`) | Inferencia directa con el formato de petición de cada modelo |
| **Converse API** (`bedrock-runtime`) | Interacciones conversacionales: mensajes con rol y el mismo formato para cualquier modelo compatible (no todos lo son). Chatbots, asistentes y atención al cliente en tiempo real; alternativa sencilla cuando un agente sería excesivo |
| **Agents** (su alias se invoca con `bedrock-agent-runtime`) | Flujos complejos de varios pasos: el agente usa el FM para interpretar la petición, dividirla en pasos y ejecutar las acciones configuradas (p. ej., llamar a APIs de la organización) |

**Cómo elegir un FM**
1. **Capacidades**: texto (generación, resumen, Q&A, código, chat) → Amazon Titan Text G1 (Premier, Express, Lite); imágenes a partir de texto → **Nova Canvas**; multimodal con entrada de texto e imagen → **Nova Lite** o **Nova Pro**; solo texto con baja latencia y bajo costo → **Nova Micro**. Conviene revisar el estado de ciclo de vida del modelo (Active, Legacy, EOL).
2. **Región**: los FMs se ofrecen por región. Si el modelo no está en la tuya, se usa **inferencia entre regiones** (*cross-Region inference*), que reparte el tráfico entre varias regiones, absorbe picos, da más throughput y resiliencia y no cobra por el enrutamiento. ⚠️ No hace falta crear un perfil: se invoca el **perfil de inferencia entre regiones predefinido por el sistema** (el ID del modelo con prefijo geográfico, p. ej., `us.`). Los **perfiles de inferencia de aplicación**, que sí creas tú, son opcionales y sirven para seguir costos y uso.
3. **Precio**: **On-Demand y Batch** cobran por tokens de entrada y de salida (los modelos de imagen, como Nova Canvas, por imagen generada); **Provisioned Throughput** es un compromiso con un nivel de rendimiento durante un periodo y resulta más rentable con uso constante.

🕒 Desde octubre de 2025 los modelos serverless de Bedrock están habilitados por defecto: ya no se «solicita acceso» en la consola (los de Anthropic solo piden rellenar una vez un formulario de uso) y el acceso se restringe con IAM y SCP.

**Modelos de embeddings de Bedrock** (p. ej., `cohere.embed-english-v3`, `cohere.embed-multilingual-v3`): generan embeddings contextuales de frases o párrafos para búsqueda semántica y RAG. ⚠️ Un modelo de embeddings no responde preguntas ni conversa; eso lo hace un FM de generación de texto, al que los embeddings alimentan vía RAG.

**Bedrock vs. SageMaker AI:** Bedrock no es la mejor opción si el modelo no está en Bedrock ni en su Marketplace, si necesitas control total del entrenamiento (p. ej., entrenar desde cero), de la arquitectura o de la infraestructura (→ SageMaker AI o un modelo propio), si hay requisitos de latencia muy específicos o si basta una solución más simple y barata. Aun así, Bedrock sí permite personalizar FMs con fine-tuning y RAG.

> **Para recordar en el examen**
> - Rekognition = imagen y vídeo (rostros, moderación, objetos); Textract = texto, formularios y tablas de documentos conservando su estructura (multipágina → API asíncrona).
> - Polly = texto → voz (SSML); Transcribe = voz → texto (vocabularios personalizados, hablantes); Translate = traducción; Lex = chatbots (intents, slots, Lambda, Amazon Connect).
> - Comprehend analiza texto (sentimiento, entidades, frases clave, idioma; admite modelos personalizados); para datos clínicos, Comprehend Medical.
> - Personalize no es preentrenado: entrena con tus interacciones y metadatos y sirve las recomendaciones vía campaña (`personalize-runtime`); el cold start se resuelve con metadatos y exploración, no con datos sintéticos.
> - Bedrock: Converse API para conversación con formato único entre modelos; Agents para tareas de varios pasos que ejecutan acciones; `InvokeModel` para inferencia directa.
> - Modelo no disponible en tu región → cross-Region inference con el perfil predefinido (`us.`…), sin costo extra de enrutamiento.
> - Precio de Bedrock: On-Demand/Batch por tokens de entrada + salida (imágenes, por imagen); uso constante → Provisioned Throughput.
> - Si se requiere control total del entrenamiento, la arquitectura o la infraestructura, la respuesta es SageMaker AI, no Bedrock ni un servicio de IA.

---

## 4. Algoritmos integrados de SageMaker AI y selección de modelos

*Dominio 2 · Tareas 2.1 y 2.2*

Los algoritmos integrados (*built-in*) convienen cuando un servicio de IA no basta: feature engineering propio, arquitecturas u objetivos de optimización específicos, ajuste de hiperparámetros o preprocesamiento con conocimiento del dominio. Están optimizados para velocidad, escala y exactitud, leen datos de **S3, EFS o FSx for Lustre**, admiten entrenamiento distribuido y se sirven en endpoints de SageMaker o con batch transform.

**Flujo común**: datos en S3 (o EFS/FSx) → *estimator* con la imagen del contenedor del algoritmo (URI por región), el rol IAM, el tipo y número de instancias y la ruta de salida en S3 → hiperparámetros → **training job** (SageMaker aprovisiona, escala y libera la infraestructura) → endpoint en tiempo real o **batch transform**. Los formatos de entrada están en §1.3.

### 4.1 Supervisados

| Algoritmo | Problema | Hiperparámetros y particularidades |
|---|---|---|
| **Linear Learner** | Regresión y clasificación binaria o multiclase | `predictor_type` obligatorio (`binary_classifier`, `multiclass_classifier` o `regressor`); `num_classes` solo en multiclase; ⚠️ `feature_dim` es opcional (`auto`). Con `loss` cubre regresión lineal (`squared_loss`), logística (`logistic`) y **SVM lineal** (`hinge_loss`); regularización con `l1` y `wd`. Muy interpretable |
| **k-NN** | Clasificación (voto mayoritario) y regresión (media de los vecinos) | Número de vecinos `k` y métrica de distancia. Entrenar es barato, pero inferir con muchos datos es caro en cómputo y memoria; requiere escalar las features |
| **XGBoost** | Clasificación y regresión con *gradient boosting* de árboles | `num_round` **obligatorio**; `objective` (`binary:logistic`, `reg:squarederror`, `multi:softprob`, o `multi:softmax` con `num_class`), `eta`, `max_depth`, `subsample`, `colsample_bytree`, `alpha`, `min_child_weight`. Da la importancia de las features |
| **LightGBM, CatBoost** | Árboles (boosting) | Los otros algoritmos integrados basados en árboles |
| **Factorization Machines** | Clasificación binaria y regresión con datos **dispersos de alta dimensión**: recomendación y **predicción de CTR** | Obligatorios `predictor_type` (`binary_classifier` o `regressor`), `num_factors` y `feature_dim`. ⚠️ `bias_lr`, `linear_lr` y `factors_lr` son *learning rates*; la regularización son `bias_wd`, `linear_wd` y `factors_wd` |
| **Object2Vec** | **Embeddings** densos y de baja dimensión de objetos diversos (generaliza Word2Vec); supervisado con pares de objetos etiquetados con su relación | `enc0_max_seq_len`, `enc0_vocab_size`, `enc_dim`, `dropout`, `early_stopping_patience`, `comparator_list`. Para recomendación, búsqueda por similitud, clustering o features de otros modelos |
| **DeepAR** | Pronóstico de series temporales con RNN | Un solo modelo para **muchas series relacionadas**; pronósticos **probabilísticos** (distribución, no un valor). Obligatorios `context_length`, `prediction_length`, `epochs` y ⚠️ `time_freq` |

⚠️ **Random Forest no es un algoritmo integrado**: se entrena con un script propio (p. ej., scikit-learn). No lo confundas con **Random Cut Forest** (anomalías). Si una opción ofrece «Random Forest integrado», desconfía. Tampoco hay SVM con kernel integrada: Linear Learner solo implementa SVM lineal.

**Cuándo usar cada enfoque**

| Modelo | Úsalo | Evítalo |
|---|---|---|
| Regresión lineal | Relación lineal con target continuo; se necesita transparencia | No linealidad, muchos outliers o multicolinealidad (se mitiga con regularización) |
| Regresión logística | Clasificación binaria con probabilidad de clase e interpretabilidad | Relaciones no lineales, multicolinealidad, pocos datos |
| SVM | Alta dimensión, clases separadas por un margen claro, más features que muestras (datasets de hasta ~10 000 muestras) | Datasets grandes, datos ruidosos o clases solapadas |
| k-NN | Datos pequeños o moderados (hasta ~10 000) y de pocas dimensiones; fronteras no lineales; predicciones fáciles de explicar | Datasets grandes, alta dimensionalidad, muchas features irrelevantes |
| Árbol de decisión | La interpretabilidad es crítica; poco preprocesamiento | Datos grandes y ruidosos, o se necesita máxima exactitud (sobreajusta) |
| Random forest | Más exactitud y menos sobreajuste que un solo árbol; robusto con la configuración por defecto | Tiempo real o recursos limitados con muchos árboles profundos |
| XGBoost | Máxima exactitud y eficiencia en tabulares grandes con interacciones complejas | No se puede invertir en ajustar hiperparámetros, o la tarea es simple y prima la interpretabilidad |
| DeepAR | Muchas series relacionadas a gran escala, cuantificando la incertidumbre (demanda, inventario) | Una sola serie simple (bastan ARIMA o suavizado exponencial) |

**Random forest vs. XGBoost:** árboles independientes entrenados en paralelo que reducen sobre todo la varianza y son poco sensibles a los hiperparámetros, frente a árboles secuenciales que corrigen los errores previos, reducen sobre todo el sesgo, suelen ser más precisos en tabulares complejos y exigen un ajuste cuidadoso. ⚠️ XGBoost controla el sobreajuste con regularización L1/L2, *shrinkage* (`eta`) y submuestreo, pero aun así puede sobreajustar.

**Factorization Machines vs. Object2Vec:** interacciones entre features en datos dispersos (recomendación, CTR) frente a embeddings de objetos aprendidos a partir de pares relacionados.

### 4.2 No supervisados

| Algoritmo | Tarea | Hiperparámetros y particularidades |
|---|---|---|
| **K-means** | Clustering en `k` grupos predefinidos (distancia euclídea; minimiza la distancia al centroide) | `k` y `feature_dim` **obligatorios**; `init_method` (`random` por defecto o `kmeans++`). `k` se elige con el **método del codo** (WCSS frente a `k`). Para clústeres esféricos y bien separados; evitar con formas, tamaños o densidades variables, muchos outliers o features no numéricas (→ DBSCAN o clustering jerárquico) |
| **PCA** | Reducción de dimensionalidad: componentes ortogonales de máxima varianza; mitiga multicolinealidad y sobreajuste | `feature_dim`, `num_components` y `mini_batch_size` **obligatorios**; datos densos y dispersos; responde en JSON, JSON Lines o RecordIO (no CSV); endpoint o batch transform. Supone relaciones lineales, trata la varianza como importancia y sus componentes son **poco interpretables**. t-SNE (preserva la estructura local) no es integrado |
| **LDA** | Modelado de temas probabilístico: cada documento es una mezcla de temas y cada tema, una mezcla de palabras | Se fija el número de temas; transforma documentos nuevos en su distribución de temas. Más simple y barato; suficiente con corpus pequeños o recursos limitados |
| **NTM** | Modelado de temas con redes neuronales | Más flexible en corpus grandes y complejos. ⚠️ Igual que LDA, trabaja con bolsas de palabras y **no modela el orden** |
| **Random Cut Forest (RCF)** | Anomalías en datos generales, grandes y de alta dimensión: fraude, intrusiones, mantenimiento predictivo | `num_trees` y `num_samples_per_tree`; detección en tiempo real desde un endpoint. ⚠️ Una anomalía queda aislada **cerca de la raíz** (poca profundidad) y recibe **puntuación alta**. Si las anomalías están etiquetadas, es mejor un clasificador supervisado |
| **IP Insights** | Uso anómalo de direcciones **IPv4** por entidad (usuario, cuenta) | Entrena con pares (entidad, IPv4) y devuelve una puntuación de anomalía; aprende embeddings reutilizables. Logins fraudulentos, cuentas comprometidas, recursos creados desde IPs inusuales. Falla si las IPs cambian con frecuencia |

**k-NN vs. K-means:** supervisado (predice con los vecinos más cercanos) frente a no supervisado (agrupa en `k` clústeres). **LDA vs. NTM:** Dirichlet, más simple y barato, frente a neuronal, más flexible en corpus grandes. **RCF vs. IP Insights:** anomalías en cualquier dato frente a pares entidad–IPv4.

En PCA, lo que decide la respuesta es «no supervisado + máxima varianza», aunque una pregunta lo califique de «muy interpretable».

### 4.3 Texto y visión

**BlazingText** es una implementación muy escalable (multihilo y GPU) de **Word2Vec** (no supervisado; `mode` = `cbow`, `skipgram` o `batch_skipgram`) y de **clasificación de texto** (supervisado; `mode` = `supervised`). ⚠️ Su clasificador extiende **fastText**, un modelo ligero, no un deep learning complejo; por eso es tan rápido. Sirve para sentimiento, categorización de documentos y similitud de palabras a gran escala. Si se necesita contexto más allá de la palabra (frases, párrafos, búsqueda semántica, RAG), la opción son los embeddings de Bedrock (§3.1).

**Seq2Seq** es supervisado: transforma una secuencia de tokens (texto o audio) en otra secuencia, con codificador-decodificador RNN o CNN y mecanismos de atención. Se usa en traducción, resumen, voz a texto y respuestas de chatbot, no en clasificación simple. **BlazingText vs. Seq2Seq:** embeddings de palabras o clasificación frente a generación de secuencias.

| Algoritmo | Salida | Particularidades | Casos |
|---|---|---|---|
| **Image Classification** | Una etiqueta para toda la imagen (CNN) | ⚠️ Solo dos variantes: MXNet y TensorFlow (transfer learning con TF Hub); no existe variante PyTorch | Diagnóstico por imagen, organizar productos |
| **Object Detection** | **Cajas delimitadoras** + clase de cada objeto | ⚠️ La variante MXNet usa **SSD** con base VGG-16 o ResNet-50; la TensorFlow hace transfer learning. No implementa R-CNN ni YOLO (están como modelos de JumpStart). Aumento de datos integrado (volteo, reescalado, *jittering*) | Conducción autónoma, vigilancia, contar productos en estanterías |
| **Semantic Segmentation** | Una clase **por píxel** (mapa de segmentación) | Backbone ResNet-50 o ResNet-101; el hiperparámetro `algorithm` admite `fcn` (por defecto), `psp` y `deeplab` (⚠️ no U-Net). Requiere una máscara por imagen y muchas imágenes anotadas | Tumores u órganos, carretera frente a acera, cultivos frente a malas hierbas, imágenes satelitales |

Si hay que extraer texto o información estructurada de documentos (OCR), la respuesta es Textract, no un algoritmo de visión. Los modelos de embeddings de imagen de JumpStart no son algoritmos integrados.

### 4.4 Criterios de selección

| Criterio | Ejemplos |
|---|---|
| Exactitud | XGBoost y SVM en clasificación y regresión |
| Interpretabilidad | Linear Learner, regresión logística, árboles de decisión |
| Escalabilidad | Linear Learner, K-means; RCF para anomalías en big data |
| Latencia | Modelos ligeros (p. ej., random forest con pocos árboles) |
| Recursos | Linear Learner y PCA consumen menos que las redes profundas; k-NN entrena barato pero infiere caro |
| Datos disponibles | LDA y BlazingText rinden con corpus grandes; K-means funciona con datasets más pequeños |
| Regulación y ética | Regresión logística y árboles, por su transparencia |
| Costo | Linear Learner es barato; las CNN son costosas |

Estos criterios suelen estar en tensión, sobre todo exactitud frente a interpretabilidad: hay que equilibrarlos según los objetivos del negocio y reevaluar el modelo con datos nuevos.

> **Para recordar en el examen**
> - Random Forest no es integrado (script propio); los integrados de árboles son XGBoost, LightGBM y CatBoost. Random Cut Forest es otra cosa: anomalías.
> - Linear Learner = regresión lineal, logística o SVM lineal según `loss`; `predictor_type` obligatorio (`num_classes` en multiclase). La SVM con kernel no está integrada.
> - k-NN (supervisado, vecinos) vs. K-means (no supervisado; `k` y `feature_dim` obligatorios; método del codo).
> - Hiperparámetros obligatorios: XGBoost `num_round`; DeepAR `context_length`, `prediction_length`, `epochs` y `time_freq`; PCA `feature_dim`, `num_components` y `mini_batch_size`; Factorization Machines `predictor_type`, `num_factors` y `feature_dim`.
> - Datos dispersos de recomendación o CTR → Factorization Machines; embeddings a partir de pares relacionados → Object2Vec; muchas series relacionadas con incertidumbre → DeepAR.
> - Anomalías: RCF (puntuación alta = punto aislado cerca de la raíz) vs. IP Insights (pares entidad–IPv4).
> - Texto: BlazingText (Word2Vec o clasificación tipo fastText, muy escalable) vs. Seq2Seq (traducción, resumen); temas: LDA (simple) o NTM (neuronal); ninguno de los dos modela el orden.
> - Visión: Image Classification (imagen completa) vs. Object Detection (cajas, SSD) vs. Semantic Segmentation (cada píxel; `fcn`, `psp`, `deeplab`).

---

## 5. Entrenamiento, ajuste de hiperparámetros y evaluación

*Dominio 2 · Tareas 2.2 y 2.3*

### 5.1 Entrenamiento local, remoto y distribuido

| Enfoque | Qué es | Para qué |
|---|---|---|
| Local | Una sola máquina con sus recursos | Probar algoritmos, features e hiperparámetros con un **subconjunto** de los datos; feedback inmediato y depuración |
| Remoto | Training job en la nube (SageMaker) o en un servidor remoto | Entrenar con el **dataset completo** sin depender del hardware local |
| Distribuido | La carga se reparte entre varias máquinas o nodos, local o remotamente | Datasets más grandes y menos tiempo, paralelizando los cálculos |

- Se prefieren **scripts de Python** versionados en Git a los notebooks: un notebook **escala verticalmente pero no horizontalmente** (no sirve para entrenamiento distribuido) y se versiona mal.
- Los **contenedores** dan un entorno reproducible y portable que después se despliega y escala en otros entornos (p. ej., EKS). Docker es el runtime más usado y recomendado, pero como ECR soporta el estándar **OCI**, sirve cualquier runtime compatible (containerd, CRI-O).
- **Training job remoto** con el SageMaker Python SDK: un `Estimator` (URI de la imagen, rol IAM, tipo y número de instancias, `volume_size`, `max_run` e hiperparámetros) y `fit()` con los canales de datos en S3 (`train`, `validation`). Como servicio totalmente administrado, SageMaker aprovisiona las instancias y las **apaga al terminar** el job, sin cargos adicionales.
- **`max_run`** fija un timeout en segundos: al cumplirse, SageMaker termina el job sea cual sea su estado. Por defecto vale **86 400 s** (24 h).
- El artefacto resultante (`model.tar.gz`) queda en `<output_path>/<nombre-del-job>/output/`.
- **SageMaker Studio** es el IDE web para ML (Code Editor basado en VS Code, JupyterLab): se crea un **dominio** en la región y un **espacio**, la instancia gestionada que ejecuta el editor, que conviene **detener** al terminar para no pagar. ⚠️ El almacenamiento de un espacio es un volumen **EBS** (5 GB por defecto, ampliable) que persiste entre sesiones; EFS era el almacenamiento de Studio Classic. Las notas del cap. 5 dicen EFS; prevalece la corrección del cap. 4.

### 5.2 Monitoreo y depuración de training jobs

- **CloudWatch** muestra en tiempo real las métricas definidas desde la consola o el SDK (training loss, accuracy, uso de recursos): SageMaker las envía automáticamente como series de tiempo, se pueden consultar por programa y permiten detener a tiempo un job que no aprende. Los **valores finales** se obtienen con **`DescribeTrainingJob`**. Los logs se analizan con CloudWatch Logs Insights (§9.1).
- **EventBridge** permite reglas que detectan training jobs fallidos y disparan respuestas automáticas: notificar al equipo, reiniciar el job o hacer rollback a una versión anterior del modelo. Admite **dead-letter queues** para capturar eventos fallidos y reintentos.
- **SageMaker Debugger** tiene dos funciones. El *debugging* captura automáticamente **tensores y métricas intermedios** para diagnosticar problemas como **vanishing gradients** o un preprocesamiento incorrecto. El *profiling* mide el uso de **CPU, GPU, memoria y red**, detecta cuellos de botella con **reglas integradas** y ofrece visualizaciones y reportes, lo que reduce costo y tiempo. La librería open source **`smdebug`** añade reglas personalizadas y la captura de tensores concretos en TensorFlow, PyTorch y MXNet. Es la respuesta habitual para monitorear y depurar training jobs distribuidos.

**Debugger vs. CloudWatch:** el estado interno del modelo y el profiling detallado del entrenamiento frente a métricas y alarmas generales del job.

### 5.3 Ajuste de hiperparámetros: SageMaker AI Automatic Model Tuning (AMT)

El ajuste manual (lanzar varios jobs y comparar) consume mucho tiempo y recursos. **AMT** busca automáticamente la mejor combinación evaluando varias configuraciones **en paralelo**. Un **tuning job** define:
- **Rangos** de hiperparámetros (p. ej., `ContinuousParameter` e `IntegerParameter`). Cada algoritmo integrado tiene sus propios hiperparámetros (documentación de cada algoritmo).
- **Métrica objetivo**, que depende del tipo de problema (p. ej., `validation:accuracy`), y si se maximiza o minimiza (`objective_type`).
- **Límites de recursos**: `max_jobs` (training jobs totales) y `max_parallel_jobs` (simultáneos). Con 20 y 3 se ven 3 jobs a la vez y 20 en total.
- **Estrategia**: **bayesiana** (modela la relación entre hiperparámetros y métrica objetivo y predice qué combinación probar a continuación; suele requerir **menos evaluaciones**, sobre todo en espacios grandes), **random search** (muestreo aleatorio) o **grid search** (recorrido exhaustivo de un espacio predefinido).
- **Warm start**: reutiliza lo aprendido en tuning jobs anteriores para guiar la nueva búsqueda.

Al terminar, `best_training_job()` devuelve el mejor training job, y `HyperparameterTuningJobAnalytics` permite analizar los resultados de todos.

### 5.4 Evaluación de modelos fundacionales en Amazon Bedrock

| Tipo de evaluación | Qué mide |
|---|---|
| Automática | Métricas predefinidas (**exactitud, robustez, toxicidad**) con datasets propios o integrados y curados |
| Humana | Métricas subjetivas o personalizadas (**amabilidad, estilo, alineación con la voz de la marca**), con tu equipo de evaluadores o con evaluaciones gestionadas por AWS |
| LLM-as-a-judge | Un LLM califica las salidas a partir de prompts propios: **correctness, completeness, harmfulness** |
| Programática | Métricas tradicionales de lenguaje natural: **BERTScore, F1, exact match** |
| Knowledge bases (RAG) | Recuperación y generación: **relevancia del contexto, cobertura del contexto, faithfulness, corrección y completitud** |

> **Para recordar en el examen**
> - Local (una máquina, subconjunto, depuración) vs. remoto (training job con el dataset completo) vs. distribuido (varios nodos para más datos o menos tiempo). Los notebooks escalan verticalmente, no horizontalmente: para producción, scripts versionados en Git.
> - SageMaker apaga las instancias al terminar el job; `max_run` lo corta pase lo que pase (por defecto, 86 400 s).
> - CloudWatch = métricas del job en tiempo real; `DescribeTrainingJob` = métricas finales; EventBridge = reaccionar a jobs fallidos (notificar, reiniciar, rollback; DLQ).
> - Debugger = tensores intermedios (vanishing gradients) + profiling de CPU, GPU, memoria y red con reglas integradas; `smdebug` para reglas personalizadas.
> - AMT: la búsqueda bayesiana aprovecha los resultados previos y necesita menos evaluaciones que grid o random; se configuran rangos, métrica objetivo (maximizar o minimizar), `max_jobs` y `max_parallel_jobs`; el warm start reutiliza tuning jobs anteriores.
> - Bedrock: métricas subjetivas (estilo, voz de marca) → evaluación humana; correctness, completeness y harmfulness → LLM-as-a-judge; BERTScore, F1 y exact match → programática; RAG → evaluación de knowledge bases.

---

## 6. Despliegue e infraestructura de inferencia

*Dominio 3 · Tareas 3.1 y 3.2*

Desplegar es integrar el modelo y sus recursos en producción para que genere inferencias en tiempo real, casi en tiempo real o por lotes (con un retraso que depende del tamaño del lote y del payload). Se consumen servicios de IA por API o se expone un modelo propio; en ambos casos rigen los seis pilares del Well-Architected Framework.

### 6.1 Gestionado vs. no gestionado, e infraestructura

| | Gestionado (SageMaker AI) | No gestionado (EC2, ECS, EKS, Lambda) |
|---|---|---|
| Quién gestiona la infraestructura | SageMaker: aprovisionamiento, escalado, balanceo de carga, alta disponibilidad y tolerancia a fallos | Tú, aunque uses servicios serverless como Lambda o Fargate |
| Ventaja principal | Poca carga operativa | Control de hardware, dependencias, red e integraciones; ajuste fino de costos (tipos de instancia, Spot, políticas de escalado propias) |
| Cuándo | Producción con alta disponibilidad y escalado sin esfuerzo | Configuraciones complejas, redes propias, integración con sistemas existentes o controles propios por residencia y cumplimiento (GDPR, HIPAA) |

La infraestructura se elige por escalabilidad, facilidad de gestión, necesidad de personalización, seguridad y costo.

**Inferencia vs. entrenamiento:** la inferencia solo hace el *forward pass* (sin *backward pass* ni actualización de pesos): necesita menos cómputo y memoria, se ejecuta todo el tiempo, en cualquier lugar (edge o nube) e integrada en la aplicación, y prioriza baja latencia y eficiencia. El entrenamiento requiere alto paralelismo, lotes grandes, más cómputo y memoria; se ejecuta en la nube y solo cuando hace falta.

**Tipo de instancia:** CPU para ML tradicional, preprocesamiento, feature engineering e inferencia ligera; GPU para deep learning e imagen (acelera el entrenamiento, pero no mejora la exactitud). La inferencia de un modelo profundo suele bastar con una GPU menor o incluso una CPU: desplegarlo en una GPU grande la infrautiliza y encarece.

| Familia | Uso |
|---|---|
| t2 | Desarrollo, pruebas e inferencia básica (CPU barata con ráfagas) |
| c5 / m5 | Optimizada para cómputo / uso general; inferencia básica y cargas mixtas |
| **Inf1 / Inf2** | Chips **AWS Inferentia**: inferencia de deep learning e IA generativa de alto rendimiento al **menor costo**; el **AWS Neuron SDK** se integra con PyTorch y TensorFlow |
| g4dn (T4), g5 (A10G), g6 (L4) | GPU NVIDIA para inferencia de deep learning |
| p4 (A100), p5 (H100) | GPU de gama alta; el costo más alto |
| **Trn1 / Trn2** | Chips **AWS Trainium**: entrenamiento (e inferencia) de LLM e IA generativa de alto rendimiento |

### 6.2 Opciones de despliegue gestionado en SageMaker AI

Todas se basan en **contenedores** (imagen en **ECR**) y en el **artefacto del modelo** `model.tar.gz` en S3 (parámetros aprendidos y archivos de inferencia), que SageMaker descarga y descomprime en las instancias. Se integran con IAM (acceso) y CloudWatch (métricas y logs).

| Opción | Latencia | Payload y tráfico | Infraestructura | Casos |
|---|---|---|---|---|
| **Tiempo real** | Baja, inmediata | Pequeño; tráfico continuo e interactivo | Instancias siempre activas, autoescalables | Fraude, recomendaciones, apps interactivas |
| **Serverless** | Baja, con posibles **cold starts** | Pequeño; tráfico **intermitente o impredecible** | Sin elegir instancias; escala según la demanda, incluso a cero | Tráfico variable con periodos de inactividad |
| **Asíncrona** | Diferida (casi tiempo real) | **Grande**, procesamiento **largo** | Cola interna; entrada y salida en S3; notificación SNS opcional | Imágenes o documentos grandes, modelos complejos |
| **Batch transform** | Irrelevante | Datasets completos, offline | **Sin endpoint**: un trabajo que procesa en paralelo y termina | Scoring periódico de clientes, análisis offline, mantenimiento predictivo |

**Tiempo real.** Se registra el **modelo** (artefacto + imagen), se crea la **configuración del endpoint** (tipo y número de instancias y modelo) y, a partir de ella, el **endpoint HTTPS** con una URL única; `deploy()` del SDK crea las tres piezas. El autoescalado con Application Auto Scaling es opcional y el balanceo entre instancias, automático. CloudWatch muestra invocaciones, latencia y tasa de errores. Los clientes invocan con `InvokeEndpoint` (HTTPS POST).

- **Endpoint multimodelo (MME):** muchos modelos que comparten **el mismo contenedor** (misma imagen, framework y runtime) en un solo endpoint. Los modelos viven en S3 y cada petición indica cuál usar con **`TargetModel`**; SageMaker carga el modelo en memoria la primera vez que se invoca y descarga los menos usados cuando falta memoria. Ideal para **muchos modelos de uso poco frecuente**, sin pagar un endpoint por modelo. Límites: una sola imagen de contenedor, contención de recursos con mucha demanda y latencia de carga (cold start) al invocar un modelo que no está en memoria.
- **Endpoint multicontenedor (MCE):** hasta **15 contenedores con frameworks distintos** en las mismas instancias, definidos con la API **`CreateModel`** (varias imágenes en un modelo). Invocación **en serie** (los contenedores forman un pipeline) o **directa** (cada petición elige un contenedor con **`TargetContainerHostname`**). Casos: frameworks distintos, mismo framework con algoritmos distintos, pruebas A/B entre versiones de un framework, o preprocesamiento con dependencias diferentes. También sufre contención de recursos.

**MME vs. MCE:** mismo contenedor para muchos modelos poco usados (ahorro) frente a contenedores distintos (frameworks o dependencias diferentes) que comparten instancias y escalan juntos.

**Serverless.** Configuras el **tamaño de memoria** y la **concurrencia máxima** (peticiones simultáneas que el endpoint procesa; en el SDK, `ServerlessInferenceConfig` con `memory_size_in_mb` y `max_concurrency`). Pagas el cómputo usado para procesar peticiones (según la memoria) y los datos procesados, no instancias. La **concurrencia aprovisionada** mantiene entornos listos y evita los cold starts de la primera petición tras un periodo inactivo, pero esos entornos se pagan aunque no reciban tráfico. Ahorra costo con tráfico intermitente, pero no mejora la latencia.

**Asíncrona.** Se deja el payload en S3 y se llama a `InvokeEndpointAsync` indicando su ubicación; SageMaker encola la petición, la procesa y escribe el resultado en S3, y opcionalmente notifica el éxito o el error por **Amazon SNS**. **Serverless vs. asíncrona:** tráfico intermitente con payloads pequeños frente a payloads grandes con procesamiento largo que tolera respuestas diferidas.

**Batch transform.** Se indican la ubicación de entrada, el modelo y la ubicación de salida; SageMaker procesa en paralelo y los recursos solo existen mientras dura el trabajo. Cada archivo de entrada produce `<archivo>.out` en la salida. Preparar un transformer no despliega ningún endpoint.

### 6.3 Despliegue no gestionado

- **EC2**: control total del sistema operativo, la red y la seguridad; eliges el tipo de instancia (CPU o GPU).
- **ECS**: contenedores descritos en *task definitions* (imágenes, recursos, red). Tipo de lanzamiento **EC2** (control granular de las instancias) o **Fargate** (serverless). Se integra con CloudWatch, IAM y EFS, y ofrece *service discovery* para microservicios.
- **EKS**: Kubernetes gestionado (AWS opera el plano de control) con add-ons, autoescalado, rolling updates y self-healing. Es la opción natural si la organización ya tiene infraestructura o experiencia en Kubernetes, con una experiencia coherente on-premises y en la nube.
- **Lambda**: serverless; escala automáticamente con las peticiones y se paga solo el tiempo de cómputo; la disparan API Gateway, S3 o DynamoDB; **Lambda Layers** para dependencias; cada invocación es stateless. Con S3 (modelo) y API Gateway (API REST) forma un despliegue escalable y barato para cargas muy variables.

### 6.4 Amazon SageMaker Neo (edge)

Neo **compila** un modelo (TensorFlow, TensorFlow Lite, PyTorch, ONNX) para un hardware y software concretos (procesadores ARM, Intel, NVIDIA…): optimiza su grafo de cómputo y genera un binario específico que **reduce la latencia y el consumo de energía**. No reentrena ni poda el modelo (la poda y la cuantización se aplican aparte, antes de compilar). Tampoco distribuye ni actualiza modelos en flotas de dispositivos (OTA): eso se hace con **AWS IoT Greengrass** (🕒 SageMaker Edge Manager, que cubría esa función, se retiró en 2024). El compilation job necesita que `S3Uri` apunte a **un único archivo `.tar.gz`** (un SavedModel de TensorFlow, que es un directorio, debe empaquetarse), la forma de la entrada (`DataInputConfig`) y el dispositivo de destino (`TargetDevice`, p. ej., `coreml`).

### 6.5 Autoescalado de endpoints

**Application Auto Scaling** escala horizontalmente las instancias de una **variante de producción**, identificada por `ResourceId` = `endpoint/<nombre>/variant/<variante>`. `AllTraffic` es solo el nombre por defecto que el SDK da a la variante cuando se despliega un único modelo, y el escalado se configura por variante. Hacen falta dos piezas:
- **Objetivo escalable**: mínimo y máximo de instancias (dimensión `sagemaker:variant:DesiredInstanceCount`).
- **Política de escalado**:
  - **Target tracking**: mantiene una métrica en un valor objetivo, p. ej., `SageMakerVariantInvocationsPerInstance` (predefinida; invocaciones medias por instancia y minuto) o `CPUUtilization` (se indica como métrica personalizada del namespace `/aws/sagemaker/Endpoints`). Otras métricas: `ModelLatency` y `GPUUtilization`.
  - **Step scaling**: cuando una alarma supera su umbral, ajusta las instancias según escalones predefinidos.
  - **Scheduled scaling**: ajusta según un calendario.

`ScaleInCooldown` y `ScaleOutCooldown` definen un periodo tras cada acción de escalado en el que no se lanza otra del mismo tipo, para no escalar en exceso por fluctuaciones puntuales. El autoescalado es compatible con MME y MCE; en un MCE, escalar por invocaciones solo funciona si los contenedores tienen un uso de CPU y una latencia similares.

### 6.6 Estrategias de despliegue y prueba

- **Variantes de producción** en un endpoint multivariante: el tráfico se reparte por **pesos** o se invoca una variante concreta con **`TargetVariant`**. Es la base de las **pruebas A/B**.
- **Shadow testing** (validación automatizada de SageMaker): compara en tiempo real el desempeño de un modelo nuevo con el de producción usando las mismas peticiones reales.
- **Blue/Green** (*deployment guardrails*): se llama a `UpdateEndpoint` con una **nueva configuración de endpoint** y un `DeploymentConfig`. La **flota azul** tiene la versión actual y la **verde**, la nueva. Durante el **periodo de baking**, que empieza cuando la verde recibe tráfico real, **alarmas de CloudWatch** vigilan sus métricas; si salta alguna, el **auto-rollback** devuelve el 100 % del tráfico a la flota azul. Sin alarmas no hay vigilancia ni rollback automático. `TerminationWaitInSeconds` es la espera antes de terminar la flota azul tras el último baking y `MaximumExecutionTimeoutInSeconds`, la duración máxima del despliegue.

| Modo de tráfico | Pasos | Parámetros | Riesgo y velocidad |
|---|---|---|---|
| **All At Once** | 1 (100 % a la verde) | — | El más rápido y el más arriesgado; exige mucha confianza en la versión nueva |
| **Canary** | 2 (canario y luego el resto) | `CanarySize` (% de capacidad o número de instancias; **≤ 50 %** de la flota verde) + `WaitIntervalInSeconds` (baking del canario) | Riesgo bajo: un problema solo afecta a una pequeña parte de los usuarios |
| **Linear** | N incrementos iguales, cada uno con su baking | `LinearStepSize` (**10–50 %** de la flota verde) + `WaitIntervalInSeconds` | El riesgo más bajo y el más lento; actualizaciones críticas |

> **Para recordar en el examen**
> - Tiempo real (baja latencia, payload pequeño, instancias siempre activas) vs. serverless (tráfico intermitente, escala a cero, cold starts; memoria + concurrencia máxima) vs. asíncrona (payload grande o proceso largo; cola, S3 y SNS) vs. batch transform (datasets completos, sin endpoint).
> - MME: muchos modelos poco usados con el mismo contenedor (`TargetModel`). MCE: hasta 15 contenedores de frameworks distintos, invocados en serie o directamente (`TargetContainerHostname`).
> - La concurrencia aprovisionada elimina los cold starts de serverless, pero se paga aunque no haya tráfico.
> - Inferencia de deep learning al menor costo → Inferentia (Inf1/Inf2, Neuron SDK); entrenamiento de LLM → Trainium; no sobredimensionar la GPU de inferencia.
> - Neo compila para hardware edge (menos latencia y energía); no reentrena, no poda ni distribuye modelos (flotas → IoT Greengrass).
> - Autoescalado por variante con Application Auto Scaling: target tracking (`SageMakerVariantInvocationsPerInstance` predefinida; CPU como métrica personalizada), step o scheduled; los cooldowns evitan oscilaciones.
> - Blue/Green: All At Once (rápido y arriesgado) → Canary (canario ≤ 50 % de la flota verde) → Linear (pasos del 10–50 %, el más seguro); todos necesitan alarmas de CloudWatch para el auto-rollback. A/B = variantes con pesos o `TargetVariant`.
> - Gestionado (SageMaker: menos operación) vs. no gestionado (EC2, ECS, EKS, Lambda: más control); Kubernetes existente → EKS; contenedores sin gestionar servidores → ECS con Fargate; tráfico muy variable pagando por uso → Lambda.

---

## 7. Orquestación, CI/CD e infraestructura como código

*Dominio 3 · Tarea 3.3*

**MLOps** aplica DevOps al ML (integración, entrega y despliegue continuos) y la **orquestación** automatiza el ciclo de vida completo, que a mano es lento y propenso a errores.

### 7.1 Amazon SageMaker Pipelines y Model Registry

SageMaker Pipelines es la herramienta central de orquestación del examen: define, programa y monitoriza flujos de ML **coherentes y repetibles**. Su infraestructura es **serverless** y autoescalable, ofrece varias interfaces (editor visual de Studio con arrastrar y soltar, SDK, API o JSON), se integra con todas las funciones de SageMaker y **no tiene costo propio**: solo pagas los jobs que orquesta.

- Los pasos se combinan en un `Pipeline` que forma un **DAG**: el orden lo determinan las **dependencias** entre pasos, no el orden de la lista, y los pasos sin dependencias entre sí se ejecutan en paralelo. `upsert()`, con el rol IAM de ejecución, crea o actualiza el pipeline; `start()` lo lanza. Puede ejecutarse bajo demanda, de forma programada o en respuesta a eventos.
- Tipos de paso: `ProcessingStep` (job de SageMaker Processing: preprocesamiento, features o evaluación), `TransformStep` (batch transform con un modelo existente), `TrainingStep` y `ModelStep` (crea el modelo en SageMaker o lo **registra en Model Registry**). **`ModelStep` no crea un endpoint**: el despliegue se resuelve aparte, por ejemplo con un paso que invoque una función Lambda o con un pipeline de CI/CD que se dispare al aprobar el modelo en Model Registry.
- Si un paso falla, la ejecución se detiene y queda marcada como fallida, pero **no notifica sola**: hay que configurar una regla de EventBridge sobre el cambio de estado de la ejecución que publique en SNS. El monitoreo y los logs del pipeline son responsabilidad del ingeniero de ML.
- Disparadores: una regla de **EventBridge** (patrón de eventos o calendario) cuyo destino es el **ARN del pipeline**, con un **rol IAM** que permita a EventBridge iniciarlo. Si el evento es un `PutObject` de S3 capturado vía CloudTrail, hace falta un trail que registre los eventos de datos de S3 de ese bucket.

**SageMaker Model Registry** cataloga los modelos durante todo su ciclo de vida: controla las **versiones**, centraliza metadatos, métricas de rendimiento e historial, y asigna a cada versión un **estado de aprobación** (pendiente, aprobado o rechazado). **Aprobar una versión puede disparar su despliegue automático** en un pipeline de CI/CD, y el registro auditable de cambios apoya la gobernanza y el cumplimiento. Se integra con Pipelines para registrar y versionar modelos reproducibles.

### 7.2 Otras herramientas de orquestación

| Herramienta | Tipo | Ideal para |
|---|---|---|
| **SageMaker Pipelines** | Específica de ML | Flujos de ML de extremo a extremo con mínima intervención manual e integración nativa con SageMaker |
| **AWS Step Functions** | General y serverless | **Máquinas de estados** visuales que coordinan servicios de AWS: orden garantizado, **ramificación condicional, gestión de errores** y flujos paralelos a gran escala, con mínima carga operativa |
| **Amazon MWAA** | Apache Airflow gestionado | Flujos complejos y personalizables definidos por programa (ETL y pipelines de datos con S3, Redshift y EMR); rico ecosistema de plugins; equipos que ya usan Airflow |

**EventBridge vs. orquestadores:** EventBridge enruta eventos y dispara acciones (complementa a Pipelines y Step Functions), pero la orquestación con estado de varios pasos es de ellos.

### 7.3 CI/CD e infraestructura como código

| Servicio | Qué hace | Qué no hace |
|---|---|---|
| **CodeArtifact** | Repositorio administrado de **paquetes y dependencias**: paquetes privados y conexión con PyPI, npm y Maven Central; se usa con pip, npm, Maven, Gradle o NuGet | No compila ni despliega |
| **CodeBuild** | **Compila, prueba y empaqueta** en entornos temporales según `buildspec.yml`; produce paquetes, binarios o imágenes de contenedor; puede obtener dependencias de CodeArtifact | No despliega |
| **CodeDeploy** | **Despliega un artefacto ya construido** en EC2, servidores on-premises, Lambda o ECS, con estrategias in-place, blue/green o desplazamiento gradual del tráfico, y rollback ante fallos | No construye el código |
| **CodePipeline** | **Orquesta** las etapas de entrega (Source → Build → Test → aprobación manual → Deploy) invocando CodeBuild, CodeDeploy u otras herramientas | Es el director de orquesta, no ejecuta la compilación ni el despliegue |

La **infraestructura como código (IaC)** define la infraestructura en archivos declarativos, así que es repetible, coherente y versionable como el código de la aplicación (también permite reconstruirla tras un fallo):
- **CloudFormation**: plantillas **JSON o YAML** que aprovisionan recursos en varias regiones y cuentas.
- **Terraform**: herramienta de HashiCorp con su propio lenguaje, **HCL**, y un flujo de CLI común para cientos de servicios en la nube.
- **AWS CDK**: define la infraestructura con lenguajes de programación (**JavaScript, TypeScript, Python, Java, C# y Go**) y genera plantillas de CloudFormation.

> **Para recordar en el examen**
> - SageMaker Pipelines: orquestador específico de ML, serverless y sin costo propio; DAG definido por dependencias; pasos de Processing, Training, Transform y Model. `ModelStep` crea o registra el modelo, pero no crea el endpoint.
> - Un fallo detiene la ejecución sin avisar: EventBridge (cambio de estado) → SNS. Para lanzar el pipeline por evento o calendario: regla de EventBridge con el ARN del pipeline y un rol IAM.
> - Model Registry: versiones, métricas y estado de aprobación; la aprobación dispara el despliegue en CI/CD; registro auditable para gobernanza.
> - Pipelines (ML de extremo a extremo) vs. Step Functions (máquinas de estados con ramificación condicional y gestión de errores entre servicios de AWS) vs. MWAA (Airflow gestionado para pipelines de datos complejos).
> - EventBridge dispara y enruta, pero no reemplaza a un orquestador con estado.
> - CodeArtifact guarda paquetes, CodeBuild compila y prueba, CodeDeploy despliega artefactos ya construidos y CodePipeline orquesta las etapas.
> - CloudFormation (plantillas JSON/YAML) vs. Terraform (HashiCorp, lenguaje HCL) vs. CDK (lenguajes de programación que generan CloudFormation).

---

## 8. Monitoreo de modelos en producción

*Dominio 4 · Tarea 4.1*

En producción la distribución de los datos cambia (comportamiento de los usuarios, estacionalidad, factores externos) y los patrones aprendidos dejan de valer: el desempeño cae de forma gradual (**model drift**). La respuesta es monitorear continuamente y reentrenar cuando haga falta, instrumentando métricas (exactitud, latencia, uso de recursos, drift), alertas y dashboards, y sabiendo por qué importa cada una (la exactitud revela la degradación; el uso de recursos permite optimizar costos).

### 8.1 Tipos de drift

| Tipo | Qué cambia | Cómo se detecta |
|---|---|---|
| Data drift | Las propiedades estadísticas de los datos de entrada | Comparando los datos capturados con el baseline de datos |
| Model quality drift | Las métricas de desempeño (accuracy, precision, recall…) | Comparando las predicciones con las **etiquetas reales** |
| Bias drift | La equidad de las predicciones entre grupos | Clarify |
| Feature attribution drift | La importancia relativa de las features frente al baseline de entrenamiento | Clarify |
| Concept drift | La relación entre entradas y target, aunque las entradas mantengan su distribución | Solo con el monitoreo de calidad del modelo (requiere ground truth) |

### 8.2 Amazon SageMaker Model Monitor 🕒

Model Monitor usa *monitoring schedules* para capturar los datos del endpoint, analizarlos frente a un baseline y alertar mediante alarmas de CloudWatch cuando detecta cambios significativos. Monitorea de forma continua endpoints en tiempo real y batch transforms que se ejecutan periódicamente, y de forma programada los batch transforms asíncronos. Tiene capacidades preconstruidas sin código o admite análisis personalizados, y ofrece cuatro tipos de monitoreo: **data quality**, **model quality**, **bias drift** y **feature attribution drift**. Los dos últimos se apoyan en **SageMaker Clarify**, que además explica cuánto pesó cada feature en una predicción (p. ej., puntaje crediticio, ingresos e historial laboral en un modelo de préstamos).

**Requisito previo: data capture.** En un endpoint en tiempo real se capturan las peticiones y las predicciones; en batch transform, las entradas y salidas del trabajo.

| Paso | Qué ocurre |
|---|---|
| 1. Baseline | Model Monitor calcula estadísticas y **restricciones** (*constraints*) de referencia y las guarda en S3 |
| 2. Data quality | Según el schedule (p. ej., cada hora o cada día), compara los datos capturados con el baseline: media, mediana, desviación estándar, tasa de faltantes |
| 3. Model quality | Compara las predicciones con etiquetas reales que tú obtienes periódicamente para una parte de los datos capturados y subes a S3 (su ubicación es un parámetro del job) |
| 4. CloudWatch | Recibe las métricas de los pasos 2 y 3; sus alarmas notifican por **SNS** (email o SMS) |
| 5. Reentrenamiento | Una regla de **EventBridge** reacciona al cambio de estado de la alarma e inicia el pipeline de SageMaker (o una Lambda o un flujo de Step Functions) |

- **Baseline de calidad de datos**: se calcula sobre el **dataset de entrenamiento** (estadísticas por feature), así que en un pipeline puede calcularse **antes** del paso de entrenamiento. **Baseline de calidad del modelo**: se calcula sobre un **dataset de validación** con las predicciones del modelo y sus etiquetas, y fija restricciones sobre las métricas (p. ej., recall ≥ 0.8, precision ≥ 0.9); por eso va **después** del entrenamiento. Ambos se recalculan en cada reentrenamiento.
- Lo que queda fuera de las restricciones es una **violación**: se reporta en S3 como JSON (analizable con **Athena** y **QuickSight**) y como métrica en CloudWatch.
- El job de calidad de datos corre en su propio cómputo con **Deequ** (open source, sobre Apache Spark): no ralentiza la inferencia y escala con el volumen. La frecuencia del schedule controla su costo y se ajusta a la velocidad con que se espera que cambien los datos.
- **Baseline de monitoreo vs. modelo baseline:** las estadísticas y restricciones de referencia del monitoreo frente a un modelo simple que, al inicio del desarrollo, sirve para comparar modelos más complejos.
- En la arquitectura de referencia, los datos se preparan con Data Wrangler, las features se guardan en Feature Store y las tareas de detección de drift pueden integrarse en SageMaker Pipelines.

**Logs vs. eventos vs. alarmas:** los logs registran lo que ocurre (troubleshooting y análisis); los eventos son cambios de estado de los recursos sobre los que se quiere actuar (una falla, un cambio de configuración); las alarmas se disparan al cruzar un umbral y, además de avisar, pueden iniciar acciones directamente o vía EventBridge (actualizar datos o hiperparámetros, recalcular el baseline, reentrenar).

### 8.3 ML Well-Architected Lens en la fase de monitoreo

La Lens aplica al ML los seis pilares del Well-Architected Framework. Para el examen importan el nombre del principio y su pilar, no el código (que cambió entre versiones de la Lens).

| Pilar | Principio | Implementación |
|---|---|---|
| Excelencia operacional | Enable model observability and tracking | Model Monitor; CloudWatch; **Model Dashboard** (portal central para ver, buscar y explorar modelos); Clarify; **ML Lineage Tracking** (linaje de experimentos y artefactos); **Model Cards** (requisitos de negocio, decisiones clave y observaciones del modelo); shadow testing |
| | Synchronize architecture and configuration, and check for skew across environments | CloudFormation (IaC); Model Monitor |
| Seguridad | Restrict access to intended legitimate consumers | Mínimo privilegio en el endpoint, tratado como cualquier API HTTPS: restricción por rangos de IP, control de bots, peticiones HTTPS firmadas, TLS |
| | Monitor human interactions with data for anomalous activity | Registro de acceso a datos; **Macie** (descubre y clasifica datos sensibles, como PII, en S3); GuardDuty |
| Confiabilidad | Allow automatic scaling of the model endpoint | Auto scaling de endpoints (target tracking, step o scheduled) |
| | Ensure a recoverable endpoint with a managed version control strategy | SageMaker Pipelines y **SageMaker Projects**; CloudFormation u otra IaC para reconstruir la infraestructura; **ECR** para versionar las imágenes de contenedor |
| Eficiencia de desempeño | Evaluate model explainability | Clarify |
| | Evaluate data drift | Model Monitor; Clarify |
| | Monitor, detect, and handle model performance degradation | Model Monitor; Auto Scaling resuelve la degradación operativa (latencia, throughput), no la calidad de las predicciones |
| | Establish an automated re-training framework | Model Monitor → alarma de CloudWatch → EventBridge → Pipelines o Step Functions |
| | Review for updated data/features for retraining | Data Wrangler, a intervalos fijados según la volatilidad de los datos |
| | Include human-in-the-loop monitoring | **Amazon A2I** 🕒: revisión humana de predicciones de baja confianza o de muestras aleatorias |
| Optimización de costos | Monitor usage and cost by ML activity | **Etiquetas** (activadas como *cost allocation tags*) por actividad, p. ej., reentrenamiento o hosting; AWS Budgets |
| | Monitor Return on Investment for ML models | KPIs de negocio definidos en la fase de definición del problema; reportes en **QuickSight**. Si el ROI es negativo, reducir el costo (p. ej., serverless con tráfico intermitente) |
| | Monitor endpoint usage and right-size the instance fleet | CloudWatch; Auto Scaling; **Frugal Architecture** |
| Sostenibilidad | Measure material efficiency | Recursos aprovisionados por unidad de trabajo como KPI, un baseline de eficiencia y la estimación de las mejoras |
| | Retrain only when necessary | KPIs acordados con el negocio (exactitud mínima, error máximo); Model Monitor dispara el reentrenamiento solo si el drift supera un umbral; Pipelines o Step Functions |

**Frugal Architecture:** el costo y la sostenibilidad son requisitos no funcionales críticos, al mismo nivel que la seguridad o el desempeño. Ejemplo: desplegar FSx for Lustre en la **misma AZ** que SageMaker para evitar costos de transferencia entre AZ.

> **Para recordar en el examen**
> - Data drift (estadísticas de entrada) vs. model quality drift (métricas frente a etiquetas reales) vs. bias drift vs. feature attribution drift. El concept drift solo aparece en el monitoreo de calidad del modelo, porque requiere ground truth.
> - Model Monitor necesita data capture y un baseline: el de datos sale del training set (puede calcularse antes de entrenar); el del modelo, de un set de validación con predicciones y etiquetas (después de entrenar). Ambos se recalculan al reentrenar.
> - Bias drift y feature attribution drift se apoyan en Clarify.
> - Flujo: violaciones → S3 (JSON; Athena, QuickSight); métricas → CloudWatch → alarma → SNS o EventBridge → reentrenamiento con Pipelines, Step Functions o Lambda.
> - El job de data quality corre en cómputo propio (Deequ sobre Spark), sin afectar la inferencia; la frecuencia del schedule controla su costo.
> - Auto Scaling corrige la degradación operativa (latencia, throughput), no la calidad del modelo.
> - Revisión humana → A2I; datos sensibles en S3 → Macie; linaje → ML Lineage Tracking; documentación del modelo → Model Cards; ROI → KPIs + QuickSight; costo por actividad → cost allocation tags.
> - Sostenibilidad: reentrenar solo cuando el drift supere un umbral, no por calendario.

---

## 9. Monitoreo de infraestructura y optimización de costos

*Dominio 4 · Tarea 4.2*

Además del modelo hay que vigilar la infraestructura que lo sirve (uso de CPU y memoria, E/S de disco, throughput de red, salud de las instancias) y su costo.

### 9.1 Observabilidad y seguridad operativa

| Servicio | Para qué | Detalles clave |
|---|---|---|
| **CloudWatch** | Métricas, logs y alarmas de los recursos | Visibilidad en tiempo real de la salud del sistema; alarmas que notifican o disparan acciones |
| **CloudWatch Logs Insights** | Buscar y analizar logs de CloudWatch Logs (causa raíz) | Lenguajes Logs Insights QL, OpenSearch PPL y OpenSearch SQL; descubre automáticamente los campos de logs de Route 53, Lambda, CloudTrail y VPC; *field indexes* (menos costo, consultas más rápidas); resultados cifrables con KMS; detección de patrones; consultas guardadas en dashboards. Logs de training jobs, endpoints y batch transform |
| **EventBridge** | Bus de eventos serverless: enruta eventos hacia destinos y reacciona | Base de las arquitecturas event-driven; dispara Lambda, Step Functions o pipelines (nuevos datos, drift, fin de un training job); envía eventos a CloudWatch; recibe hallazgos de Security Hub y GuardDuty |
| **CloudTrail** | Auditoría de las llamadas a la API: **quién hizo qué y cuándo** | Registra la creación, actualización y eliminación de training jobs, endpoints e instancias; cumplimiento y gobernanza; alarmas vía CloudWatch |
| **X-Ray** | Trazado de solicitudes entre componentes distribuidos, con mapa de servicios | SageMaker **no** tiene integración nativa: se traza la aplicación que invoca el endpoint (API Gateway, Lambda), donde la llamada aparece como segmento downstream con su latencia; para ver dentro del contenedor hay que instrumentar el código |
| **GuardDuty** | **Detección de amenazas** | Analiza eventos de CloudTrail, VPC Flow Logs y logs DNS con inteligencia de amenazas y ML. Genera hallazgos, pero **no remedia**: la respuesta se automatiza con EventBridge + Lambda y se centraliza en Security Hub. Exfiltración desde notebooks o almacenamiento, llamadas desde IPs maliciosas, credenciales comprometidas |
| **Inspector** | **Gestión de vulnerabilidades** | Escanea continuamente instancias EC2 e **imágenes de ECR** (p. ej., imágenes propias de entrenamiento o inferencia) en busca de **CVE** y exposición de red no deseada; remediación con Lambda + Systems Manager |
| **Security Hub** | Gestión de la postura de seguridad (**CSPM**) | Agrega y prioriza los hallazgos de GuardDuty, Inspector, AWS Config, IAM Access Analyzer y terceros, y evalúa la configuración frente a buenas prácticas; no detecta amenazas por sí mismo |

**CloudWatch vs. EventBridge vs. CloudTrail:** monitorear métricas y logs frente a enrutar eventos y reaccionar frente a auditar llamadas a la API. **GuardDuty vs. Inspector:** «¿alguien ataca o usa indebidamente mis recursos?» (p. ej., un acceso desde una IP maliciosa) frente a «¿mis recursos tienen debilidades explotables?» (p. ej., una CVE en una imagen de ECR). Juntos cubren amenazas en tiempo real y vulnerabilidades; evaluar el cumplimiento de buenas prácticas le corresponde a Security Hub.

### 9.2 Seguimiento y optimización de costos

| Servicio | Para qué |
|---|---|
| **Cost Explorer** | Visualizar y analizar el gasto y el uso a lo largo del tiempo por servicio, tipo de uso, periodo o etiqueta; **pronósticos** basados en el historial; detectar sobreaprovisionamiento y oportunidades (Spot, Savings Plans, Reserved Instances); se integra con Budgets |
| **Cost and Usage Reports (CUR)** | Los datos de facturación **más granulares** (por recurso y hasta por hora) como archivos crudos en un bucket de S3, actualizados al menos una vez al día (hoy CUR 2.0 desde AWS Data Exports); análisis a medida con Athena, QuickSight o herramientas de terceros |
| **Budgets** | Presupuestos personalizados de costo y uso con alertas (email o SNS) cuando el gasto **real o pronosticado** supera un umbral; por proyecto, equipo o etapa filtrando por *cost allocation tags* |
| **Trusted Advisor** | Recomendaciones de buenas prácticas en seis categorías (optimización de costos, desempeño, seguridad, tolerancia a fallos, límites de servicio, excelencia operacional): recursos inactivos o infrautilizados, buckets S3 abiertos, cuenta raíz sin MFA. Con el plan de soporte Basic solo muestra los checks de límites de servicio y algunos de seguridad; el conjunto completo requiere un plan superior |

**Cost Explorer vs. CUR:** interfaz visual con datos agregados y pronósticos, para la gestión de alto nivel, frente a datos crudos de máximo detalle para análisis a medida.

El **etiquetado** (pares clave-valor por propósito, responsable, entorno o fase de ML) es una capacidad clave de FinOps: permite asignar costos a actividades concretas, como el reentrenamiento y el hosting, y las etiquetas deben activarse como *cost allocation tags* para aparecer en los reportes de facturación.

### 9.3 Modelos de precios

| Modelo | Cómo funciona | Casos en ML |
|---|---|---|
| On-Demand | Pagas el tiempo de cómputo usado, sin compromiso | Cargas impredecibles, análisis exploratorio, entrenamiento ad hoc |
| Reserved Instances | Compromiso de capacidad durante 1 o 3 años, con descuento | Cargas estables de largo plazo |
| Savings Plans | Compromiso de gasto por hora ($/hora) durante 1 o 3 años; más flexibles que las RI | Uso predecible; entrenamiento e inferencia continuos |
| Spot Instances | Capacidad EC2 sobrante con hasta **90 %** de descuento; AWS puede interrumpirla con **2 minutos** de aviso. Ya no se puja: se paga el precio Spot vigente, con un precio máximo opcional | Tareas no críticas y tolerantes a interrupciones: procesamiento batch, preprocesamiento; en SageMaker, sobre todo el entrenamiento con **Managed Spot Training** |
| Dedicated Hosts | Servidor físico dedicado a tu uso | Requisitos regulatorios de aislamiento físico |

Hay tres tipos de Savings Plans: **Compute** (EC2, Lambda y Fargate), **EC2 Instance** y **SageMaker**. ⚠️ Las Reserved Instances, los Dedicated Hosts y los Savings Plans Compute o EC2 Instance **no cubren las instancias de ML de SageMaker**.

**SageMaker Savings Plans:** se aplican solo al uso de instancias de SageMaker AI. Comprometes un gasto constante ($/hora) durante 1 o 3 años y ahorras **hasta un 64 %** frente a On-Demand. El descuento se aplica automáticamente a Studio notebooks, notebooks On-Demand, Processing, Data Wrangler, Training, Real-Time Inference y Batch Transform, y se conserva aunque cambies de **familia, tamaño o región** de instancia (p. ej., de `ml.c5.xlarge` en Ohio a `ml.inf1.2xlarge` en Oregón).

> **Para recordar en el examen**
> - CloudWatch (métricas, logs, alarmas) vs. EventBridge (enrutar eventos y reaccionar) vs. CloudTrail (quién llamó a qué API y cuándo) vs. X-Ray (trazas de solicitudes; SageMaker sin integración nativa).
> - GuardDuty (amenazas; hallazgos sin remediación → EventBridge + Lambda) vs. Inspector (CVE y exposición de red en EC2 e imágenes de ECR) vs. Security Hub (CSPM: agrega y prioriza hallazgos y evalúa buenas prácticas; no detecta).
> - Analizar los logs de training jobs, endpoints o batch transform → CloudWatch Logs Insights.
> - Cost Explorer (visual, agregado, pronósticos) vs. CUR (datos crudos más granulares en S3) vs. Budgets (alertas por gasto real o pronosticado) vs. Trusted Advisor (recomendaciones; el set completo requiere un plan de soporte superior).
> - Costos por actividad de ML → etiquetas activadas como cost allocation tags.
> - Instancias de ML de SageMaker: solo las cubren los SageMaker Savings Plans (hasta 64 %; flexibles en familia, tamaño y región); las RI, los Dedicated Hosts y los Compute o EC2 Instance Savings Plans no.
> - Spot: hasta 90 % de descuento con aviso de 2 minutos → tareas tolerantes a interrupciones (Managed Spot Training); carga estable → Savings Plans o RI; aislamiento físico por regulación → Dedicated Hosts.

---

## 10. Seguridad de soluciones de ML

*Dominio 4 · Tarea 4.3*

### 10.1 Principios de diseño: seguridad por diseño

La seguridad se incorpora desde el diseño y en todas las fases del ciclo de vida (**security by design**). Toda solución debería poder relacionarse con alguno de estos principios:

| Principio | Idea | Servicios |
|---|---|---|
| Base sólida de identidad | **Mínimo privilegio** (solo los permisos necesarios; acota el daño si se filtran credenciales) y **separación de funciones** (ninguna identidad controla todas las operaciones críticas) | IAM |
| Seguridad en todas las capas | **Defensa en profundidad**: si una capa falla, las demás siguen protegiendo | De fuera hacia dentro: CloudFront (CDN, protección DDoS) y **AWS WAF** (exploits web) en el borde → Application Load Balancer → NACL (subred) → security group (recurso); además GuardDuty, KMS e Inspector |
| Trazabilidad | **No repudio**: cada acción se rastrea hasta su origen | CloudTrail (API; con logs en S3 para conservarlos y hacer forense), **AWS Config** (cambios de configuración y cumplimiento), GuardDuty, CloudWatch, EventBridge |
| Proteger los datos | Tríada **CIA** (confidencialidad, integridad, disponibilidad) en reposo, en uso y en tránsito | Ver §10.4 |
| Automatizar la seguridad | **Policy-as-Code**: políticas versionadas, probadas y desplegadas como código, aplicadas en cada etapa de MLOps (p. ej., con CodePipeline y CodeBuild) | Config e IAM; alertas de GuardDuty y Security Hub con remediación automática vía Lambda, CloudWatch y EventBridge |
| Prepararse para incidentes | Plan de respuesta e investigación | **Amazon Detective**: reúne logs (p. ej., de CloudTrail) y hallazgos de GuardDuty y los relaciona con ML, análisis estadístico y teoría de grafos |

Orden sistemático para proteger una carga: identidades → infraestructura → datos → flujos de ML → cumplimiento.

### 10.2 IAM: identidades

IAM controla la **autenticación** (quién inició sesión) y la **autorización** (qué permisos tiene). Un **principal** es una persona o aplicación autenticada como usuario o rol IAM. Cuenta AWS ≠ identidad: la cuenta contiene los recursos; la identidad (usuario, grupo o rol) recibe permisos sobre ellos.

| Identidad | Credenciales | Claves |
|---|---|---|
| **Usuario IAM** | De largo plazo (contraseña, access keys) | Pertenece a una sola cuenta, que factura su actividad. Buena práctica: los humanos acceden por **federación** con credenciales temporales, no como usuarios IAM |
| **Rol IAM** | **Temporales**, emitidas por **AWS STS** al asumirlo y válidas durante la sesión | Lo asumen usuarios de la misma u otra cuenta, otros roles (*role chaining*), **service principals** (servicios de AWS) y usuarios federados, si su **trust policy** lo permite. El rol tiene ARN `arn:aws:iam::<cuenta>:role/<rol>`; la sesión, `arn:aws:sts::<cuenta>:assumed-role/<rol>/<sesión>` |
| **Grupo IAM** | — | Reúne usuarios que heredan sus políticas (p. ej., `ML-Trainers` con `sagemaker:CreateTrainingJob`); **no puede ser principal** |

**IAM Identity Center** gestiona de forma centralizada las identidades de la fuerza laboral (sus propios usuarios y grupos) y asigna el acceso con **permission sets**, que crean automáticamente los roles necesarios. Con **federación**, las identidades vienen de proveedores externos (Microsoft Entra ID, Okta, Cisco Duo) y asumen roles.

**Amazon SageMaker Role Manager** 🕒 crea roles para ML a partir de *personas* con **actividades de ML** preseleccionadas, cada una con permisos predefinidos que se pueden añadir o quitar. Ayuda a aplicar el mínimo privilegio según la función.

| Persona | Para | Actividades preseleccionadas |
|---|---|---|
| **Data Scientist** | Desarrollo y experimentación | Studio Classic, gestionar ML jobs y modelos, tablas de Glue (Feature Store, Data Wrangler), Canvas (AI Services, MLOps, Kendra), MLflow, EMR Serverless |
| **MLOps** | Operación | Studio Classic, gestionar modelos y Pipelines, buscar y visualizar experimentos, S3 Full Access |
| **SageMaker AI Compute** | Roles de ejecución de jobs y endpoints | Access Required AWS Services (S3, ECR, CloudWatch, EC2) |

### 10.3 Políticas de acceso

Una política es un documento JSON con `Version` (`2012-10-17`), `Statement`, `Effect` (`Allow` o `Deny`), `Action`, `Resource`, `Principal` (en las políticas basadas en recursos) y `Condition` (opcional). Evaluación: lo que ninguna política permite se **deniega implícitamente**, y un **`Deny` explícito siempre prevalece** sobre cualquier `Allow`.

| Tipo | Se adjunta a | Qué hace |
|---|---|---|
| Basada en identidad | Usuario, grupo o rol | Define qué acciones puede realizar la identidad, sobre qué recursos y en qué condiciones |
| Basada en recursos | Un recurso (p. ej., una bucket policy de S3 con `Principal`) | Define quién puede acceder al recurso y con qué acciones; no todos los recursos la admiten |
| **Permissions boundary** | Usuario o rol (no grupos) | Fija el máximo que pueden conceder las políticas de identidad: solo se permite lo que permitan ambos. **No concede permisos** (con un límite de DynamoDB, S3 y CloudWatch, la identidad nunca operará otro servicio, ni siquiera IAM) |
| **SCP** | Organización, OU o cuenta (AWS Organizations) | Fija el máximo para todos los usuarios y roles de las cuentas miembro, **incluido el usuario raíz**; es un *guardrail* que **no concede permisos** (p. ej., denegar `ec2:RunInstances` fuera de una región en una OU) |

**AWS Organizations** gobierna de forma centralizada varias cuentas (facturación unificada, SCP, aprovisionamiento automatizado). **AWS Control Tower** lo amplía con un entorno multicuenta ya configurado, seguro y conforme (Account Factory, seguridad centralizada, buenas prácticas de gobierno). Imponer la residencia de los datos de entrenamiento en un entorno multicuenta → Control Tower + SCP.

### 10.4 Red, datos, auditoría y cumplimiento

Una **VPC** es una sección aislada lógicamente de la nube de AWS (rangos de IP, subredes, tablas de rutas, gateways) donde se lanzan notebooks, training jobs y endpoints de SageMaker separados de otras redes.

| | NACL | Security group |
|---|---|---|
| Nivel | Subred | Instancia o recurso (EC2, RDS, load balancers, notebooks de SageMaker) |
| Estado | **Sin estado**: hay que definir reglas de entrada y de salida | **Con estado**: el tráfico de respuesta se permite automáticamente |
| Reglas | Allow y Deny (IP, protocolo, puerto) | Solo Allow; como origen admite otros security groups o prefix lists (útil con IPs dinámicas, como las de un ELB) |
| Asociación | Una NACL por subred (una NACL puede cubrir varias subredes) | Varios por instancia, con reglas combinadas |

Los **VPC endpoints** conectan la VPC con servicios de AWS sin pasar por internet, así que el tráfico no sale de la red de AWS:
- **Interface endpoint**: usa **AWS PrivateLink** y crea una ENI dentro de la VPC; sirve para muchos servicios, como SageMaker AI, S3 y EC2.
- **Gateway endpoint**: añade entradas a las tablas de rutas; solo existe para **S3 y DynamoDB**.
- PrivateLink también conecta de forma privada servicios propios entre VPC y cuentas.

| Estado de los datos | Servicios |
|---|---|
| En reposo | **AWS KMS** (crea y gestiona las claves de cifrado; integración nativa con S3, EBS, RDS, DynamoDB y muchos más); **Secrets Manager** (credenciales de bases de datos y API keys cifradas con KMS) |
| En uso | **AWS Nitro Enclaves** (entornos de cómputo aislados y protegidos por hardware para datos muy sensibles); IAM |
| En tránsito | SSL/TLS; **AWS Certificate Manager (ACM)** aprovisiona y renueva certificados para CloudFront, Elastic Load Balancing y API Gateway |

Cifrado en reposo de las fuentes de SageMaker: **S3** cifra cada objeto nuevo automáticamente (SSE-S3); **FSx for Lustre** siempre cifra en reposo; en **EFS** el cifrado se elige al crear el file system (la consola lo activa por defecto, pero con CLI, API o SDK hay que habilitarlo) y después no se puede cambiar.

**Auditoría:** CloudTrail registra la actividad de la API, **no el tráfico de red**; el tráfico IP de la VPC se registra con **VPC Flow Logs** (publicables en CloudWatch Logs o S3). CloudWatch aporta métricas, logs y alarmas.

**Cumplimiento:** SageMaker AI está en el alcance de HIPAA (*HIPAA eligible*), PCI DSS, ISO 27001, SOC 1/2/3, FedRAMP y GDPR, entre otros. Estar en el alcance no hace que tu carga cumpla automáticamente: el cumplimiento es **responsabilidad compartida**, y GDPR es un reglamento, no una certificación. **AWS Config** monitorea continuamente la configuración para evaluar el cumplimiento de políticas internas y normas (hay un conformance pack de buenas prácticas de seguridad para SageMaker); **Security Hub** centraliza la postura de seguridad; **AWS Artifact** permite descargar los reportes de auditores y certificaciones de terceros (ISO, PCI, SOC).

> **Para recordar en el examen**
> - Usuario IAM (credenciales de largo plazo) vs. rol (credenciales temporales de STS; lo asumen usuarios, roles, servicios o identidades federadas según su trust policy) vs. grupo (agrupa permisos; nunca es principal). Humanos → federación o IAM Identity Center.
> - Denegación implícita por defecto y `Deny` explícito siempre gana. Permissions boundary (usuario o rol) y SCP (cuentas y OU, incluido root) solo limitan: no conceden permisos.
> - Guardrails o residencia de datos en un entorno multicuenta → Organizations o Control Tower con SCP.
> - NACL (subred, sin estado, Allow y Deny, reglas de entrada y salida) vs. security group (recurso, con estado, solo Allow).
> - Tráfico privado hacia servicios de AWS → VPC endpoints: gateway (solo S3 y DynamoDB, vía tablas de rutas) vs. interface (PrivateLink y ENI; p. ej., SageMaker AI).
> - Cifrado: KMS (claves en reposo), Secrets Manager (secretos), ACM y TLS (en tránsito), Nitro Enclaves (en uso). S3 y FSx for Lustre cifran por defecto; en EFS se decide al crearlo y no se cambia.
> - Auditoría: CloudTrail (llamadas a la API) vs. VPC Flow Logs (tráfico IP) vs. Config (configuración y cumplimiento); investigación de incidentes → Detective; reportes de auditores → Artifact.
> - Estar «en el alcance» de HIPAA, PCI DSS o ISO no equivale a cumplir (responsabilidad compartida). Incorporar la seguridad desde el diseño = security by design; varias capas de controles = defensa en profundidad.
