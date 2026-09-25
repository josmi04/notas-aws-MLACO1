---
tema: "Capítulo 2 — Ingesta y almacenamiento de datos (chunks de repaso)"
fuente: Guia Oficial/02_data_ingestion_and_storage.md
guia-mla-c01: [Dominio 1, 1.1 Ingest and store data]
verificado: 2026-09-23 (cifras heredadas de la versión explicada)
tags: [aws, mla-c01, repaso, ingesta, almacenamiento, formatos-de-datos, firehose, kinesis, msk, flink, datasync, glue, s3, athena, efs, fsx, ebs, rds, dynamodb]
---

> [!warning] Estado de servicios (verificado 2026-09-23)
> - SageMaker → **SageMaker AI**. AppStream 2.0 → **WorkSpaces Applications**.
> - **Kinesis Data Analytics for SQL**: retirado el **2026-01-27**, y con él la función SQL `RANDOM_CUT_FOREST`. Reemplazo: Managed Service for Apache Flink (Flink SQL). En preguntas del libro que lo mencionan, responde con la lógica del libro.
> - **DataSync Discovery**: fin de soporte **2025-05-20**. El resto de DataSync sigue vigente.
> - **Snowcone**: descontinuado (nov-2024). **Snowball Edge**: solo clientes existentes desde **2025-11-07**. Clientes nuevos: DataSync o AWS Data Transfer Terminal.
> - **Glue for Ray**: sin clientes nuevos desde **2026-04-30** (alternativa: Ray en EKS).
> - **MXNet**: retirado (Apache Attic, 2024). RecordIO-protobuf sigue siendo válido en los algoritmos integrados.
> - **Amazon Glacier** (el servicio original, con bóvedas): sin clientes nuevos desde 2025-11-07. **No afecta** a las clases de almacenamiento **S3 Glacier**.
> - **MLA-C02** (beta desde 2026-09-29): la configuración de EFS/FSx como origen de datos de entrenamiento salió del temario. Esas partes van marcadas **[C01]**.

---

## 1 · Tipos de almacenamiento: bloques, archivos y objetos

**Qué recordar**

| | Bloques | Archivos | Objetos |
|---|---|---|---|
| Servicio AWS | EBS | EFS, FSx (Lustre, ONTAP, Windows, OpenZFS) | S3 |
| Acceso | Volumen conectado a una instancia, que lo formatea con su propio sistema de archivos | Sistema de archivos compartido por red (NFS, SMB, cliente Lustre) | API HTTP sobre un espacio plano de claves |
| Fortaleza | Latencia más baja; cargas transaccionales | Semántica POSIX, bloqueo de archivos, acceso concurrente | Escala casi ilimitada, menor costo/GB, 11 nueves |
| Límite | 1 AZ y normalmente 1 instancia | Más caro por GB que S3 | Un objeto no se modifica por partes (se reemplaza entero); sin carpetas reales |

**Reglas/condiciones**
- Bloques: bloques de tamaño fijo con dirección y **sin metadatos**. Los nombres, carpetas y permisos los pone el sistema de archivos que se instala encima.
- Objeto = bytes + metadatos + clave única (`s3://bucket/clave`). S3 trata el contenido como opaco (para S3, un Parquet es «no estructurado»).
- En ML, con bloques o archivos el código abre rutas; con objetos usa la API (boto3, s3fs) o un servicio que copia o monta los datos.

**Diferencias o trampas**
- Ingesta = recolectar datos de distintas fuentes y enviarlos a AWS, por lotes o en tiempo real. Almacenamiento = persistirlos en un almacén de AWS.
- Almacenamiento de streaming (KDS, Kafka): registro append-only con retención. **Leer no borra**, a diferencia de una cola.

---

## 2 · Formatos de datos

**Qué recordar**

| Formato | Orientación | Clave para el examen |
|---|---|---|
| CSV | Filas, texto | Estructurado tabular |
| JSON / JSON Lines | Documento / un objeto JSON por línea | Semiestructurado; JSON Lines se procesa línea a línea |
| Parquet, ORC | **Columnar** | Compresión y codificación por columna (diccionario, RLE); se leen solo las columnas pedidas |
| Avro | **Filas**, binario | El esquema viaja con los datos: serialización compacta, sin sobrecarga por valor; **evolución de esquema** |
| RecordIO(-protobuf) | Secuencia de registros con la longitud antepuesta | Origen en MXNet; en SageMaker cada registro es protobuf; lectura en streaming y en paralelo |
| LibSVM | Texto disperso `etiqueta idx:valor` | Solo lista los valores distintos de cero |

**Reglas/condiciones**
- Escritura registro a registro (streaming, Kafka) → formato por filas (Avro). Lectura de pocas columnas de muchos registros (analítica, entrenamiento) → columnar (Parquet/ORC).
- Columnar + servicio que cobra por datos escaneados (Athena) → menos E/S y menos costo.
- Evolución de esquema en Avro: si el lector espera un campo ausente en el archivo, recibe el valor por defecto de su esquema; si el archivo trae un campo que el lector no usa, se ignora.
- Criterios para elegir el formato: **qué formatos acepta el algoritmo** y cuál es el patrón de acceso.

**Diferencias o trampas**
- Avro **no** es columnar.
- «Consultar pocas columnas» o «abaratar Athena» → Parquet/ORC, no Avro ni CSV.

---

## 3 · Formatos aceptados por los algoritmos integrados de SageMaker

**Qué recordar**

| Formatos | Algoritmos |
|---|---|
| RecordIO-protobuf, CSV | Factorization Machines, K-means, k-NN, Linear Learner, LDA, Neural Topic Model, PCA, Random Cut Forest |
| RecordIO, imágenes (.jpg/.png) | Image classification, Object detection, Semantic segmentation |
| CSV, LibSVM, Parquet | XGBoost |
| JSON Lines, Parquet | DeepAR forecasting |
| CSV (solo) | IP Insights |
| Texto (una oración por línea) | BlazingText |
| RecordIO-protobuf, texto | Seq2Seq |

**Diferencias o trampas**
- XGBoost es el único que acepta LibSVM. La documentación actual también lista RecordIO-protobuf. El entrenamiento distribuido con **Dask solo admite CSV y Parquet**.
- DeepAR usa JSON Lines, no CSV. IP Insights solo acepta CSV.
- Los algoritmos de visión usan RecordIO (no «-protobuf») o archivos de imagen.
- La tabla del libro no es exhaustiva.

