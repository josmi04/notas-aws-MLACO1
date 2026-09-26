# 02 · Ingesta y almacenamiento — chunks de repaso

Dominio 1.1 (Ingest and store data). Marcas: **⚠️** = corrección al libro verificada en docs AWS (sept. 2026) · **🕒** = cambio de estado del servicio posterior al libro · **[C01]** = contenido que la guía MLA-C02 ya no nombra explícitamente.

---

## 1 · Criterios de selección (ingesta y almacenamiento)

**Qué recordar**
- Ingesta = llevar datos crudos de las fuentes a AWS (batch o tiempo real). Almacenamiento = persistirlos en un data store de AWS hasta su procesamiento.
- Streaming → Data Firehose, Kinesis Data Streams, MSK, Managed Service for Apache Flink. Batch/migración/ETL → DataSync, Glue.

**Criterios de ingesta**: escalabilidad (velocidad + volumen) · resiliencia (reanudar desde el punto de fallo) · seguridad/compliance (en tránsito y en reposo; PCI DSS, HIPAA, residencia de datos) · costo (el streaming no se detiene y el costo crece rápido) · flexibilidad (adaptarse a cambios).

**Criterios de almacenamiento**

| Criterio | Pregunta que responde |
|---|---|
| Durabilidad | ¿Cuánto tiempo debo conservar los datos? |
| Disponibilidad | ¿Qué tan pronto necesito usarlos? (% de tiempo operativo) |
| Tipo de almacenamiento | ¿Objeto, archivo o bloque para prepararlos? |
| Costo | ¿Cuánto pagar por almacenarlos? (pilar Well-Architected) |
| Seguridad | ¿Qué protección necesitan en reposo? (según sensibilidad) |

**Trampa**: durabilidad (*cuánto tiempo*) ≠ disponibilidad (*cuándo/qué tan pronto*).

---

## 2 · Formatos de datos

| Formato | Orientación | Claves |
|---|---|---|
| CSV | Filas | Datos estructurados tabulares |
| JSON | Documento | Datos semiestructurados |
| Parquet / ORC | **Columnar** | Compresión y codificación por columna; leer columnas sin leer filas completas → menos almacenamiento, consultas más rápidas |
| Avro | **Filas** | Basado en esquema; el esquema del escritor viaja con los datos (autodescriptivo) → serialización rápida y compacta, sin overhead por valor; resuelve diferencias de esquema lector/escritor |
| RecordIO | Registros | Usado sobre todo por Apache MXNet; cada registro lleva su longitud en bytes antepuesta |

**Reglas**
- Primer criterio de elección → **que lo acepte el algoritmo** que vas a entrenar.
- Consultas analíticas sobre pocas columnas → Parquet/ORC.
- Evolución de esquema / serialización por filas → Avro.

**Trampa**: Avro es por **filas**, no columnar (Parquet y ORC sí lo son).

---

## 3 · Formatos de entrenamiento de los algoritmos integrados de SageMaker

| Formatos aceptados | Algoritmos |
|---|---|
| RecordIO-protobuf **o** CSV | K-means, k-NN, Linear Learner, LDA, NTM, PCA, Random Cut Forest |
| **Solo** RecordIO-protobuf | Factorization Machines, Seq2Seq |
| Solo CSV | IP Insights |
| JSON Lines o Parquet | DeepAR |
| CSV, LibSVM o Parquet | XGBoost |
| Texto (una oración por línea, tokens separados por espacio) | BlazingText |
| RecordIO (MXNet) **o** .jpg/.png | Image Classification, Object Detection |
| Solo archivos de imagen | Semantic Segmentation |

⚠️ **Correcciones a la tabla 2.1 del libro**: Factorization Machines **no** acepta CSV para entrenar; Seq2Seq **no** acepta texto plano, sino solo RecordIO-protobuf con tokens enteros; Semantic Segmentation **no** acepta RecordIO.

**Trampas**
- "RecordIO" (imágenes, MXNet) ≠ "RecordIO-protobuf" (tensores/tabulares).
- DeepAR es el único de la lista con JSON Lines; XGBoost, el único con LibSVM.

---

## 4 · Patrón de acceso a datos

**Qué recordar**: define cómo productores y consumidores consultan, almacenan y recuperan datos. Junto con el algoritmo, determina el **formato** y el **servicio**.

| Factor | Qué determina |
|---|---|
| Tamaño (volumen) | Particionamiento efectivo |
| Forma | Organizar los datos según las consultas → velocidad y escalabilidad |
| Velocidad (picos de consulta) | Particionamiento para capacidad de I/O |