---

## 4 · Patrón de acceso a los datos y particionamiento

**Qué recordar**
- Un patrón de acceso describe cómo productores y consumidores escriben, consultan y recuperan los datos. Campos: nombre, descripción, prioridad, operación (lectura/escritura), tipo (un elemento, varios o todos), filtro y orden.
- Ejemplo: «pedidos por ID de cliente en las últimas 24 h, ordenados por tiempo descendente».

**Reglas/condiciones**
- Tamaño de los datos → define el particionamiento.
- Forma → organizar los datos según las consultas.
- Velocidad (picos de consulta) → particionar para repartir la E/S. Si todo el pico cae en una partición, esa partición se satura aunque las demás estén ociosas.
- En S3 se particiona con prefijos (`anio=2024/mes=05/`) → cada consulta lee solo su partición.
- En DynamoDB, el ejemplo anterior se resuelve con `customer_id` como clave de partición y la fecha como clave de ordenamiento.

---

## 5 · Servicios según el tipo de dato

**Qué recordar**

| Estructurados | Semiestructurados | No estructurados |
|---|---|---|
| RDS, Aurora, Redshift, S3, Athena | DynamoDB, DocumentDB, Athena, S3 | S3, Rekognition, Transcribe, Comprehend |

- S3 es el único servicio que aparece en las tres columnas.
- DocumentDB: documentos JSON, compatible con MongoDB. Redshift: data warehouse. Aurora: compatible con MySQL/PostgreSQL.

**Diferencias o trampas**
- Rekognition, Transcribe y Comprehend **procesan** datos no estructurados; no los almacenan. Athena **consulta** S3; tampoco almacena. «¿Dónde guardar datos no estructurados?» → **S3**.
- Athena sobre S3 = schema-on-read: el esquema se declara en el Glue Data Catalog y los datos no se cargan en ninguna base de datos.

---

## 6 · Criterios para elegir un servicio de ingesta

**Reglas/condiciones**
- **Volumen, velocidad y variedad** determinan el servicio. Una velocidad alta exige streaming.
- **Escalabilidad**: absorber la velocidad y el volumen de todas las fuentes.
- **Resiliencia**: reanudar desde el punto de falla gracias a un checkpoint u offset, sin perder ni duplicar datos.
- **Seguridad y cumplimiento**: cifrado en tránsito y en reposo; PCI DSS (tarjetas de pago), HIPAA (salud, EE. UU.). La **residencia de datos** se cumple eligiendo la región y vigilando que ningún servicio copie datos a otra región.
- **Costo**: el streaming corre 24/7. Cobro por hora de capacidad ≈ 730 h/mes aunque haya poco tráfico; cobro por GB = proporcional al tráfico real.
- **Flexibilidad**: pipelines que se adapten al cambio.

---

## 7 · Amazon Data Firehose (antes Kinesis Data Firehose)

**Qué recordar**
- Conducto de **entrega** totalmente administrado y con autoescalado: origen → transformación opcional → destino. El recurso se llama *Firehose stream* (antes *delivery stream*).
- **No almacena para consumo**: ninguna aplicación lee de Firehose. Sin retención ni replay.
- **Orígenes**: Direct PUT (tu aplicación llama a la API), Kinesis Data Streams y topics de MSK. Integraciones: CloudWatch Logs, logs de web ACL de WAF, Network Firewall, SNS, AWS IoT (más de 20 servicios).
- **Destinos**: S3, Redshift, OpenSearch Service, Splunk, Snowflake, endpoint HTTP personalizado, tablas Apache Iceberg y plataformas de monitoreo de terceros.

**Reglas/condiciones**
- Conversión de formato a **Parquet/ORC** → la entrada debe ser **JSON**; el esquema se toma de una tabla del **Glue Data Catalog**.
- Transformación personalizada → **Lambda**. Ejemplos: CSV → JSON, o llamar a un endpoint de SageMaker para añadir una predicción a cada registro.
- Particionamiento dinámico → usa un atributo del registro para decidir el prefijo de destino (`cliente_id=123/`).
- Descompresión → para orígenes que entregan GZIP (p. ej., CloudWatch Logs).
- Búfer: se escribe al alcanzar el tamaño o el tiempo, lo que ocurra primero. Para S3, el valor por defecto es **5 MB / 300 s**; el intervalo va de **0 a 900 s**. Con 0 s, entrega «en unos pocos segundos».
- Intervalo corto → más archivos pequeños y más PUT a S3 → más costo.

**Diferencias o trampas**
- Firehose es **casi** tiempo real (por el búfer). Tiempo real (< 1 s), varios consumidores o relectura → **Kinesis Data Streams**.
- CSV que debe llegar como Parquet a S3 → **Firehose + Lambda**: Lambda convierte CSV a JSON y Firehose convierte a Parquet.
- «Cargar streaming en S3/Redshift/OpenSearch con configuración mínima» → Firehose.

**Detalles examinables**: casos de uso típicos: streaming a data lakes o warehouses con conversión a Parquet, observabilidad de seguridad hacia un SIEM (Splunk) y enriquecimiento con ML en vuelo mediante Lambda.

---

## 8 · Amazon Kinesis Data Streams (KDS)

**Qué recordar**
- Streaming en **tiempo real**: el retraso put-to-get suele ser **< 1 s**.
- **Almacena** los registros: retención de **24 h por defecto, hasta 365 días**. Admite múltiples consumidores independientes, cada uno a su ritmo, con relectura.
- Durabilidad: réplica en varias AZ.
- Capacidad por **shards**: cada uno admite **1 MB/s o 1000 registros/s de escritura**. La **clave de partición** que elijas (p. ej., el ID del dispositivo) decide el shard.
- Consumidores: Managed Service for Apache Flink, Spark Streaming, aplicaciones en EC2 y Lambda (casi tiempo real, sin servidores).

**Reglas/condiciones**

| Modo | Escalado | Condición → consecuencia |
|---|---|---|
| On-demand (Standard / Advantage) | AWS gestiona los shards | Admite hasta **2× el pico de escritura de los últimos 30 días**; superar el doble en < 15 min → rechazos temporales que el productor debe reintentar |
| Provisioned | Tú fijas los shards | No escala solo |

**Diferencias o trampas**
- El escalado protege la **escritura**. Si un consumidor se atrasa más que el periodo de retención → los registros **expiran sin haberse leído**.
- KDS no convierte formatos ni entrega a S3 por sí mismo: necesita un consumidor, o Firehose con KDS como origen.

**Detalles examinables**
- Logs: el evento sale del servidor en cuanto se produce, sin lotes locales → no se pierde si el servidor cae.
- Métricas y reportes en tiempo real.
- Varias aplicaciones en paralelo sobre el mismo stream (p. ej., clickstream).
- Combinar varios streams en otros nuevos.

---

## 9 · Amazon MSK (Managed Streaming for Apache Kafka)

**Qué recordar**
- Apache Kafka **de código abierto** totalmente administrado y de alta disponibilidad. Habla el mismo protocolo, así que apps, herramientas y plugins (Kafka Connect, Kafka Streams) funcionan **sin cambios de código**. Solo cambia la configuración: dirección del clúster y autenticación.
- **Plano de control** (API de AWS mediante consola, CLI o SDK): crear, eliminar o listar clústeres, ver propiedades y cambiar el número o tipo de brokers. **Plano de datos** (protocolo Kafka): crear topics, producir y consumir.
- **Provisioned** (tú eliges el número y tipo de brokers) vs **Serverless** (aprovisiona y escala la capacidad y gestiona las particiones; sin right-sizing).
- Kafka: un topic se divide en particiones (≈ shards). El orden solo se garantiza **dentro de una partición**, así que la misma clave va siempre a la misma partición. Los consumidores guardan su offset y pueden hacer replay. La retención por defecto de Kafka es de 7 días y es configurable.

**Reglas/condiciones (arquitectura)**
- **Brokers**: ingieren y almacenan las particiones. Se definen brokers **por AZ** (mínimo 1 por AZ), y cada AZ necesita su propia **subred** de la VPC.
- Un clúster provisioned se reparte en **2 o 3 AZ** según el tipo de broker y la región; los brokers **Express** exigen 3. El mínimo real es de 2 a 3 brokers.
- Réplicas por partición (normalmente 3): la **líder** recibe las escrituras y las seguidoras la copian. Si cae la líder, una seguidora la reemplaza.
- **Nodos controladores**: gestionan el estado de particiones y réplicas, la reasignación y la relación líder-seguidor.
- **ZooKeeper** (coordinación; MSK lo crea por ti) vs **KRaft** (metadatos en controladores dentro del propio clúster, sin costo adicional ni configuración). MSK admite KRaft desde Kafka **3.7.x** y permite migrar de ZooKeeper a KRaft. Kafka **4.0** (mar-2025) eliminó ZooKeeper.

**Diferencias o trampas**
- «Solo pagas lo que usas» en Serverless es una simplificación: cobra por hora de clúster, por hora de partición, por almacenamiento y por datos transferidos → **tiene costo base aunque no haya tráfico**.
- Solo **MSK Serverless** encaja en «sin clústeres que administrar».
- KDS tiene conceptos equivalentes pero una **API propia**: llevar apps Kafka a KDS obliga a reescribirlas.
- **MSK Connect** ejecuta conectores de Kafka Connect como servicio administrado.

**Detalles examinables**
- Casos de uso: eventos de aplicaciones y de bases de datos (CDC) hacia un data lake.
- **System of record** / event sourcing: el estado se reconstruye releyendo el topic, lo que exige una retención larga o indefinida.
- Arquitecturas orientadas a eventos: si un consumidor cae, los eventos lo esperan en el topic.

---

## 10 · Amazon Managed Service for Apache Flink (antes Kinesis Data Analytics)

**Qué recordar**
- Procesamiento de flujos **con estado** (ventanas, promedios móviles, sesiones). El estado se guarda periódicamente en **checkpoints**.
- **No almacena datos**: lee y escribe en KDS, MSK, S3, etc. Patrón típico: KDS/MSK → Flink → S3 o una base de datos.
- Tolerancia a fallas y **exactly-once**. At-most-once puede perder eventos y at-least-once puede duplicarlos. El exactly-once de punta a punta requiere que el origen y el destino colaboren (escrituras transaccionales).
- APIs: Java, Scala, Python y **SQL**. **Studio**: notebooks (Apache Zeppelin) para consultar streams en vivo de forma interactiva.
- Procesa datasets **acotados** (batch) y flujos **no acotados** (consultas continuas).

**Reglas/condiciones**
- Facturación: por hora de **KPU** asignada (1 vCPU + 4 GB) → pagas lo asignado aunque esté ocioso.
- SQL en tiempo real sobre un stream → Flink (antes, KDA for SQL).

**Diferencias o trampas**
- El libro compara con «Apache MSK» (errata: es **Amazon** MSK) y dice que ninguno tiene clústeres que administrar. Eso solo es cierto para MSK **Serverless**.
- Detección de anomalías con SQL sobre un stream → `RANDOM_CUT_FOREST` de KDA SQL, **retirado**. Random Cut Forest sigue existiendo como algoritmo integrado de SageMaker.

**Detalles examinables**
- Casos de uso:
  - ETL de streaming e ingesta continua en un data lake.
  - Medición y facturación, monitoreo de red, feature engineering en vivo y rendimiento de campañas.
  - Apps orientadas a eventos: detección de fraude, monitoreo de procesos y geo-fencing.

---

## 11 · Comparativa de servicios de streaming

| | Firehose | Kinesis Data Streams | MSK | Managed Flink |
|---|---|---|---|---|
| Rol | Entrega a un destino | Almacén de stream | Almacén de stream (Kafka) | Cómputo sobre streams |
| Almacena / replay | No | Sí (24 h a 365 d) | Sí (retención configurable) | No |
| Latencia | Casi tiempo real (búfer) | < 1 s | Tiempo real | Tiempo real |
| Consumidores propios | No | Sí, múltiples | Sí, grupos de consumidores | — |
| Transforma | Formato (JSON → Parquet/ORC), Lambda, particionado dinámico, descompresión | No (lo hacen los consumidores) | No (Kafka Streams / Flink) | Sí, con estado y exactly-once |
| Interfaz | API de AWS | API propia de AWS | Protocolo Kafka | Flink (Java/Scala/Python/SQL) |

**Reglas/condiciones (síntesis del libro)**
- Procesamiento **personalizado** en tiempo real → **KDS**.
- Cargar streaming en almacenes de AWS con configuración mínima → **Firehose**.
- Gestionar y distribuir streaming (agregación de logs, analítica en tiempo real, event sourcing, ecosistema Kafka existente) → **MSK**.
- Procesar y analizar en tiempo real (SQL, estado, ventanas) → **Managed Flink**.
- Ingesta en tiempo real: Firehose, KDS, MSK y Flink. Ingesta **por lotes**: DataSync y Glue.