**Campos de un patrón documentado**: nombre, descripción, prioridad, operación (lectura/escritura), tipo (un ítem / varios / todos), filtro, orden.

---

## 5 · Servicios según el tipo de dato

| Estructurado | Semiestructurado | No estructurado |
|---|---|---|
| RDS, Aurora, Redshift, S3, Athena | DynamoDB, DocumentDB, Athena, S3 | S3, Rekognition, Transcribe, Comprehend |

**Reglas**
- S3 es el **único** que aparece en las tres categorías, por lo que es el más flexible.
- S3 + Athena → consultar con SQL directamente en S3, sin reformatear y sin infraestructura.

**Trampa**: Rekognition, Transcribe y Comprehend **procesan** datos no estructurados; no son almacenamiento.

---

## 6 · Tipos de almacenamiento: objeto vs archivo vs bloque

| Tipo | Servicio | Modelo | Ideal para |
|---|---|---|---|
| Objeto | S3 | Dato + metadatos + ID único (URI); acceso por API | Cloud-native, data lakes, gran escala a bajo costo |
| Archivo | EFS, familia FSx | Jerárquico; compartido | Directorios compartidos, medios, CMS |
| Bloque | EBS | Bloques de tamaño fijo con dirección única, **sin metadatos** | Transaccional, baja latencia |

**Regla clave**: fuentes de datos de entrenamiento con integración nativa en SageMaker = **S3, EFS y FSx for Lustre**. EBS **no** es una de ellas.

---

## 7 · Amazon Data Firehose (antes Kinesis Data Firehose)

**Qué recordar**: servicio totalmente administrado que captura, transforma y **entrega** streams a destinos en **near real time (segundos)**. Escala automáticamente y no requiere código consumidor.

**Configuración**: fuente → transformaciones opcionales → destino.
- **Fuentes**: Direct PUT (API), Kinesis Data Streams, MSK; más de 20 integraciones (CloudWatch Logs, logs de WAF y de Network Firewall, SNS, AWS IoT).
- **Destinos**: S3, Redshift, OpenSearch, Splunk, Snowflake, endpoint HTTP personalizado.
- **Opcionales**: conversión a Parquet/ORC, descompresión, transformación con Lambda, particionamiento dinámico por atributos del registro.

**Reglas**
- ⚠️ Conversión de formato: la entrada debe ser **JSON** → Parquet/ORC, con el esquema tomado de una tabla del **Glue Data Catalog**. Si llega CSV → Lambda (CSV→JSON) y luego la conversión nativa.
- "Cargar streaming en S3/Redshift/OpenSearch con mínima configuración" → Firehose.

**Trampas**
- Firehose es near real time (segundos), no tiempo real (<1 s: ver KDS).
- Firehose entrega datos; no es una plataforma para aplicaciones de procesamiento personalizado.

**Casos de uso**: streaming a data lakes/warehouses con conversión a Parquet, observabilidad de seguridad (SIEM), enriquecer streams con modelos de ML en tránsito.

---

## 8 · Amazon Kinesis Data Streams (KDS)

**Qué recordar**: ingesta **y almacena** streams para procesamiento personalizado en **tiempo real**; el retardo *put-to-get* suele ser **< 1 s**. Es elástico y escala para no perder registros antes de que expiren.

**Consumidores**: Managed Service for Apache Flink, Apache Spark, aplicaciones en EC2, Lambda (near real time sin servidores).

**Reglas**
- Procesamiento personalizado en tiempo real o varias aplicaciones consumiendo el mismo stream en paralelo → KDS.
- Los productores escriben directamente al stream, sin agrupar en el servidor; si el servidor de aplicaciones falla, no se pierden datos.

**Trampa**: KDS necesita consumidores (código o servicio); Firehose no.

**Casos de uso**: ingesta acelerada de logs y feeds, métricas y reportes en tiempo real, analítica en tiempo real (clickstream), procesamiento complejo (combinar streams en nuevos streams).

---

## 9 · Amazon MSK (Managed Streaming for Apache Kafka)

**Qué recordar**: Apache Kafka open source totalmente administrado y de alta disponibilidad. Las apps, herramientas y plugins de Kafka existentes funcionan **sin cambios de código**.

**Componentes**