**Diferencias o trampas**
- SNS reparte mensajes a sus suscriptores pero no los retiene para releerlos.
- Firehose puede leer de MSK o KDS, pero no los reemplaza.

---

## 12 · AWS DataSync

**Qué recordar**
- Transferencia y migración **por red** de archivos u objetos hacia, desde y entre servicios de almacenamiento de AWS. Los extremos pueden estar on-premises, en otras nubes o ser autoadministrados (p. ej., un servidor NFS en EC2).
- **Agente**: una VM que instalas dentro de tu red. AWS no puede entrar a tu red privada, así que el agente lee desde dentro y envía los datos.
- Orígenes externos: **NFS, SMB, HDFS** y almacenamiento de objetos (Google Cloud Storage, Azure Blob, Wasabi y almacenamiento autoadministrado compatible con la API de S3).
- Servicios de AWS: **S3, EFS, FSx for Windows, FSx for Lustre, FSx for OpenZFS, FSx for ONTAP**, Snowcone y Snowball Edge (ver estado).

**Reglas/condiciones**
- Automatiza la programación, los reintentos, la verificación y la **copia incremental** (en cada ejecución solo copia lo que cambió). Sirve para migrar datasets activos hasta el corte final.
- Seguridad: cifrado, **validación de integridad** (checksums en origen y destino), **roles IAM** para acceder a AWS y **endpoints de VPC** para no pasar por internet.
- Velocidad: protocolo propio y arquitectura paralela multihilo.
- Precio fijo por GB transferido (verifica el precio vigente).

**Diferencias o trampas**
- Discovery (métricas y recomendaciones de migración) **ya no existe**.
- Red vs dispositivo físico: 500 TB a 1 Gbps ≈ 46 días. Hoy, los clientes nuevos usan DataSync o Data Transfer Terminal en lugar de Snow.
- DataSync es transferencia por lotes, no streaming. DataSync mueve datos; Glue hace ETL.

**Detalles examinables**: casos de uso: migrar datasets activos; **archivar datos fríos directamente en S3 Glacier Flexible Retrieval o Deep Archive**; replicar a cualquier clase de S3 o a EFS / FSx (Windows, Lustre, OpenZFS) como sistema de reserva (standby); transferencias híbridas recurrentes para procesar datos (ML en ciencias de la vida, video, finanzas, sísmica).

---

## 13 · AWS Glue

**Qué recordar**
- ETL **serverless**: no hay servidores que aprovisionar ni cuyo ciclo de vida administrar.
- **Crawlers**: recorren los datos (p. ej., un prefijo de S3), infieren el esquema y los tipos, y los registran como tablas en el **Glue Data Catalog**. Es el mismo catálogo que usan Athena y la conversión de formato de Firehose.
- Genera automáticamente scripts de ETL (del editor visual sale código PySpark).
- Conectores integrados: Redshift, Aurora, SQL Server, MySQL, MongoDB, MariaDB y PostgreSQL, más **controladores JDBC personalizados** (así alcanza fuentes on-premises).
- Frameworks: Spark (PySpark, Scala) y Ray (Glue for Ray no acepta clientes nuevos).

**Reglas/condiciones**
- **Auto Scaling** de workers durante el job → encaja con demanda irregular, volumen impredecible y muchas fuentes. Pagas los workers usados.
- Más de 70 fuentes. Cargas por lotes, **microlotes** y streaming (el ETL de streaming de Glue funciona con microlotes de Spark).
- **Sesiones interactivas**: Spark serverless bajo demanda desde un IDE o notebook, con cobro por tiempo de uso.

**Diferencias o trampas**
- Glue = ETL y pipelines de datos; DataSync = transferir o migrar.
- Descubrir datos en AWS, on-premises o en otras nubes → crawlers + Data Catalog (on-premises, vía JDBC).

---

## 14 · Criterios para elegir almacenamiento

| Factor | Pregunta | Detalle examinable |
|---|---|---|
| Durabilidad | ¿Cuánto tiempo conservar los datos? | S3: **99.999999999 %** (con 10 M de objetos, pierdes 1 cada 10 000 años de media). EBS de uso general: **99.8–99.9 % anual** (tasa de falla anual de 0.1–0.2 %) |
| Disponibilidad | ¿Qué tan pronto hay que usarlos? | 99.99 % ≈ 53 min/año sin servicio · 99.9 % ≈ 8.8 h · 99.5 % ≈ 44 h |
| Tipo | ¿En qué forma se accede? | Objetos, bloques, archivos, bases de datos, streaming |
| Costo | ¿Cuánto gastar? | Pilar del Well-Architected Framework |
| Seguridad | ¿Qué protección necesitan en reposo? | Cifrado en reposo; pilar del Well-Architected Framework |

**Reglas/condiciones**
- Durable ≠ disponible: un dato puede estar a salvo y aun así ser inaccesible durante una caída.
- El Well-Architected Framework tiene 6 pilares: excelencia operativa, seguridad, confiabilidad, eficiencia del rendimiento, optimización de costos y sostenibilidad.
- El almacenamiento también guarda los **artefactos** del modelo (`model.tar.gz` en S3 con los pesos, que luego se usa para desplegar).
- **Integración nativa con SageMaker** (canal de entrada de un training job): **S3, EFS y FSx for Lustre**. Con cualquier otro almacén, tu código tiene que ir a buscar los datos. **[C01]** para EFS/FSx.

---

## 15 · Amazon S3

**Qué recordar**
- Almacenamiento de objetos con escalabilidad casi ilimitada y el menor costo por GB. Se accede por API y se integra de forma nativa con SageMaker (datos de entrenamiento y artefactos), Lambda, ECS y EKS.
- Las clases estándar guardan los datos en **≥ 3 AZ de una misma región**; las clases One Zone, en 1 AZ.
- **Consistencia fuerte de lectura tras escritura** desde dic-2020.
- Latencia de S3 Standard en objetos pequeños: ~100–200 ms.

**Diferencias o trampas**
- S3 **no** copia datos a otra región por su cuenta; hace falta configurar **Cross-Region Replication**.
- En el entrenamiento de LLM, S3 es el **repositorio**, pero se suele poner una capa rápida delante (FSx for Lustre o S3 Express One Zone) para no dejar las GPU esperando datos.
- Las **políticas de ciclo de vida** mueven los objetos a otra clase o los borran tras N días.