| Componente | Función |
|---|---|
| Brokers | Ingestan, almacenan (particiones de topic) y procesan datos. Mínimo **1 broker por AZ**; cada AZ tiene su propia subnet |
| Controladores | Brokers elegidos: estado de particiones y réplicas, reasignación, relación leader-follower |
| ZooKeeper | Coordinación distribuida entre brokers (creado por MSK) |
| KRaft | Reemplaza a ZooKeeper; los metadatos viven en controladores de Kafka dentro del clúster. **Sin costo adicional ni gestión** |

**Plano de control vs plano de datos**
- Control (consola/CLI/SDK de AWS) → crear o borrar clústeres, listarlos, ver propiedades, cambiar número o tipo de brokers.
- Datos (APIs de Kafka) → crear topics, producir y consumir.

**Reglas**
- On-demand y cero operación → **MSK Serverless**: aprovisiona y escala capacidad, gestiona particiones y cobra solo por uso.
- Ya usan Kafka / system of record de streaming / arquitecturas event-driven → MSK.

**Trampa**: MSK sirve para **gestionar y distribuir** streams (agregación de logs, event sourcing); el procesamiento o análisis en tiempo real es tarea de Flink.

---

## 10 · Amazon Managed Service for Apache Flink (antes Kinesis Data Analytics)

**Qué recordar**: procesamiento y análisis **stateful** de streams en tiempo real, con garantía **exactly-once** y tolerancia a fallos. Serverless: se paga por uso.

**Reglas**
- **No tiene almacenamiento propio**: el estado y los datos van en S3, MSK, KDS u otros.
- Soporta consultas continuas sobre datos **no acotados** (streams) y consultas batch sobre datos **acotados**.
- **Flink Studio** → consultas interactivas sobre streams y lanzamiento de apps stateful en pocos pasos.
- Consultas SQL o analítica en tiempo real sobre un stream → Flink.

**Casos de uso**: ETL en streaming, pipelines de ML, feature engineering, medición y facturación, monitoreo de red, apps event-driven (detección de fraude, geofencing, monitoreo de procesos).

🕒 **Kinesis Data Analytics for SQL** (incluida la función SQL `RANDOM_CUT_FOREST` para anomalías): no admite nuevas apps desde el 15-10-2025; las apps se eliminan desde el 27-01-2026. Si aparece en una pregunta, se refiere a ese servicio legado. No confundir con el algoritmo Random Cut Forest de SageMaker.

---

## 11 · Elegir el servicio de streaming

| Necesidad | Servicio |
|---|---|
| Cargar un stream en S3/Redshift/OpenSearch/Splunk con mínima gestión y conversión de formato | Data Firehose |
| Procesamiento personalizado en tiempo real (<1 s) y varios consumidores | Kinesis Data Streams |
| Kafka existente, event sourcing, system of record de streaming | MSK (Serverless si no se quiere dimensionar) |
| Analítica o transformación stateful en tiempo real, SQL sobre streams, exactly-once | Managed Service for Apache Flink |
| Procesamiento ligero de cada registro sin servidores | Lambda como complemento de cualquiera de los anteriores |

---

## 12 · AWS DataSync

**Qué recordar**: transferencia **online** de archivos u objetos hacia, desde y entre servicios de almacenamiento de AWS, otras nubes y on-premises. Usa un protocolo propio con arquitectura paralela y multihilo, y cobra una **tarifa plana por GB**.

**Orígenes externos**: NFS, SMB, HDFS, almacenamiento de objetos (Google Cloud Storage, Azure Blob, Wasabi, compatibles con la API de S3). También NFS autogestionado en tu VPC.

**Destinos AWS**: S3 (cualquier clase, incluidas Glacier Flexible Retrieval y Deep Archive), EFS, FSx (Windows, Lustre, OpenZFS, NetApp ONTAP), Snowcone y Snowball Edge.

**Seguridad**: cifrado + **validación de integridad** de extremo a extremo; acceso vía roles IAM; **VPC endpoints** para no atravesar Internet público.

**Reglas**
- Migrar, replicar o archivar datos activos o fríos por red → DataSync (sin scripts propios ni herramientas comerciales).
- Archivar datos fríos on-premises → directamente a Glacier Flexible Retrieval o Deep Archive.
- File system en standby → EFS o FSx.

🕒 **DataSync Discovery** (análisis de almacenamiento on-premises con agente y recomendaciones de migración) dejó de existir el 20-05-2025. **Snowcone** está discontinuado (11-2024) y la familia **Snow** está cerrada a clientes nuevos desde el 07-11-2025.

**Trampa**: DataSync **mueve** datos; Glue los **transforma** (ETL).

---

## 13 · AWS Glue

**Qué recordar**: ETL **serverless** con auto scaling.
- **Crawlers** → infieren esquema y tipos de archivo, y registran los metadatos en el **Data Catalog**.
- **Generación automática** de scripts ETL.
- Conectores integrados (Redshift, Aurora, SQL Server, MySQL, MongoDB, MariaDB, PostgreSQL) + **JDBC personalizado**; más de 70 fuentes.
- Frameworks: Spark, PySpark, Scala, **Ray** (Glue for Ray).
- Cargas de trabajo: batch, micro-batch y **streaming**.
- **Interactive sessions** → explorar y preparar datos desde el IDE o notebook que prefieras.

**Reglas**
- ETL con demanda de cómputo irregular, volumen impredecible y muchas fuentes → Glue (auto scaling).
- Descubrir y catalogar datos en AWS, on-premises u otras nubes para consultarlos → crawlers + Data Catalog.

**Trampa**: Glue = preparar e integrar (ETL); DataSync = migrar o transferir.

---

## 14 · Amazon S3

**Qué recordar**: almacenamiento de objetos con escala prácticamente ilimitada. Replica los objetos en varios dispositivos y AZs, lo que le da durabilidad y resiliencia. Es la opción más costo-eficiente y con integración nativa en SageMaker, tanto para **datos de entrenamiento** como para **artefactos del modelo**.

**Reglas**
- Datos crudos centralizados y de alta disponibilidad (data lake) → S3.
- Backup/restore que cumpla RTO, RPO y compliance → replicación y protección de datos de S3.
- Datos fríos retenidos por compliance → clases Glacier.
- Entrenamiento de LLMs a gran escala → S3.
- Costos → políticas de **lifecycle** para mover objetos entre clases según el patrón de acceso.

**Integraciones**: SageMaker, Lambda, ECS, EKS, Athena.

---

## 15 · Clases de almacenamiento de S3

| Clase | Clave para el examen |
|---|---|
| Standard | Acceso frecuente |
| Intelligent-Tiering | Mueve objetos automáticamente según el acceso; para patrones desconocidos o cambiantes |
| Express One Zone | Una sola AZ; latencia consistente de **milisegundos de un dígito**; apps sensibles a la latencia |
| Standard-IA | Acceso infrecuente pero rápido; más barata que Standard; multi-AZ |
| One Zone-IA | Acceso infrecuente, **sin resiliencia multi-AZ**, más barata |
| Glacier Instant Retrieval | Datos de larga vida, raramente accedidos, con acceso **inmediato** |
| Glacier Flexible Retrieval | Archivo; recuperación de minutos a horas |
| Glacier Deep Archive | La más barata; archivo a largo plazo |
| S3 on Outposts | S3 on-premises; residencia de datos |

⚠️ **Intelligent-Tiering** (el libro dice "dos niveles"): tiene 3 niveles automáticos: Frequent → Infrequent Access (tras **30 días** sin acceso) → Archive Instant Access (tras **90 días**). Además ofrece 2 niveles opcionales asíncronos: Archive Access (≥90 días) y Deep Archive Access (≥180 días). No cobra por recuperación.

⚠️ **Deep Archive** (el libro dice "hasta 12 h"): recuperación Standard ≤ **12 h**; Bulk ≤ **48 h**.

**Reglas**
- "Recuperable en ≤12 h y retención de años al menor costo" → Deep Archive (recuperación Standard).
- "Poco acceso pero inmediato" → Glacier Instant Retrieval, no Flexible Retrieval.
- "Recreable y sin necesidad de multi-AZ" → One Zone-IA.

---

## 16 · Amazon Athena

**Qué recordar**: consultas SQL estándar **serverless e interactivas** directamente sobre S3 (CSV, JSON, Parquet, ORC), sin cargar datos ni gestionar infraestructura.

**Regla**: "almacenamiento barato + consultas SQL" → S3 + Athena, no Redshift ni DynamoDB.

**Trampa**: Athena aparece como servicio para datos estructurados **y** semiestructurados.

---

## 17 · Amazon EFS

**Qué recordar**: file system **serverless y elástico** (hasta petabytes y GB/s), con **NFSv4.0/4.1**. Se monta en EC2, EKS, ECS y Lambda.
- **Lifecycle management**: mueve los datos fríos a las clases Infrequent Access y Archive.
- Protección con **AWS Backup** y **replicación de EFS**.
- Semántica de file system: **consistencia fuerte** y **bloqueo de archivos**.