**Detalles examinables**
- Casos de uso: data lakes y ML/HPC; backup y restauración según **RTO** (tiempo máximo para restaurar el servicio) y **RPO** (pérdida máxima aceptable de datos, medida en tiempo); archivo en las clases Glacier; GenAI.
- Escala: más de 500 billones de objetos, cientos de EB y más de 200 M de peticiones/s (mar-2026). El libro daba 350 billones y 100 M.

---

## 16 · Clases de almacenamiento de S3

| Clase | Uso | Disponibilidad | AZ | Duración mínima facturada | Cargo por recuperación |
|---|---|---|---|---|---|
| Standard | Acceso frecuente | 99.99 % | ≥ 3 | — | No |
| Intelligent-Tiering | Patrón de acceso cambiante o desconocido | 99.9 % | ≥ 3 | — | No (hay cargo de monitoreo por objeto) |
| Express One Zone | Latencia constante de 1 dígito de ms | 99.95 % | 1 | — | No |
| Standard-IA | Acceso poco frecuente pero rápido | 99.9 % | ≥ 3 | 30 d | Sí |
| One Zone-IA | Acceso poco frecuente sin resiliencia multi-AZ; más barata | 99.5 % | 1 | 30 d | Sí |
| Glacier Instant Retrieval | Larga vida, acceso raro pero inmediato | 99.9 % | ≥ 3 | 90 d | Sí |
| Glacier Flexible Retrieval | Archivo; acceso de minutos a horas | 99.99 %\* | ≥ 3 | 90 d | Sí |
| Glacier Deep Archive | Costo mínimo a largo plazo; recuperación estándar ≤ **12 h**, masiva (bulk) ≤ **48 h** | 99.99 %\* | ≥ 3 | 180 d | Sí |

\*Tras la restauración. Todas las clases tienen una durabilidad de 11 nueves. **S3 on Outposts** lleva S3 a tu centro de datos (rendimiento local y residencia de datos).

**Reglas/condiciones**
- Glacier Flexible Retrieval y Deep Archive → **hay que restaurar antes de leer**. Glacier Instant Retrieval → lectura directa.
- Borrar un objeto antes de la duración mínima → pagas igual los días restantes.
- «Accesible en ≤ 12 h, conservar años, lo más barato» → **Deep Archive**.

**Diferencias o trampas**
- Intelligent-Tiering (el libro dice «dos niveles», dato desactualizado): tiene **3 niveles automáticos**, Frequent → Infrequent (tras 30 d sin acceso) → Archive Instant Access (tras 90 d), y **2 opcionales** que hay que activar, Archive Access y Deep Archive Access, cuyos objetos deben restaurarse antes de leerse.
- En Intelligent-Tiering, los objetos de **< 128 KB** no se monitorean y se quedan siempre en Frequent.

---

## 17 · Amazon Athena

**Qué recordar**
- Consultas SQL estándar, serverless e interactivas (resultados en segundos) sobre datos en S3: CSV, JSON, Parquet y ORC.
- Toma las definiciones de tabla (columnas y ubicación de los archivos) del **Glue Data Catalog**. Es schema-on-read: no carga los datos.
- Cobra por **datos escaneados** → los formatos columnares y el particionamiento abaratan cada consulta.

**Diferencias o trampas**
- Athena no almacena nada.
- S3 (+ ciclo de vida hacia Glacier) + Athena = repositorio barato consultable con SQL.
- **Redshift Spectrum**: Redshift consulta archivos en S3 sin cargarlos. **Redshift Streaming Ingestion**: Redshift lee directamente de **KDS o MSK**, sin pasar por S3.

---

## 18 · Amazon EFS

**Qué recordar**
- Almacenamiento de archivos **serverless y totalmente elástico**: no eliges tamaño y pagas los GB guardados. Es regional (datos en varias AZ).
- Protocolo **NFS v4.0 y v4.1**. **No admite instancias EC2 con Windows.**
- Lo montan a la vez EC2, EKS, ECS, Lambda y servidores on-premises.
- El ciclo de vida mueve los archivos fríos a las clases **Infrequent Access** y **Archive** (primer byte en decenas de ms; la clase Standard usa SSD, ~1 ms).
- Protección: AWS Backup y replicación de EFS (a otra región o zona).

**Reglas/condiciones**
- Modo de throughput *Elastic*: cada sistema de archivos regional da **20–60 GiB/s de lectura** y **1–5 GiB/s de escritura**, según la región, con ~1 ms de latencia en lectura y 2.7 ms en escritura.
- Un solo cliente llega a ~1500 MiB/s (con amazon-efs-utils ≥ 2.0). Los GB/s totales se alcanzan sumando muchos clientes.
- **[C01] Con SageMaker**: el training job debe **conectarse a la VPC**, con subredes y grupos de seguridad que alcancen los **mount targets** de EFS. El grupo de seguridad debe permitir NFS en el puerto **2049**. Los endpoints de VPC solo hacen falta si la VPC no tiene internet, para llegar a S3 y demás APIs. El libro dice «endpoint de VPC de interfaz para EFS», lo cual es impreciso.

**Diferencias o trampas**
- EFS vs S3: EFS ofrece semántica de sistema de archivos (modificar en el lugar, append, renombrar, permisos POSIX), **bloqueo de archivos** y acceso compartido. La consistencia fuerte **ya no los distingue**: S3 también la tiene.
- Costo: EFS Standard ≈ **10× S3 Standard** por GB.
- Acceso rápido a datos de entrenamiento en un sistema de archivos compartido → EFS. Almacenamiento barato a gran escala → S3.
- Archivos en cantidad impredecible compartidos por muchos consumidores → EFS. EBS tiene tamaño fijo y sirve a una sola instancia; S3 no tiene semántica de archivos.

**Detalles examinables**
- Casos de uso: ciencia de datos y ML (SageMaker Studio Classic guarda en EFS los directorios personales de los usuarios); almacenamiento persistente y compartido para contenedores y serverless; CMS con varios servidores web (WordPress).

---

## 19 · Amazon FSx for Lustre