**Reglas**
- Datos compartidos entre muchos consumidores, en volumen impredecible → EFS.
- Datos de entrenamiento que **ya están** en EFS → usarlos directamente en SageMaker **[C01]**.

**Trampas**
- Usar EFS con SageMaker requiere trabajo de red: el job va en una **VPC** (el libro menciona un endpoint de interfaz), con security groups e IAM configurados. S3 no lo requiere.
- En costo, EFS > S3. Elige EFS por la semántica de file system compartido, no por precio.

**Casos de uso**: ciencia de datos y ML, compartir código y archivos en DevOps y contenedores, CMS.

---

## 18 · Amazon FSx for Lustre

**Qué recordar**: file system paralelo de alto rendimiento: latencia **sub-ms**, cientos de GB/s y millones de IOPS. Casos: ML, HPC, video, modelado financiero. Integración nativa con SageMaker **[C01]**.

**Reglas**
- Actúa como capa sobre un bucket de S3: las instancias de entrenamiento no descargan los datos de S3 (no saben que vienen de S3). Resultado: **arranque y entrenamiento más rápidos**.
- Carga de datos desde S3:
  - **Carga única** → dataset grande disponible pronto, con mejor rendimiento y **mayor costo inicial**.
  - **Lazy loading** → carga al acceder, con **menor costo inicial**; el primer acceso a cada dato es más lento.
- Despliegue:
  - **Scratch** → efímero, procesamiento de corto plazo.
  - **Persistent** → largo plazo, cargas orientadas a throughput.

**Regla de examen**: "alto throughput + baja latencia + datos en S3 + entrenamiento en SageMaker" → FSx for Lustre.

---

## 19 · S3 vs EFS vs FSx for Lustre como fuente de entrenamiento en SageMaker [C01]

| Dimensión | S3 | EFS | FSx for Lustre |
|---|---|---|---|
| Tiempo de carga y entrenamiento | El más lento | Intermedio | **El más rápido** |
| Costo | **El más barato** | Intermedio (equilibrio costo/rendimiento) | El más caro |
| Setup | Mínimo | VPC, security groups, IAM | VPC; se vincula a S3 |
| Cuándo elegirlo | Gran escala, largo plazo, costo | Datos ya en EFS, file system compartido con consistencia fuerte | Entrenamiento intensivo y latencia sub-ms |

**Regla**: solo tiempo de entrenamiento → FSx for Lustre; costo crítico → S3; equilibrio o datos ya en EFS → EFS.

---

## 20 · Elegir variante de FSx (no Lustre)

| Servicio | Protocolos / base | Diferenciador | Caso típico |
|---|---|---|---|
| FSx for NetApp ONTAP | NFS, SMB, iSCSI, NVMe-over-TCP (SAN + NAS) | **Compresión y deduplicación**; gestión unificada de flash, disco y nube | Migrar NetApp u otros servidores NFS/SMB/iSCSI **sin cambiar código**; DR entre regiones; bases de datos de alto rendimiento (sub-ms, millones de IOPS) |
| FSx for Windows File Server | **SMB** + **Active Directory** | Deduplicación, SSD | Migrar file servers Windows; **SQL Server en alta disponibilidad sin licencia Enterprise**; perfiles de WorkSpaces y AppStream 2.0 |
| FSx for OpenZFS | NFS v3, v4, v4.1, v4.2 (clientes Linux, Windows, macOS) | Más de 1 M de IOPS, sub-ms | Migrar ZFS o file servers Linux sin cambios; apps con uso intensivo de datos |

**Trampas**
- Active Directory o SMB nativo de Windows → FSx for Windows, no ONTAP.
- SAN (iSCSI) + NAS en el mismo servicio → ONTAP.

---

## 21 · Amazon EBS

**Qué recordar**: almacenamiento en **bloque** que funciona como SAN en la nube. Los volúmenes se adjuntan a instancias EC2.

**Reglas**
- **SSD** → transaccional, I/O pequeño y frecuente; métrica clave: **IOPS**.
- **HDD** → streaming grande; métrica clave: **throughput**.
- **Snapshots** → backups point-in-time a partir de los cuales se restauran **volúmenes nuevos**.

**Casos de uso**: migrar SAN on-premises, bases de datos (SAP HANA, Oracle, SQL Server, PostgreSQL, MySQL, Cassandra, MongoDB), clústeres de Hadoop/Spark (desadjuntar y readjuntar volúmenes).

**Trampa**: EBS no es fuente nativa de datos de entrenamiento en SageMaker (esas son S3, EFS y FSx for Lustre).

---

## 22 · Amazon RDS

**Qué recordar**: base de datos relacional administrada con **8 motores**: Aurora PostgreSQL, Aurora MySQL, PostgreSQL, MySQL, MariaDB, SQL Server, Oracle y Db2. AWS se encarga de aprovisionar, parchear, respaldar, recuperar, detectar fallos y repararlos.

**Despliegues**: nube (Aurora o RDS) · híbrido (**RDS on Outposts**) · acceso privilegiado al SO y la BD (**RDS Custom**).

**Reglas**
- Migrar una BD legada sin refactorizar (solo cambia la conexión; los stored procedures y herramientas se mantienen) → RDS con el mismo motor.
- Ahorro de licencias + pago por uso → migrar a RDS.

---

## 23 · Amazon DynamoDB

**Qué recordar**: NoSQL **serverless** con latencia de milisegundos de un dígito a cualquier escala y mantenimiento sin downtime. Modelos **clave-valor y documento**.

**Reglas**
- Admite **lecturas fuertemente consistentes** y **transacciones ACID** en una o varias tablas con una sola solicitud → finanzas y pedidos.
- **No hay JOINs**: se omiten por ineficientes a escala.
- Evitar **Scan**; usar Query.

**Casos de uso**: financieros (ACID; escala en horas de mercado), gaming (estado, jugadores, sesiones, leaderboards), media (índice de metadatos, estadísticas deportivas near real time, watchlists, eventos para recomendaciones).

**Trampa**: DynamoDB se clasifica como servicio de datos **semiestructurados**.

---

## 24 · Troubleshooting de capacidad y escalado

| Problema / objetivo | Acción |
|---|---|
| Detectar anomalías (CPU, memoria, I/O) | **CloudWatch**: métricas + alarmas |
| Causa raíz: quién cambió qué | **CloudTrail**: registro de llamadas a la API |
| Demanda variable | Auto Scaling (EC2 y otros recursos) |
| Más tráfico de lectura en la BD | **Read replicas**; auto scaling de Aurora |
| Pipelines lentos | Particionamiento, caché, procesamiento paralelo |
| Streaming de alto volumen | MSK / Flink / KDS / Firehose + Lambda (escala con el volumen) |
| Costo de almacenamiento en S3 | Políticas de **lifecycle** entre clases |
| Rendimiento de EBS | Monitorear y cambiar tipo o tamaño de volumen |
| BD lenta | Optimizar queries e índices; **sharding**; en DynamoDB evitar Scan |
| Costos | **Cost Explorer**; Reserved Instances o Savings Plans (carga predecible); **Spot** (cargas elásticas y efímeras) |

**Trampa**: CloudWatch = métricas y alarmas; CloudTrail = auditoría de llamadas a la API.

---

## 25 · Escenarios tipo del capítulo → regla

| Escenario | Regla |
|---|---|
| Datos crudos de IoT, repositorio centralizado y de alta disponibilidad | S3 |
| Procesados con acceso inmediato 6 meses; crudos recuperables en ≤12 h durante 6 años; SQL; costo mínimo | S3 (lifecycle → Deep Archive) + Athena |
| SQL en tiempo real sobre un stream de Firehose con registros GZIP | Managed Service for Apache Flink + Lambda para preprocesar |
| Anomalías en transacciones en streaming hacia un data lake en S3, mínimo overhead | Firehose + `RANDOM_CUT_FOREST` de KDA SQL (🕒 discontinuado) |
| Geolocalización near real time en Redshift, costo-eficiente | KDS + **Redshift Streaming Ingestion** |
| CSV near real time → Parquet en S3, lo más eficiente | Firehose + Lambda (CSV→JSON) + conversión nativa a Parquet |
| Entrenamiento a gran escala de imágenes, alto throughput y baja latencia, datos en S3 | FSx for Lustre |
| Mucha data de sensores, acceso frecuente, costo-eficiente, para SageMaker | S3 |

⚠️ **Redshift Streaming Ingestion** solo acepta como fuente **KDS o MSK** (y Kafka externo), no Firehose. Tampoco descomprime registros: hay que enviarlos descomprimidos. **Redshift Spectrum** consulta datos en S3; no es ingesta de streaming.

---

*Fuentes de las correcciones ⚠️/🕒: docs AWS — parámetros de algoritmos integrados de SageMaker, conversión de formato de Firehose, streaming ingestion de Redshift, clases de S3 y Glacier, discontinuación de KDA for SQL, ciclo de vida de DataSync y Snow Family.*