**Qué recordar**
- Sistema de archivos **paralelo** y distribuido que escala horizontalmente (scale-out). Cada archivo se reparte en franjas (stripes) entre muchos servidores, y los metadatos viven en servidores aparte → el throughput de todos se suma.
- **Solo clientes Linux** (cliente Lustre para Amazon Linux, RHEL, Ubuntu y SUSE). No hay acceso desde Windows ni por SMB.
- Latencia **por debajo del ms**, hasta **varios TB/s** por sistema de archivos (el libro dice «cientos de GB/s»), millones de IOPS y hasta 1200 Gbps por cliente con **EFA**.
- Compatible con POSIX, con bloqueo de archivos y consistencia de lectura tras escritura.

**Reglas/condiciones**
- **Data repository association** con un bucket de S3: el listado de objetos (metadatos) se importa de inmediato; el contenido se trae al primer acceso (**lazy loading**) salvo que lo precargues. Los resultados pueden escribirse de vuelta en S3.
- Carga **única o precarga** → mejor rendimiento, mayor costo inicial. **Lazy loading** → mínima transferencia inicial, primer acceso más lento.
- Despliegue **scratch** → sin replicación; los datos no sobreviven a la falla de un servidor. Sirve para trabajos cortos cuyos originales siguen en S3. Despliegue **persistent** → replicado, reemplaza los servidores que fallan; para largo plazo y cargas centradas en el throughput.
- Clases: SSD (sub-ms constante), HDD (1 dígito de ms) e Intelligent-Tiering (elástica, con caché SSD opcional).
- Se integra con AWS ParallelCluster y AWS Batch.
- **[C01] Con SageMaker**: origen nativo de datos de entrenamiento. El job se conecta a la VPC, con una subred de la **misma AZ** que el sistema de archivos.

**Diferencias o trampas**
- File mode (por defecto en S3) copia el dataset completo a cada instancia antes de entrenar. Lustre se **monta** y el entrenamiento empieza de inmediato.
- Alternativas sin FSx: **FastFile mode** (lee de S3 bajo demanda) y **Pipe mode** (lee como flujo).
- Prioridad absoluta en el tiempo de entrenamiento → FSx for Lustre.

**Detalles examinables**
- Casos de uso: ML con datasets grandes y latencia sub-ms; HPC (genómica, clima, simulaciones, Monte Carlo); medios (renderizado, transcodificación, edición).

---

## 20 · S3 vs EFS vs FSx for Lustre para entrenar en SageMaker [C01]

| Dimensión | S3 | EFS | FSx for Lustre |
|---|---|---|---|
| Tiempo de entrenamiento | Más lento | Intermedio | **Más rápido** |
| Costo | **Más barato** | Intermedio (equilibrio) | Más caro |
| Por qué | Tarifa más baja, latencia más alta | Pagas solo lo guardado, pero ~10× S3 por GB | Pagas la capacidad aprovisionada (SSD/HDD) mientras el sistema exista, y los datos siguen facturándose en S3 |

**Reglas/condiciones**
- Tiempo de entrenamiento → FSx for Lustre. Costo crítico → S3. Equilibrio entre costo y rendimiento → EFS.

---

## 21 · Amazon FSx for NetApp ONTAP

**Qué recordar**
- NetApp ONTAP administrado, con almacenamiento **unificado**: NAS (archivos por NFS/SMB), SAN (bloques por iSCSI y NVMe/NVMe-over-TCP) y objetos, sobre flash, disco y nube.
- Acceso **multiprotocolo sobre los mismos datos**: clientes Linux por NFS y Windows por SMB a la vez, con integración con Active Directory.
- **SnapMirror** (la replicación nativa de NetApp, compatible con NetApp on-premises), FlexCache, snapshots y clones, paso automático a un nivel más barato, compresión y **deduplicación**.
- Latencia sub-ms, millones de IOPS, decenas de GB/s por sistema de archivos y scale-out.
- También se administra con la CLI y la API REST de ONTAP.

**Diferencias o trampas**
- Es el **único FSx con iSCSI (bloques) y multiprotocolo**.
- Mismos datos por SMB y NFS, base de datos certificada solo en iSCSI, o réplica SnapMirror hacia NetApp on-premises → ONTAP.

**Detalles examinables**
- Casos de uso: lift and shift desde NetApp o desde servidores NFS/SMB/iSCSI/NVMe-over-TCP sin tocar las aplicaciones; **BCDR** (backup, archivo y replicación entre on-premises y AWS o entre regiones); bases de datos de alto rendimiento (Oracle o SQL Server sobre NFS, SMB o iSCSI).

---

## 22 · Amazon FSx for Windows File Server

**Qué recordar**
- Servidor de archivos Windows nativo y administrado, con **SMB 2.0 a 3.1.1**. Clientes: Windows (desde 7 / Server 2008) y Linux actual.
- **Debe unirse a un Active Directory al crearse**. Los permisos son ACL de usuarios y grupos de AD.
- La deduplicación de Windows Server trabaja por fragmentos dentro de los archivos → menos costo.
- Almacenamiento SSD (baja latencia, altas IOPS) o HDD (más barato: directorios personales y carpetas departamentales). Despliegue Single-AZ o Multi-AZ (con servidor de reserva). Latencia sub-ms. Se administra con PowerShell.

**Reglas/condiciones**
- **SQL Server en alta disponibilidad sin licencias Enterprise**: las Failover Cluster Instances (FCI) están en la edición Standard (2 nodos) pero necesitan almacenamiento compartido, y FSx for Windows lo aporta por SMB. Los Always On Availability Groups completos sí requieren Enterprise.
- Escritorios virtuales (**WorkSpaces**, **AppStream 2.0 / WorkSpaces Applications**): los perfiles de usuario viven en almacenamiento compartido. Con FSLogix, el perfil se monta como disco virtual → inicio de sesión más rápido.

**Diferencias o trampas**
- No ofrece NFS, iSCSI ni SnapMirror → si se necesitan, ONTAP.
- Clúster Linux de HPC que espera POSIX → no es esta opción.

**Detalles examinables**: casos de uso: directorios personales, perfiles de usuario, aplicaciones empresariales con carpetas compartidas y migración de servidores de archivos Windows sin cambiar las aplicaciones.

---

## 23 · Amazon FSx for OpenZFS

**Qué recordar**
- OpenZFS administrado: copy-on-write, checksums por bloque, snapshots, **clones** escribibles instantáneos, compresión transparente y cuotas.
- **NFS v3, v4.0, v4.1 y v4.2** (EFS solo admite 4.0 y 4.1). Clientes Linux, Windows y macOS, a través de sus clientes NFS.
- Hasta **2 M de IOPS** (el libro dice > 1 M) con latencias de cientos de µs.
- Throughput de hasta **21 GB/s** desde la caché (memoria o NVMe). Desde disco: 400 000 IOPS y **10 GB/s** (21 GB/s con compresión).
- Clase Intelligent-Tiering, despliegue Multi-AZ y **S3 Access Points** que se pueden adjuntar a los volúmenes para leer los mismos datos con la API de S3.

**Diferencias o trampas**
- Los snapshots y clones se crean con la **API, consola o CLI de FSx**, no con `zfs snapshot` / `zfs clone`, porque no tienes acceso al servidor. Los scripts existentes deben adaptarse.
- Permisos nativos de Windows → FSx for Windows, no OpenZFS.
- Solo habla NFS: sin SMB nativo, sin iSCSI y sin SnapMirror.

**Detalles examinables**
- Casos de uso: migrar ZFS o servidores de archivos Linux por NFS sin cambiar las aplicaciones (basta con cambiar la dirección de montaje); aplicaciones intensivas en datos (analítica, preprocesamiento para ML, compilaciones con muchos archivos pequeños, servidores web).

---

## 24 · Comparativa de almacenamiento de archivos

| Servicio | Protocolos | Clientes | Diferenciador | Señal en el enunciado |
|---|---|---|---|---|
| EFS | NFS 4.0/4.1 | Linux (no EC2 Windows) | Serverless y elástico, sin dimensionar | Archivos compartidos, volumen impredecible, contenedores o Lambda |
| FSx for Lustre | Cliente Lustre | Solo Linux | Paralelo, TB/s, vinculado a S3 | HPC, entrenamiento más rápido, dataset en S3 |
| FSx for ONTAP | NFS, SMB, iSCSI, NVMe | Linux + Windows | Multiprotocolo, SnapMirror, dedup | Migrar NetApp, mismos datos por SMB y NFS, bloques iSCSI |
| FSx for Windows | SMB | Windows (+ Linux) | AD obligatorio, ACL | Directorios personales, perfiles, SQL Server FCI, WorkSpaces |
| FSx for OpenZFS | NFS v3 a v4.2 | Linux, Windows, macOS | Snapshots y clones ZFS, 2 M IOPS | Migrar ZFS o un servidor NFS Linux, NFSv3 |

---

## 25 · Amazon EBS (Elastic **Block Store**)

**Qué recordar**
- Almacenamiento de bloques que funciona como una SAN en la nube: los volúmenes son discos virtuales para EC2.
- Un volumen vive en **1 AZ** y solo se conecta a instancias de esa AZ, normalmente a **1 instancia**.
- **Multi-Attach**: solo en **io1/io2**, para pocas instancias de la misma AZ, y exige un sistema de archivos o una aplicación que coordine las escrituras.
- SSD → cargas transaccionales con E/S pequeña; métrica clave: **IOPS**. HDD → lectura secuencial de archivos grandes («streaming» en este contexto); métrica clave: **throughput**.

| Tipo | Medio | IOPS máx. | Throughput máx. |
|---|---|---|---|
| gp3 | SSD | 80 000 | 2000 MiB/s |
| gp2 | SSD | 16 000 | 250 MiB/s |
| io2 Block Express | SSD | 256 000 | 4000 MiB/s |
| io1 | SSD | 64 000 | 1000 MiB/s |
| st1 | HDD | 500 | 500 MiB/s |
| sc1 | HDD | 250 | 250 MiB/s |

**Reglas/condiciones**
- throughput ≈ IOPS × tamaño de cada E/S. Ejemplo: st1 da 500 IOPS × 1 MiB = 500 MiB/s.
- **Snapshots incrementales**, guardados en S3 pero no visibles en tus buckets. Restaurar un snapshot en otra AZ es la forma de «mover» un volumen de AZ.
- Un volumen se puede agrandar o cambiar de tipo sin desconectarlo, y se puede desconectar y reconectar a otro nodo de la misma AZ.

**Diferencias o trampas**
- El nombre oficial es *Elastic Block Store*, no *Storage*.
- Durabilidad de 99.8–99.9 % anual, muy inferior a S3.
- No se comparte entre muchas instancias (→ EFS/FSx) ni cruza AZ.
- Base de datos autoadministrada en EC2 → EBS. Motor disponible en RDS y sin necesidad de control total → RDS.

**Detalles examinables**
- Casos de uso: migrar una SAN on-premises de gama media y aplicaciones de misión crítica; bases de datos **autoadministradas** (SAP HANA, Oracle, SQL Server, PostgreSQL, MySQL, Cassandra, MongoDB); clústeres de Hadoop o Spark.

---

## 26 · Amazon RDS

**Qué recordar**
- Bases de datos relacionales administradas, con **8 motores**: Aurora PostgreSQL, Aurora MySQL, RDS for PostgreSQL, MySQL, MariaDB, SQL Server, Oracle y Db2.
- **Aurora**: motor de AWS compatible con MySQL/PostgreSQL. Su almacenamiento distribuido guarda **6 copias en 3 AZ**, y puede escalar réplicas automáticamente.
- AWS se encarga del aprovisionamiento, los parches, los backups, la recuperación y la detección y reparación de fallas.

**Reglas/condiciones**
- Con el mismo motor no hay que refactorizar ni tocar los procedimientos almacenados: solo cambia la cadena de conexión.
- Funciones que exigen acceso de administrador al SO → **RDS Custom** (solo Oracle y SQL Server).
- Despliegues: nube (Aurora o RDS), híbrido (**RDS on Outposts**) y acceso privilegiado (**RDS Custom**).
- Precio: horas de instancia encendida + almacenamiento, no por consulta. Las instancias reservadas dan descuento.
- Las **réplicas de lectura** escalan el tráfico de lectura.

**Diferencias o trampas**
- Ahorro de licencias al migrar, dos caminos: (a) cambiar a un motor de código abierto (Oracle → PostgreSQL/Aurora), que **sí obliga a convertir el SQL**; (b) conservar el motor con **licencia incluida** (se paga por hora).

**Detalles examinables**: casos de uso: aplicaciones web y móviles modernas; migración de bases de datos heredadas.

---

## 27 · Amazon DynamoDB

**Qué recordar**
- NoSQL **serverless** de **clave-valor y documentos**, con latencia constante de **1 dígito de ms a cualquier escala**. Mantenimiento sin tiempo de inactividad.
- Lecturas **eventualmente consistentes por defecto**; la lectura **fuertemente consistente** se pide explícitamente en cada petición.
- **Transacciones ACID** sobre una o más tablas en una sola petición.

**Reglas/condiciones**
- Sin JOIN ni operaciones cuyo costo crezca con el tamaño de la tabla → las tablas se diseñan **a partir de los patrones de acceso** (clave de partición + clave de ordenamiento).
- **Query** lee una sola clave de partición. **Scan** lee la tabla completa y su costo crece con el tamaño → evita Scan.
- El modo on-demand se ajusta al instante, pero tiene límites ante picos muy bruscos.

**Diferencias o trampas**
- DynamoDB no es el repositorio central de datos crudos para ML → S3.

**Detalles examinables**
- Casos de uso: servicios financieros (ACID, picos del mercado); videojuegos (estado, jugadores, sesiones, leaderboards; scale in/out); streaming de medios (índice de metadatos, listas de seguimiento, marcadores, estadísticas deportivas casi en tiempo real, eventos para recomendaciones).

---

## 28 · Resolución de problemas de capacidad y escalabilidad

| Problema | Herramienta o acción |
|---|---|
| Anomalías de CPU, memoria o E/S | **CloudWatch**: métricas + alarmas (se disparan cuando una métrica cruza un umbral durante cierto tiempo) |
| Causa raíz: ¿quién cambió qué? | **CloudTrail**: registra cada llamada a la API (quién, qué, cuándo, desde dónde) |
| Demanda variable | Auto scaling de EC2 y demás recursos escalables |
| Mucho tráfico de lectura en la BD | Réplicas de lectura; auto scaling de réplicas en Aurora |
| Pipeline lento | Particionamiento, caché y paralelismo; en streaming, MSK / Flink / KDS / Firehose + Lambda |
| Costo o capacidad de almacenamiento | Políticas de ciclo de vida de S3; monitorear EBS y ajustar tipo o tamaño |
| BD lenta | Optimizar consultas e índices; sharding; en DynamoDB, evitar Scan |
| Gasto | **Cost Explorer**; instancias reservadas o **Savings Plans** (1 o 3 años) para cargas predecibles; **Spot** (hasta 90 % de descuento, aviso de 2 min) para cargas interrumpibles (batch, entrenamiento con checkpoints) |

---

## 29 · Escenarios de decisión: restricción → servicio

**FSx for Lustre** (pipeline genómico en ~400 nodos EC2 Linux, datos en S3)
- Herramientas de terceros que no se pueden modificar y hacen E/S aleatoria por ruta → POSIX y bloqueo.
- ~100 GB/s de lectura agregada y decenas de GB/s de escritura → sistema de archivos paralelo.
- Todos los nodos ven el mismo espacio de archivos; datos en S3 sin copia manual → data repository association.
- Almacenamiento solo durante la corrida → **scratch**.
- Por qué no las alternativas:
  - S3: no es un sistema de archivos. Mountpoint for S3 solo hace escrituras secuenciales y no ofrece bloqueos.
  - EFS: escribe 1–5 GiB/s y no se vincula a S3.
  - EBS: sirve a 1 instancia.
  - OpenZFS: 21/10 GB/s y no importa objetos de S3.
  - ONTAP: decenas de GB/s; su multiprotocolo no aporta aquí.
  - Windows: SMB, no POSIX.

**FSx for NetApp ONTAP** (cierre de un centro de datos con NetApp)
- Mismos datos por SMB (con AD) y NFS → multiprotocolo.
- Base de datos certificada solo en iSCSI → volúmenes de bloques.
- Recuperación ante desastres con SnapMirror hacia el NetApp on-premises → SnapMirror.
- Lift and shift con un equipo que conoce ONTAP → CLI y API de ONTAP.
- Histórico enorme → dedup, compresión y tiering.
- Para ML: DataSync copia el histórico de ONTAP a S3 y se entrena desde S3.
- Por qué no las alternativas:
  - Windows: sin NFS, iSCSI ni SnapMirror.
  - OpenZFS: solo NFS.
  - Lustre: solo Linux.
  - EFS: solo NFS, sin Windows.
  - EBS: no comparte archivos ni replica con SnapMirror.
  - S3: sin SMB, NFS ni iSCSI.

**MSK** (60 microservicios con clientes Kafka y Kafka Streams)
- Sin cambios de código → mismo protocolo.
- Reutilizar conectores → MSK Connect.
- Replay de 30 días con varios grupos de consumidores → retención configurable y offsets por grupo.
- Nadie opera servidores → MSK (o Serverless).
- Orden por vehículo → clave = ID del vehículo → misma partición.
- Por qué no las alternativas:
  - KDS: API propia, obliga a reescribir.
  - Firehose: no almacena ni permite replay.
  - Flink: procesa pero no almacena.
  - SNS: no retiene mensajes.
  - Kafka en EC2: hay que operarlo.

---

## 30 · Señales de enunciado (preguntas de repaso del capítulo)

El .md original no incluye la clave de respuestas. Estas respuestas se deducen de las reglas del capítulo.

| Señal en el enunciado | Respuesta |
|---|---|
| Datos crudos de IoT, repositorio central, alta disponibilidad | S3 |
| Datos procesados accesibles al instante durante 6 meses + crudos accesibles en ≤ 12 h durante 6 años + SQL + lo más barato | S3 + Athena (ciclo de vida hacia Glacier Deep Archive) |
| Firehose con registros GZIP + SQL en tiempo real sobre el flujo | Managed Service for Apache Flink (antes KDA SQL) + Lambda |
| Anomalías en transacciones en streaming, data lake en S3, menor carga operativa | Firehose + `RANDOM_CUT_FOREST` de KDA SQL/Flink (inferido; esa función SQL ya no existe) |
| Geolocalización hacia Redshift casi en tiempo real, lo más barato | KDS + Redshift Streaming Ingestion (lee de KDS/MSK, no de Firehose; Spectrum consulta S3, no ingiere streams) |
| CSV casi en tiempo real → Parquet en S3 | Firehose + Lambda (CSV → JSON; Firehose convierte a Parquet) |
| Entrenamiento de imágenes a gran escala en SageMaker, alto throughput y baja latencia, datos en S3 | FSx for Lustre |
| Guardar y consultar con frecuencia muchos datos de sensores, de forma económica | S3 |
