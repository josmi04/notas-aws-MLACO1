---
tema: Almacenamiento y formatos de datos para entrenar en AWS
nota: 3/29
guia-mla-c01: [1.1 K formatos e ingesta, 1.1 K fuentes de datos (S3, EFS, FSx ONTAP), 1.1 K opciones de almacenamiento, 1.1 S extracción (S3, EBS, EFS, RDS, DynamoDB), 1.1 S elección de formato por patrón de acceso, 1.1 S capacidad y escalabilidad, 1.1 S decisiones iniciales de almacenamiento, 1.3 S cargar datos en el recurso de entrenamiento]
prerrequisitos: [nota 1 (IAM, perfiles, roles de ejecución, políticas, boto3), nota 2 (jobs de SageMaker, contrato /opt/ml, canales, CreateTrainingJob, CreateProcessingJob, estados y SecondaryStatus, waiters, ModelTrainer)]
no-se-usa-aqui: [Kinesis y Data Firehose, Glue, Athena, Lake Formation, Feature Store, algoritmos integrados en detalle, KMS más allá de pasar la llave, VPC (caja negra declarada), endpoints de inferencia, costos detallados]
region: us-east-1
versiones:
  aws-cli: "2.36.49"
  boto3: "1.43.98"
  sagemaker-python-sdk: "3.22.1 (sagemaker-core 2.22.1, sagemaker-train 1.22.1)"
verificado: 2026-09-20
tags: [aws, mla-c01, s3, ebs, efs, fsx, parquet, recordio, sagemaker, input-modes]
---

# Almacenamiento y formatos de datos para entrenar en AWS

## Quinientos gigabytes de transacciones y tres decisiones

El histórico de transacciones de Kanan —unos 500 GB de CSV en
`s3://kanan-ml-dev-raw-us-east-1/fraude/2026/transacciones/`— ya está en el sitio
correcto. S3 es donde viven los datos de entrenamiento en AWS salvo excepción
justificada, y esta nota dedica buena parte de su extensión a explicar cuáles son
esas excepciones y cómo se reconocen.

Lo que no está decidido es lo demás, y son tres cosas distintas que el examen
mezcla a propósito:

1. **Cómo están repartidos los objetos dentro del bucket.** Decide a qué velocidad
   se pueden leer y escribir.
2. **En qué formato están escritos.** Decide cuántos bytes hay que mover para leer
   las ocho columnas que el modelo de fraude usa, de las sesenta que tiene el CSV.
3. **Cómo llegan al contenedor del training job.** Decide cuánto tiempo pasa el job
   en `Downloading` antes de empezar a entrenar, y cuánto disco hay que pagar.

Las tres se deciden juntas porque se condicionan: un formato pensado para leer
trozos sueltos de un archivo solo rinde si el mecanismo de entrega permite pedir
trozos sueltos, y ese mecanismo solo rinde si los objetos no son diminutos. Al
final de la nota las tres están fijadas para el caso de fraude, con la cifra que
justifica cada una.

---

## Claves, prefijos y el reparto que hace escalar a S3

### No hay carpetas

Un **objeto** de S3 es un par: una **clave** (una cadena de hasta 1.024 bytes) y su
contenido. La clave
`fraude/2026/transacciones/2026-01-03-0007.csv` no describe una jerarquía: es una
cadena, y las barras son caracteres normales. La consola dibuja carpetas porque
agrupa por la barra, pero dentro del servicio no existe ningún directorio que haya
que crear, ni que se pueda renombrar, ni que tenga permisos propios.

Un **prefijo** es cualquier cadena inicial de una clave. Es el concepto que importa
aquí por dos razones que el lector ya vio a medias: en la nota 1 los prefijos
recortaban permisos (la política `KananMLEngineerS3-dev` limita a `fraude/2026/*`,
y la clave de condición `s3:prefix` solo aplicaba a `ListBucket`), y en la nota 2 un
prefijo era lo que un canal de entrada enumeraba para decidir qué objetos bajar. La
tercera razón es de rendimiento y es nueva.

### El límite que se mide por prefijo

S3 reparte internamente las claves de un bucket entre **particiones**, y el
rendimiento se mide contra cada partición, no contra el bucket. Los números
publicados, por prefijo particionado:

| Operaciones | Peticiones por segundo |
| --- | --- |
| `PUT`, `COPY`, `POST`, `DELETE` | al menos 3.500 |
| `GET`, `HEAD` | 5.500 |

No hay límite en el número de prefijos, así que el techo real de un bucket lo pone
el reparto de las claves: diez prefijos que S3 haya particionado por separado dan
del orden de 55.000 lecturas por segundo. El reparto no es instantáneo. Cuando la
carga sube de golpe sobre un prefijo nuevo, S3 responde con `503 Slow Down`
mientras termina de particionar, y esos errores desaparecen solos cuando el reparto
se completa.

Lo que el lector hace con esto es elegir el orden de los componentes de la clave.
Para los 3.000 cajeros del pronóstico de efectivo, la diferencia entre

```
cajeros/2026-01-03/atm-0001.json
cajeros/2026-01-03/atm-0002.json
```

y

```
cajeros/atm-0001/2026-01-03.json
cajeros/atm-0002/2026-01-03.json
```

es que la primera concentra la escritura nocturna de los 3.000 cajeros en un único
prefijo por fecha, y la segunda la reparte entre 3.000 prefijos. Si la escritura es
un volcado diario que dura minutos, la primera basta y es más cómoda de leer por
rango de fechas. Si la escritura es continua y densa, la segunda es la que no choca
con el límite.

Para el histórico de fraude no hay problema de escritura: se escribe una vez. El
criterio que manda ahí es el de lectura por rango, y por eso el layout que esta nota
va a fijar agrupa por mes.

### Subir 500 GB sin que la red decida el resultado

Una petición `PutObject` sube un objeto entero en una sola operación. El
**multipart upload** lo trocea: la subida se abre, se envían partes numeradas de
forma independiente —y por tanto en paralelo, y reintentables una a una— y una
llamada final las une en un solo objeto. Los límites que hay que recordar:

| Concepto | Valor |
| --- | --- |
| Partes por subida | 10.000, numeradas de 1 a 10.000 |
| Tamaño de parte | 5 MiB a 5 GiB (la última puede ser menor) |
| Tamaño máximo por `PutObject` en una sola petición | 5 GiB |
| Tamaño máximo de objeto | 48,8 TiB |
| Umbral recomendado por AWS para trocear | a partir de 100 MB |

> **Servicio que cambió.** El tope de un objeto de S3 fue 5 TB durante años, y esa
> es la cifra que espera la guía del examen. En diciembre de 2025 AWS lo subió a
> 50 TB (48,8 TiB en la documentación). Si una pregunta ofrece 5 TB como respuesta,
> esa es la que el examen considera correcta.

El lector casi nunca llama a esta API. La CLI y boto3 la usan por debajo en cuanto
el archivo pasa de un umbral, y lo que se toca son los ajustes.

Este comando sube el histórico desde la laptop con la CLI configurada, con el perfil
`kanan-dev` de la nota 1. El bucket y el prefijo ya existen; la identidad es el
usuario `miguel.reyes`, cuya política permite escribir bajo `fraude/2026/*`.

```bash
aws configure set s3.max_concurrent_requests 30 --profile kanan-dev
aws configure set s3.multipart_threshold 64MB --profile kanan-dev
aws configure set s3.multipart_chunksize 64MB --profile kanan-dev

aws s3 sync ./historico-transacciones \
  s3://kanan-ml-dev-raw-us-east-1/fraude/2026/transacciones/ \
  --exclude "*" --include "*.csv" \
  --profile kanan-dev
```

Glosa:

- `aws configure set s3.<clave>` escribe una subsección `s3 =` dentro del bloque del
  perfil en `~/.aws/config`. Los valores por defecto son 10 peticiones concurrentes
  y 8 MB tanto de umbral como de tamaño de parte. Con archivos de cientos de MB,
  8 MB genera muchas partes pequeñas y muchas peticiones; 64 MB reduce el número de
  peticiones sin acercarse a las 10.000 partes.
- `sync` compara origen y destino por nombre, tamaño y fecha de modificación, y solo
  sube lo que falta o cambió. Es la diferencia práctica con `cp --recursive`: si la
  subida se corta a la mitad, `sync` reanuda; `cp` vuelve a empezar.
- `--exclude "*" --include "*.csv"` se evalúa en orden: primero se excluye todo,
  después se readmiten los CSV. Invertir el orden no filtra nada.

Resultado esperado: una línea `upload: ...` por archivo y, al final, el conjunto
replicado bajo el prefijo. Un `AccessDenied` aquí es de los de la nota 1 —tu
identidad contra S3—, y la coletilla del mensaje dice si falta un `Allow` o si hay
un `Deny` explícito.

El equivalente programático, cuando la subida es un paso dentro de un script de
Python y no una operación manual:

```python
import boto3
from boto3.s3.transfer import TransferConfig

session = boto3.Session(profile_name="kanan-dev", region_name="us-east-1")
s3 = session.client("s3")

config = TransferConfig(
    multipart_threshold=64 * 1024 * 1024,
    multipart_chunksize=64 * 1024 * 1024,
    max_concurrency=30,
)

s3.upload_file(
    Filename="historico-transacciones/2026-01.csv",
    Bucket="kanan-ml-dev-raw-us-east-1",
    Key="fraude/2026/transacciones/2026-01.csv",
    Config=config,
)
```

`TransferConfig` vive en `boto3.s3.transfer` y se pasa por el argumento `Config`,
que solo aceptan los métodos de transferencia (`upload_file`, `download_file`,
`upload_fileobj`, `download_fileobj`), no `put_object`. Sus valores por defecto son
los mismos que los de la CLI —8.388.608 bytes de umbral y de parte, concurrencia
10—, porque la CLI usa esta misma biblioteca.

Cuándo se usa cada una: la CLI para mover datos una vez, o desde un paso de
`aws s3 sync` dentro de un script de shell; boto3 cuando la subida es una parte de
una lógica mayor —subir solo las particiones que un job acaba de producir, decidir
la clave a partir del contenido, reintentar con una política propia— o cuando el
código ya es Python y meter un `subprocess` sería peor.

### Transfer Acceleration: cuándo el cuello de botella es la distancia

**S3 Transfer Acceleration** es un ajuste del bucket que habilita un nombre de host
alternativo, `<bucket>.s3-accelerate.amazonaws.com`. Las peticiones que van a ese
nombre entran por la ubicación de borde de CloudFront más cercana al cliente y
viajan hasta la región por la red interna de AWS en lugar de por la Internet
pública. El objeto acaba en el mismo bucket, en la misma región; lo único que cambia
es el camino.

```bash
aws s3api put-bucket-accelerate-configuration \
  --bucket kanan-ml-dev-raw-us-east-1 \
  --accelerate-configuration Status=Enabled \
  --profile kanan-dev

aws s3 cp ./2026-01.csv \
  s3://kanan-ml-dev-raw-us-east-1/fraude/2026/transacciones/2026-01.csv \
  --endpoint-url https://s3-accelerate.amazonaws.com \
  --profile kanan-dev
```

Detalles que cambian la respuesta en un examen:

- El nombre del bucket no puede contener puntos, porque el certificado comodín del
  endpoint acelerado no cubre nombres con punto. La convención de Kanan
  (`kanan-ml-dev-raw-us-east-1`) cumple.
- Activarlo tarda hasta 20 minutos en surtir efecto y **se cobra aparte** por GB
  transferido. AWS publica una herramienta de comparación de velocidad para decidir
  antes de pagar.
- Está disponible en un subconjunto de regiones, y solo funciona con peticiones de
  estilo *virtual-hosted*.

Sirve cuando hay una distancia real que recortar: sucursales o socios que suben
gigabytes desde otros continentes a un bucket central, o una subida que no consigue
llenar el ancho de banda disponible. **No** sirve —y es la trampa clásica— cuando el
origen está en la misma región que el bucket, cuando los objetos son pequeños (el
coste es por transferencia y la ganancia está en el trayecto largo de objetos
grandes), ni cuando el problema es de peticiones por segundo contra un prefijo, que
es el límite de la sección anterior y no se arregla cambiando de camino.

### Clases de almacenamiento, en una página

Cada objeto tiene una **clase de almacenamiento** que fija su precio por GB, su
coste de recuperación y si está disponible al instante. Lo que importa para ML es
corto:

| Clase | Acceso | Duración mínima facturable |
| --- | --- | --- |
| S3 Standard | inmediato | — |
| S3 Intelligent-Tiering | inmediato (los niveles de archivo opcionales, no) | — |
| S3 Standard-IA / One Zone-IA | inmediato, con cargo por recuperación | 30 días |
| S3 Glacier Instant Retrieval | inmediato, con cargo por recuperación | 90 días |
| S3 Glacier Flexible Retrieval | **requiere `RestoreObject`** | 90 días |
| S3 Glacier Deep Archive | **requiere `RestoreObject`** | 180 días |

La regla operativa: **un training job no puede leer un objeto archivado**. Si el
canal apunta a claves en Glacier Flexible Retrieval o Deep Archive, el job falla al
intentar leerlas; hay que restaurarlas antes, y la restauración tarda de 1–5 minutos
(nivel `Expedited`, solo en Flexible Retrieval) a 3–5 horas (`Standard`), 5–12 horas
(`Bulk`), o 9–12 y hasta 48 horas en Deep Archive. Por eso los datos de
entrenamiento activos viven en Standard o Intelligent-Tiering, y Glacier es el
destino del crudo de años anteriores que cumplimiento obliga a conservar y nadie
piensa entrenar. El cambio de clase no se hace objeto a objeto: se declara una
**regla de ciclo de vida** en el bucket (`put-bucket-lifecycle-configuration`) que
dice, por prefijo, a qué clase pasan los objetos al cumplir cierta antigüedad y
cuándo se borran.

---

## Cuando S3 no basta: volúmenes y sistemas de archivos

S3 es almacenamiento de **objetos**: se accede por una API HTTP, se escribe un
objeto entero de una vez y no hay nada parecido a abrir un archivo, posicionarse en
el byte 4.000.000 y leer 200 bytes. Los tres servicios de esta sección existen
porque hay cargas de ML que necesitan exactamente eso: un sistema de archivos con
semántica POSIX, escritura parcial y lecturas aleatorias baratas.

### EBS: el disco de la instancia

**Amazon EBS** (Elastic Block Store) ofrece volúmenes de bloques que se conectan a
una instancia EC2. Un volumen vive en **una** zona de disponibilidad, lo usa una
instancia a la vez (salvo `io2` con *multi-attach*) y para el sistema operativo es
un disco: se formatea y se monta.

El lector ya paga por EBS sin haberlo nombrado. El `VolumeSizeInGB` que la nota 2
declaraba obligatorio en `ProcessingResources.ClusterConfig` es el tamaño del
volumen EBS que SageMaker crea para las instancias del job y destruye al terminar.
Es ahí donde aterrizan `/opt/ml/input/data/<canal>`, `/opt/ml/model` y
`/opt/ml/output`.

Los tipos que el examen distingue:

| Tipo | Línea base | Máximo por volumen | Cuándo |
| --- | --- | --- | --- |
| `gp3` (SSD de propósito general) | 3.000 IOPS y 125 MiB/s, **independientes del tamaño** | 80.000 IOPS, 2.000 MiB/s (según instancia) | valor por defecto razonable para casi todo |
| `gp2` (generación anterior) | IOPS atados al tamaño | 16.000 IOPS | solo en volúmenes heredados |
| `io2` / io2 Block Express (**Provisioned IOPS**) | las IOPS que se provisionan | 256.000 IOPS, 4.000 MiB/s, durabilidad 99,999 % | latencia submilisegundo sostenida y consistente |
| `st1` (HDD optimizado para throughput) | throughput, no IOPS | 500 MiB/s, 500 IOPS | lectura secuencial masiva y barata |

La diferencia práctica entre `gp3` y `gp2` es que en `gp3` se provisionan IOPS y
throughput por separado del tamaño: un volumen de 100 GiB puede tener 3.000 IOPS sin
inflarlo a 1 TiB. *Provisioned IOPS* —el término que usa la guía del examen— es
`io1`/`io2`: se paga por un número garantizado de operaciones por segundo. En ML
casi nunca hace falta para un job de entrenamiento, porque el patrón es leer mucho y
secuencialmente, no muchas operaciones pequeñas con latencia crítica; sí aparece
cuando la base de datos que alimenta el pipeline —la RDS PostgreSQL de Kanan— es el
cuello de botella.

### EFS: el mismo directorio visto por varias máquinas

**Amazon EFS** es un sistema de archivos NFS gestionado. Es elástico (crece y
encoge solo), es regional —los datos se replican entre zonas de disponibilidad—, y
lo monta cualquier número de clientes a la vez sobre la misma ruta.

En ML aparece por una razón concreta: **los datos ya están ahí**. Un equipo que
etiqueta imágenes desde varias instancias, o un grupo de notebooks que comparte un
directorio de trabajo, escribe en EFS de forma natural, y montar ese mismo sistema
de archivos en el job de entrenamiento evita copiar nada a S3. Sus modos de
throughput —`Elastic`, que escala solo y es el recomendado; `Provisioned`, que fija
una cifra; `Bursting`, que la ata al tamaño— importan porque la lectura de un
dataset durante el entrenamiento puede agotar los créditos de `Bursting` y dejar el
job leyendo a la línea base.

Lo que EFS no es: almacenamiento de alto rendimiento para entrenar. Su latencia por
operación (del orden de un milisegundo en lectura) es excelente para un sistema de
archivos de red y mediocre frente a lo que sigue.

### FSx for Lustre: el sistema de archivos que se apoya en S3

**Amazon FSx for Lustre** es un sistema de archivos POSIX de alto rendimiento
—Lustre es el sistema de archivos paralelo estándar en cómputo de alto
rendimiento—, y su rasgo decisivo para ML es que se puede **vincular a un bucket de
S3** mediante una *data repository association* (DRA). Vinculado, el contenido del
prefijo de S3 aparece como archivos dentro del sistema de archivos; el contenido
real se carga desde S3 la primera vez que se lee.

El rendimiento se provisiona por unidad de almacenamiento, en MB/s por TiB:

| Tipo de despliegue | Throughput base | Ráfaga |
| --- | --- | --- |
| `SCRATCH_2` | 200 MB/s por TiB | 1.300 MB/s por TiB |
| `PERSISTENT-125` | 320 MB/s por TiB | 1.300 MB/s por TiB |
| `PERSISTENT-250` | 640 MB/s por TiB | 1.300 MB/s por TiB |
| `PERSISTENT-500` | 1.300 MB/s por TiB | — |
| `PERSISTENT-1000` | 2.600 MB/s por TiB | — |

Además, en los sistemas `Persistent 2` con almacenamiento SSD se provisionan
**IOPS de metadatos** aparte (1.500, 3.000, 6.000, 12.000 y múltiplos de 12.000).
Ese número es el que decide el rendimiento cuando el dataset son millones de
archivos pequeños, porque entonces el trabajo no es mover bytes sino abrir y cerrar
archivos.

Crear el sistema de archivos vinculado al prefijo curado, desde la laptop:

```bash
aws fsx create-file-system \
  --file-system-type LUSTRE \
  --storage-capacity 1200 \
  --storage-type SSD \
  --subnet-ids subnet-0ab1c2d3e4f56789a \
  --security-group-ids sg-0123456789abcdef0 \
  --lustre-configuration '{
      "DeploymentType": "PERSISTENT_2",
      "PerUnitStorageThroughput": 250,
      "DataCompressionType": "LZ4"
  }' \
  --kms-key-id arn:aws:kms:us-east-1:111111111111:key/3f1c9e02-kanan-ml-dev \
  --profile kanan-dev

aws fsx create-data-repository-association \
  --file-system-id fs-0123456789abcdef0 \
  --file-system-path /ns1 \
  --data-repository-path s3://kanan-ml-dev-curated-us-east-1/fraude/2026/ \
  --batch-import-meta-data-on-create \
  --profile kanan-dev
```

Glosa:

- `--storage-capacity 1200` son GiB: 1,2 TiB es el escalón inicial de los sistemas
  SSD. Con `PerUnitStorageThroughput` 250, eso da 640 MB/s por TiB × 1,2 TiB de
  throughput base.
- `--subnet-ids` y `--security-group-ids`: un sistema de archivos vive dentro de una
  red privada, y eso arrastra una condición sobre quien quiera leerlo.
- `--file-system-path /ns1` es la ruta dentro del sistema de archivos donde aparece
  el prefijo de S3; `--data-repository-path` es el prefijo. La opción
  `--batch-import-meta-data-on-create` importa los metadatos de los objetos ya
  existentes al crear la asociación, para que los archivos se vean de inmediato.
- `DataCompressionType: LZ4` comprime en reposo dentro del sistema de archivos.

`aws fsx describe-file-systems` devuelve, además del `FileSystemId`, un `MountName`
(una cadena corta como `1234abcd`) que hace falta para construir la ruta del canal.

> **Caja negra.** Todo lo que lea un sistema de archivos —incluido un job de
> SageMaker— tiene que correr dentro de la misma red privada donde vive. En la
> petición de creación del sistema de archivos eso son `--subnet-ids` y
> `--security-group-ids`; en la petición del job es
> `VpcConfig = {"Subnets": [...], "SecurityGroupIds": [...]}`, con la misma subred
> y un grupo de seguridad que permita el tráfico. Copia los identificadores tal
> cual; qué son, cómo se eligen y qué más hay que abrir es el tema de
> [[23 - Red]].

### FSx for NetApp ONTAP, y por qué está en el temario

**Amazon FSx for NetApp ONTAP** es un sistema de archivos gestionado que corre el
software ONTAP de NetApp y expone el mismo volumen por NFS, SMB e iSCSI a la vez,
con instantáneas, clonado y replicación desde y hacia cabinas ONTAP locales.

Su papel en ML es de puente: una empresa cuyos datos viven en NetApp en su centro de
datos replica el volumen a ONTAP en AWS y trabaja sobre él sin reescribir el
almacenamiento. Para el examen basta reconocerlo por esa señal —multiprotocolo,
procedencia NetApp, replicación desde on-premises—; cuando una pregunta describe un
dataset que ya está en S3, ONTAP nunca es la respuesta.

### Qué distingue a los cinco

| | S3 | EBS | EFS | FSx for Lustre | FSx for ONTAP |
| --- | --- | --- | --- | --- | --- |
| Tipo | objetos | bloques | archivos (NFS) | archivos (POSIX/Lustre) | archivos (NFS/SMB/iSCSI) |
| Alcance | regional | una AZ, una instancia | regional, muchos clientes | una AZ (o multi-AZ según tipo), muchos clientes | multi-AZ, muchos clientes |
| Acceso aleatorio barato | no | sí | sí | sí | sí |
| Se vincula a S3 | — | no | no | **sí** | no |
| Señal típica en ML | datos de entrenamiento, artefactos | disco del job, disco de la instancia | datos ya compartidos entre notebooks o etiquetado | millones de archivos pequeños, muchas épocas, entrenamientos repetidos | datos que vienen de NetApp on-premises |

---

## Fila contra columna: qué formato guardar

### Lo que significa que un formato esté «validado»

La guía del examen distingue formatos **validados** y **no validados**. La
distinción es esta: un formato validado lleva dentro del propio archivo la
descripción de sus columnas y sus tipos —el **esquema**—, de modo que quien escribe
no puede meter una cadena donde va un entero sin que la biblioteca de escritura lo
rechace, y quien lee sabe qué tipo tiene cada campo sin adivinarlo. Parquet, ORC y
Avro son de esta clase.

Un formato no validado es texto sin contrato: CSV y JSON crudo. Nada impide que la
fila 4.000.001 tenga una columna de más, que `monto` traiga `N/A` en lugar de un
número, o que un campo con comas dentro de comillas se parta mal. El error no
aparece al escribir sino al leer, normalmente dentro del contenedor de
entrenamiento, y con el mensaje del lector de turno, que rara vez dice qué fila fue.

Esa es toda la diferencia conceptual. Lo demás es cómo se colocan los bytes.

### Todos los valores de una columna, juntos

Un formato **orientado a filas** guarda un registro completo y después el
siguiente. Es lo natural cuando se escribe: una transacción llega, se añade. CSV,
JSON Lines y Avro son de fila.

Un formato **columnar** guarda todos los valores de la primera columna, después
todos los de la segunda, y así. Eso tiene dos consecuencias que dominan la decisión:

1. **Se puede leer una columna sin leer las demás.** Cada bloque de columna está en
   un rango de bytes conocido, y el lector pide solo ese rango.
2. **Comprime mucho mejor.** Los valores de una misma columna son del mismo tipo y
   se parecen entre sí, así que la compresión encuentra más redundancia que en una
   fila que mezcla una fecha, un importe y un identificador.

**Parquet** organiza el archivo en *row groups* (grupos de filas, típicamente de
decenas o cientos de miles) y dentro de cada uno, un bloque por columna, con
estadísticas por bloque: mínimo, máximo y número de nulos. Un lector que busca
`monto > 10000` puede saltarse un bloque entero cuyo máximo es 800 sin
descomprimirlo. **ORC** hace lo mismo con otra nomenclatura (*stripes* en lugar de
row groups) y viene del mundo Hive; en AWS se lo encuentra sobre todo en datos que
ya venían de un clúster Hadoop. **Avro** es de fila, lleva su esquema en JSON dentro
del archivo y está pensado para evolución de esquema: un consumidor escrito para la
versión vieja puede leer datos de la versión nueva. Por eso Avro es un formato de
**ingesta** y Parquet u ORC de **consulta**.

### El número que decide

Medido sobre una muestra sintética con el esquema del histórico de fraude
—1.000.000 de filas, 24 columnas, la mezcla de identificadores, importes,
categóricas y agregados que tiene la tabla real— con pandas 3.0.2 y pyarrow 25.0.1:

| Archivo | Tamaño |
| --- | --- |
| CSV | 145,3 MB |
| CSV comprimido con gzip | 54,0 MB |
| Parquet con Snappy (row groups de 200.000) | 69,2 MB |
| Parquet con zstd | 55,4 MB |

Y el tiempo de leer las **ocho** columnas que el modelo de fraude usa, contra leer
las veinticuatro:

| Lectura | Tiempo |
| --- | --- |
| CSV, 8 columnas (`usecols`) | 0,90 s |
| Parquet, 8 columnas | 0,06 s |
| Parquet, 24 columnas | 0,24 s |

`usecols` en CSV no ahorra E/S: el archivo se lee entero y las columnas sobrantes se
descartan después de parsearlas. En Parquet las ocho columnas ocupan 14,6 MB de los
69,2 del archivo, y son los únicos bytes que se transfieren.

Extrapolado al histórico: 500 GB de CSV son unos 240 GB en Parquet con Snappy, y una
lectura de las ocho columnas del modelo mueve del orden de 50 GB. La misma lectura
sobre el CSV mueve 500 GB. Esa es la decisión de formato, y no depende de ninguna
preferencia estética.

### Compresión: la que se puede partir y la que no

Un archivo `.csv.gz` de 54 MB parece una ganga hasta que hay que leerlo en paralelo.
gzip no es **divisible**: para descomprimir el byte del medio hay que haber
descomprimido todo lo anterior, así que un único proceso lee el archivo entero y no
hay forma de repartirlo entre instancias. Un CSV sin comprimir sí se puede partir
por líneas.

En Parquet y ORC el problema no existe: la compresión se aplica por bloque de
columna dentro de cada row group, y los row groups son independientes. Por eso
Snappy (rápido, compresión media) y zstd (más lento, compresión mejor) son las
opciones habituales dentro de Parquet, y la elección entre ellas es tiempo de CPU
contra bytes movidos, no divisibilidad.

### La conversión, con lo que ya sabemos lanzar

Convertir 500 GB es un job de Processing: exactamente la pieza de la nota 2, con la
imagen de scikit-learn ya verificada, y con `ShardedByS3Key` para que cada instancia
convierta su parte. La versión a escala con un motor distribuido es
[[05 - Catálogo y transformación a escala]].

El script, que vive en
`s3://kanan-ml-dev-artifacts-us-east-1/code/csv_a_parquet.py`:

```python
import os
import subprocess
import sys

try:
    import pyarrow  # noqa: F401
except ImportError:
    subprocess.check_call([sys.executable, "-m", "pip", "install", "-q", "pyarrow"])

import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq

ENTRADA = "/opt/ml/processing/input/transacciones"
SALIDA = "/opt/ml/processing/output/parquet"

ESQUEMA = pa.schema([
    ("transaccion_id", pa.string()),
    ("cliente_id", pa.int64()),
    ("ts", pa.timestamp("s")),
    ("monto", pa.float64()),
    ("moneda", pa.dictionary(pa.int32(), pa.string())),
    ("canal", pa.dictionary(pa.int32(), pa.string())),
    ("tx_24h", pa.int32()),
    ("monto_24h", pa.float64()),
    ("score_riesgo_comercio", pa.float32()),
    ("intentos_fallidos_7d", pa.int32()),
    ("cambio_pais_24h", pa.int8()),
    ("fraude", pa.int8()),
])

os.makedirs(SALIDA, exist_ok=True)

for nombre in sorted(os.listdir(ENTRADA)):
    if not nombre.endswith(".csv"):
        continue
    mes = nombre[:7]
    destino = os.path.join(SALIDA, f"mes={mes}")
    os.makedirs(destino, exist_ok=True)

    escritor = None
    for i, trozo in enumerate(pd.read_csv(os.path.join(ENTRADA, nombre),
                                          chunksize=500_000,
                                          parse_dates=["ts"])):
        tabla = pa.Table.from_pandas(trozo[ESQUEMA.names], schema=ESQUEMA,
                                     preserve_index=False)
        if escritor is None:
            escritor = pq.ParquetWriter(
                os.path.join(destino, f"{nombre[:-4]}.parquet"),
                ESQUEMA, compression="snappy")
        escritor.write_table(tabla, row_group_size=200_000)
    if escritor is not None:
        escritor.close()
```

Glosa de lo que no es obvio:

- El bloque `try/except ImportError` está porque **no verifiqué** que la imagen
  `sagemaker-scikit-learn:1.2-1-cpu-py3` traiga pyarrow. Con la línea, el script
  funciona traiga o no traiga; sin ella, el job fallaría con un `ModuleNotFoundError`
  en `FailureReason` y habría que repetirlo.
- `ESQUEMA` declarado a mano es lo que convierte un formato no validado en uno
  validado: si el CSV trae `monto` con un valor no numérico, la conversión falla
  aquí, en un job que se puede repetir, y no dentro del entrenamiento.
- `pa.dictionary(pa.int32(), pa.string())` guarda `moneda` y `canal` como enteros
  con un diccionario de valores, que es la representación que hace que una columna
  categórica ocupe casi nada.
- `chunksize=500_000` y `ParquetWriter` incremental evitan cargar un CSV de decenas
  de GB en memoria: el volumen EBS del job tiene el tamaño que le pusimos, y la RAM
  de la instancia también es finita.
- `row_group_size=200_000` fija el grano de las estadísticas y del salto de bloques.
  Row groups muy pequeños hinchan los metadatos; muy grandes hacen que saltarse uno
  ahorre menos.
- La salida va a `mes=<AAAA-MM>/`, que es el layout de prefijos de la segunda
  sección: agrupa por el criterio con el que se lee.

El lanzamiento, desde un notebook de Studio con el rol
`KananSageMakerExecutionRole-dev`, reutiliza la forma de `create_processing_job` de
la nota 2 y solo cambia el reparto:

```python
import boto3, time

sm = boto3.Session(profile_name="kanan-dev", region_name="us-east-1").client("sagemaker")
marca = time.strftime("%Y%m%d-%H%M%S")

sm.create_processing_job(
    ProcessingJobName=f"kanan-csv-a-parquet-{marca}",
    RoleArn="arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev",
    AppSpecification={
        "ImageUri": "683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-scikit-learn:1.2-1-cpu-py3",
        "ContainerEntrypoint": ["python3", "/opt/ml/processing/input/code/csv_a_parquet.py"],
    },
    ProcessingResources={"ClusterConfig": {
        "InstanceType": "ml.m5.4xlarge",
        "InstanceCount": 10,
        "VolumeSizeInGB": 200,
        "VolumeKmsKeyId": "arn:aws:kms:us-east-1:111111111111:key/3f1c9e02-kanan-ml-dev",
    }},
    ProcessingInputs=[
        {"InputName": "transacciones", "S3Input": {
            "S3Uri": "s3://kanan-ml-dev-raw-us-east-1/fraude/2026/transacciones/",
            "LocalPath": "/opt/ml/processing/input/transacciones",
            "S3DataType": "S3Prefix", "S3InputMode": "File",
            "S3DataDistributionType": "ShardedByS3Key"}},
        {"InputName": "code", "S3Input": {
            "S3Uri": "s3://kanan-ml-dev-artifacts-us-east-1/code/csv_a_parquet.py",
            "LocalPath": "/opt/ml/processing/input/code",
            "S3DataType": "S3Prefix", "S3InputMode": "File",
            "S3DataDistributionType": "FullyReplicated"}},
    ],
    ProcessingOutputConfig={
        "KmsKeyId": "arn:aws:kms:us-east-1:111111111111:key/3f1c9e02-kanan-ml-dev",
        "Outputs": [{"OutputName": "parquet", "S3Output": {
            "S3Uri": "s3://kanan-ml-dev-curated-us-east-1/fraude/2026/parquet/",
            "LocalPath": "/opt/ml/processing/output/parquet",
            "S3UploadMode": "EndOfJob"}}],
    },
    StoppingCondition={"MaxRuntimeInSeconds": 7200},
)
```

Las dos cosas que hacen que esto escale y no son evidentes: `ShardedByS3Key` en la
entrada de datos reparte los objetos entre las diez instancias —cada una convierte
una décima parte—, mientras que el código va `FullyReplicated` porque las diez lo
necesitan entero; y `VolumeSizeInGB: 200` tiene que caber la décima parte del CSV
más el Parquet que produce, no los 500 GB.

Resultado esperado: el job pasa por `InProgress` y termina en `Completed`, y bajo
`fraude/2026/parquet/mes=2026-01/` aparecen los archivos `.parquet`. Si el volumen
se queda corto, el job termina en `Failed` y el `FailureReason` habla de falta de
espacio en disco dentro del contenedor.

### RecordIO-protobuf, y por qué existe

**RecordIO-protobuf** es un formato binario con el tipo de contenido
`application/x-recordio-protobuf`: cada observación se serializa como un mensaje
protobuf con sus rasgos en un tensor de flotantes de 4 bytes, y los mensajes se
encadenan con un envoltorio RecordIO que marca dónde empieza y acaba cada uno.

Existe por una razón concreta: los algoritmos integrados de SageMaker
—Linear Learner, K-Means, PCA, Factorization Machines, k-NN, NTM, Random Cut
Forest, LDA, Seq2Seq— lo aceptan como entrada, y es el único formato de esa lista
que permite leer los registros **de uno en uno mientras llegan**, sin tener el
archivo completo. Un CSV también se puede leer por líneas, pero hay que parsear
texto por cada valor; aquí los bytes ya están en la representación que el algoritmo
consume. Eso es lo que lo hace el formato natural para los mecanismos de entrega
que empujan registros al contenedor según llegan, en vez de dejar archivos en el
disco. (XGBoost, que es el algoritmo del caso de
fraude, no está en esa lista: acepta `text/csv` y `text/libsvm`.)

Escribirlo desde un array de NumPy, con el SDK v3:

```python
import io
import boto3
import numpy as np
from sagemaker.core.serializers.utils import write_numpy_to_dense_tensor

X = np.load("rasgos.npy").astype("float32")      # (n, 8)
y = np.load("etiquetas.npy").astype("float32")   # (n,)

buf = io.BytesIO()
write_numpy_to_dense_tensor(buf, X, y)
buf.seek(0)

boto3.client("s3").upload_fileobj(
    buf,
    "kanan-ml-dev-curated-us-east-1",
    "fraude/2026/recordio/train/parte-0001.protobuf",
)
```

`write_numpy_to_dense_tensor(file, array, labels=None)` escribe la matriz densa y,
si se le pasa, el vector de etiquetas en el campo `label` de cada registro. Vive en
`sagemaker.core.serializers.utils`; en el SDK v2 el mismo par de funciones
—`write_numpy_to_dense_tensor` y `write_spmatrix_to_sparse_tensor`, esta última para
matrices dispersas de SciPy— estaba en `sagemaker.amazon.common`, módulo que el
paquete `sagemaker` 3.x ya no contiene.

### La señal que identifica a cada formato

| Formato | Orientación | Esquema | Señal de que es la respuesta |
| --- | --- | --- | --- |
| CSV | fila | no | el algoritmo integrado solo acepta eso; datos pequeños; intercambio |
| JSON Lines | fila | no | registros anidados o de forma variable; entrada de BlazingText y DeepAR |
| Avro | fila | sí | ingesta continua con esquema que evoluciona |
| Parquet | columna | sí | consultas repetidas sobre un subconjunto de columnas; dataset grande |
| ORC | columna | sí | lo mismo que Parquet, cuando el origen ya es Hive |
| RecordIO-protobuf | registro binario | tipos fijos | algoritmo integrado que lo acepta, con lectura por streaming |

---

## Cómo llegan los datos al contenedor: los modos de entrada

En la nota 2, `AlgorithmSpecification.TrainingInputMode` era un campo obligatorio de
`CreateTrainingJob` que se rellenaba con `"File"` sin más explicación. Esta sección
es esa explicación. El campo elige el mecanismo por el que los objetos de S3 se
vuelven accesibles dentro del contenedor, y cada canal puede además sobreescribirlo
con su propio `InputMode`: es útil cuando el canal `train` pesa 400 GB y el canal
`validation` pesa 2 GB y conviene tratarlos distinto.

### File: se baja todo antes de empezar

SageMaker copia todos los objetos del canal al volumen EBS de cada instancia, los
deja en `/opt/ml/input/data/<canal>` y **después** arranca el contenedor. El
programa de entrenamiento ve archivos locales normales.

Lo que eso implica:

- El `VolumeSizeInGB` tiene que caber los datos del canal —replicados enteros en
  cada instancia si es `FullyReplicated`, o su parte si es `ShardedByS3Key`— más el
  modelo y las salidas. Si no caben, el job muere descargando.
- Hay un tiempo muerto medible. La documentación de AWS da como referencia unos
  **5 minutos para 50 GB** repartidos en fragmentos de 100 MB. Ese tiempo aparece en
  el `SecondaryStatus` `Downloading` de la nota 2, y se factura.
- Si el job se reinicia —una instancia interrumpida, un reintento—, se vuelve a
  descargar todo.

Es la opción correcta cuando el dataset es pequeño frente a la duración del
entrenamiento: bajar 20 GB para entrenar cuatro horas es ruido; bajar 400 GB para
entrenar veinte minutos es absurdo.

### FastFile: se monta S3 y se lee lo que haga falta

En modo **FastFile**, SageMaker expone el prefijo de S3 como un sistema de archivos
en la misma ruta `/opt/ml/input/data/<canal>`, pero no descarga nada por
adelantado: al arrancar solo enumera los objetos, y cada lectura que hace el
programa se traduce en una petición a S3 por el rango de bytes que pidió.

Consecuencias:

- El arranque es casi independiente del tamaño del dataset: lo que se hace al
  principio es listar metadatos, a un ritmo del orden de 5.500 objetos por segundo.
- El disco de la instancia no necesita caber los datos.
- El programa de entrenamiento no cambia: sigue abriendo archivos. Esa es la
  diferencia clave con el modo siguiente.
- Rinde bien con archivos grandes leídos de principio a fin —AWS sitúa el punto
  dulce por encima de 150 MB por archivo— y se degrada cuando el dataset son
  millones de archivos diminutos, porque entonces cada archivo cuesta al menos una
  petición HTTP.

Es el modo por defecto razonable hoy para datasets grandes en S3, y el que combina
bien con Parquet: leer ocho columnas de un archivo Parquet son unas pocas lecturas
por rangos de bytes, que es exactamente lo que FastFile hace barato.

### Pipe: los bytes llegan por una tubería

En modo **Pipe**, SageMaker no crea archivos: crea una tubería con nombre (un FIFO
de Unix) por canal y por época, en
`/opt/ml/input/data/<canal>_<número de época>`, y va empujando los bytes de los
objetos de S3 por ahí mientras el programa lee. El programa **no** puede
posicionarse ni releer: es un flujo secuencial de un solo sentido.

Por eso el algoritmo tiene que estar escrito para Pipe, y por eso los formatos que
lo acompañan son los que se pueden consumir registro a registro:
RecordIO-protobuf, texto por líneas, o un archivo envuelto en RecordIO. Tres campos
del canal existen casi solo para este modo:

- `CompressionType: "Gzip"` —se usa únicamente en Pipe; en File se deja sin poner.
- `RecordWrapperType: "RecordIO"` —envuelve cada objeto de S3 en un registro
  RecordIO cuando los datos están en crudo y el algoritmo espera RecordIO.
- `S3DataType: "AugmentedManifestFile"` —un archivo JSON Lines donde cada línea
  lleva la ruta de un objeto y sus atributos (por ejemplo, la etiqueta). Solo se
  puede usar con Pipe.

A cambio, el arranque es inmediato y el disco no necesita caber los datos.

**Estado actual.** Pipe sigue soportado y el examen lo evalúa, pero FastFile cubre
hoy casi todos sus casos con una interfaz que no obliga a reescribir el programa de
entrenamiento. Una pregunta que diga «arranque rápido y sin espacio en disco» admite
las dos; si además dice «sin cambiar el código de entrenamiento», la respuesta es
FastFile, y si dice «el algoritmo integrado consume RecordIO-protobuf por
streaming», es Pipe.

### EFS y FSx for Lustre como origen del canal

Un canal puede no venir de S3. En lugar de `S3DataSource`, el canal lleva un
`FileSystemDataSource` con cuatro campos, todos obligatorios:

| Campo | Valores |
| --- | --- |
| `FileSystemId` | `fs-` seguido de al menos 8 hexadecimales |
| `FileSystemType` | `EFS` o `FSxLustre` |
| `DirectoryPath` | ruta absoluta dentro del sistema de archivos |
| `FileSystemAccessMode` | `ro` o `rw` |

En FSx for Lustre, `DirectoryPath` empieza por el `MountName` que devolvió
`describe-file-systems`: con `MountName` `1234abcd` y la asociación creada sobre
`/ns1`, la ruta del canal es `/1234abcd/ns1/train`.

Y arrastra la condición de la caja negra de la sección anterior: la petición del
job tiene que llevar `VpcConfig` con la subred del sistema de archivos y un grupo de
seguridad que permita el tráfico. Es el único modo de entrada que obliga a eso.

Cuándo compensa: cuando el mismo dataset se lee muchas veces —barridos de
hiperparámetros, decenas de épocas sobre millones de archivos pequeños— y el coste
de tener el sistema de archivos encendido se reparte entre todos esos jobs. Para un
entrenamiento único sobre un dataset que ya está en S3, no compensa: se paga
almacenamiento duplicado y complejidad de red a cambio de nada.

### Repartir y barajar

`S3DataDistributionType` ya se vio en la nota 2: `FullyReplicated` copia el conjunto
a cada instancia, `ShardedByS3Key` reparte los objetos. Con `ShardedByS3Key` el
reparto es **por objeto**, así que un canal de cuatro objetos y ocho instancias deja
cuatro instancias sin datos, encendidas y facturando. El número de objetos tiene que
ser bastante mayor que el número de instancias.

`ShuffleConfig` es el campo hermano: baraja el orden en que se presentan los
objetos, con una semilla. Con `S3Prefix` baraja el resultado de la enumeración del
prefijo; con un manifiesto, el orden de las entradas. En Pipe el barajado se rehace
al empezar cada época, y combinado con `ShardedByS3Key` en un job multi-instancia
reparte distinto en cada época, de modo que un mismo objeto no cae siempre en el
mismo nodo.

### Los cuatro, en una petición

Esto se lanza desde un notebook de Studio con el rol
`KananSageMakerExecutionRole-dev`; el bucket curado y el modelo ya existen. Solo
cambia el bloque `InputDataConfig` y el `TrainingInputMode`:

```python
import boto3, time

sm = boto3.Session(profile_name="kanan-dev", region_name="us-east-1").client("sagemaker")

ROL = "arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev"
IMAGEN = "683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.7-1"
KMS = "arn:aws:kms:us-east-1:111111111111:key/3f1c9e02-kanan-ml-dev"

canal_file = {
    "ChannelName": "train",
    "ContentType": "text/csv",
    "DataSource": {"S3DataSource": {
        "S3DataType": "S3Prefix",
        "S3Uri": "s3://kanan-ml-dev-curated-us-east-1/fraude/2026/train/",
        "S3DataDistributionType": "ShardedByS3Key"}},
}

canal_fastfile = {**canal_file, "InputMode": "FastFile"}

canal_pipe = {
    "ChannelName": "train",
    "ContentType": "application/x-recordio-protobuf",
    "CompressionType": "None",
    "InputMode": "Pipe",
    "ShuffleConfig": {"Seed": 20260920},
    "DataSource": {"S3DataSource": {
        "S3DataType": "S3Prefix",
        "S3Uri": "s3://kanan-ml-dev-curated-us-east-1/fraude/2026/recordio/train/",
        "S3DataDistributionType": "ShardedByS3Key"}},
}

canal_fsx = {
    "ChannelName": "train",
    "ContentType": "text/csv",
    "DataSource": {"FileSystemDataSource": {
        "FileSystemId": "fs-0123456789abcdef0",
        "FileSystemType": "FSxLustre",
        "DirectoryPath": "/1234abcd/ns1/train",
        "FileSystemAccessMode": "ro"}},
}

sm.create_training_job(
    TrainingJobName=f"kanan-fraude-xgb-{time.strftime('%Y%m%d-%H%M%S')}",
    RoleArn=ROL,
    AlgorithmSpecification={"TrainingImage": IMAGEN, "TrainingInputMode": "FastFile"},
    InputDataConfig=[canal_fastfile],
    OutputDataConfig={
        "S3OutputPath": "s3://kanan-ml-dev-artifacts-us-east-1/fraude/2026/modelos/",
        "KmsKeyId": KMS},
    ResourceConfig={"InstanceType": "ml.m5.4xlarge", "InstanceCount": 4,
                    "VolumeSizeInGB": 50, "VolumeKmsKeyId": KMS},
    HyperParameters={"objective": "binary:logistic", "num_round": "300",
                     "scale_pos_weight": "500"},
    StoppingCondition={"MaxRuntimeInSeconds": 10800},
)
```

Glosa de las diferencias:

- `canal_file` no lleva `InputMode`: hereda el `TrainingInputMode` de
  `AlgorithmSpecification`. `canal_fastfile` lo sobreescribe. Esa es la relación
  entre los dos campos.
- Con `FastFile`, `VolumeSizeInGB: 50` ya no tiene que caber el dataset, solo el
  modelo y lo que el contenedor escriba. Con `File` y 240 GB de datos repartidos
  entre cuatro instancias haría falta más de 60 GB por instancia.
- `canal_pipe` apunta al prefijo de RecordIO-protobuf, no al de CSV, y añade
  `ShuffleConfig`, que en Pipe se aplica en cada época.
- `canal_fsx` no menciona S3 en ningún sitio, y exige añadir a la llamada el
  `VpcConfig` de la caja negra.

La misma elección con el SDK v3, cuando el código vive en un script y no en una
petición cruda:

```python
from sagemaker.core.shapes import Channel, DataSource, S3DataSource
from sagemaker.train import ModelTrainer
from sagemaker.core.training.configs import Compute

canal = Channel(
    channel_name="train",
    content_type="text/csv",
    input_mode="FastFile",
    data_source=DataSource(s3_data_source=S3DataSource(
        s3_data_type="S3Prefix",
        s3_uri="s3://kanan-ml-dev-curated-us-east-1/fraude/2026/train/",
        s3_data_distribution_type="ShardedByS3Key")),
)

entrenador = ModelTrainer(
    training_image=IMAGEN,
    role=ROL,
    base_job_name="kanan-fraude-xgb",
    compute=Compute(instance_type="ml.m5.4xlarge", instance_count=4, volume_size_in_gb=50),
)
entrenador.train(input_data_config=[canal], wait=True, logs=True)
```

Un cambio respecto a la nota 2: en el SDK 3.22.1, `sagemaker.train.configs` es un
alias que reexporta `sagemaker.core.training.configs` y emite un
`DeprecationWarning` al importarse. El módulo con el nombre largo es el que
sobrevive.

### El modo que corresponde a cada situación

| | File | FastFile | Pipe | EFS / FSx for Lustre |
| --- | --- | --- | --- | --- |
| Tiempo de arranque | proporcional al dataset (≈5 min por 50 GB) | casi nulo (enumera metadatos) | casi nulo | montaje rápido; FSx tarda en crearse la primera vez |
| Disco del job | tiene que caber los datos | no | no | no |
| Acceso aleatorio | sí (archivos locales) | sí, con coste por petición | **no**, solo secuencial | sí |
| Cambios en el código | ninguno | ninguno | **sí**, hay que leer del FIFO | ninguno |
| Red privada obligatoria | no | no | no | **sí** |
| Coste añadido | disco EBS del job | peticiones a S3 | peticiones a S3 | el sistema de archivos, encendido |
| Señal típica | dataset pequeño, muchas épocas locales | dataset grande en S3, archivos grandes | algoritmo integrado con RecordIO y streaming | millones de archivos pequeños, jobs repetidos sobre el mismo dataset |

### Lo que sale mal al cargar los datos

- **El job se queda descargando y muere.** `SecondaryStatus` llega a `Downloading` y
  el job termina en `Failed`; el `FailureReason` habla de falta de espacio. Causa:
  `VolumeSizeInGB` menor que el canal en modo File. Arreglo: subir el volumen,
  pasar a `ShardedByS3Key`, o cambiar a FastFile.
- **`ValidationException` en la llamada.** Llega inmediata, no en `FailureReason`.
  Causas típicas en esta nota: un canal con `FileSystemDataSource` sin `VpcConfig`;
  `CompressionType: "Gzip"` en un canal que no es Pipe; `AugmentedManifestFile` con
  `InputMode` distinto de Pipe.
- **`AccessDenied` en `FailureReason`.** Es el segundo de los dos `AccessDenied` de
  la nota 2: el rol de ejecución contra S3. Con un canal de FSx, el equivalente es
  que al rol le falten permisos sobre el sistema de archivos.
- **El job no arranca nunca y se queda en `Starting`.** Con canales de sistema de
  archivos, apunta a la red: subred distinta a la del sistema de archivos, grupo de
  seguridad que no deja pasar el tráfico, o falta de salida hacia S3 desde esa red.
- **El entrenamiento va lentísimo en FastFile.** Millones de objetos pequeños: cada
  archivo cuesta peticiones. Arreglo: consolidar en archivos grandes, o pasar a FSx
  for Lustre.

---

## El histórico no empieza en S3

Los 500 GB de CSV llegaron a S3 porque alguien los puso ahí. Las dos fuentes vivas
de Kanan —la RDS PostgreSQL con el historial y la tabla de DynamoDB con el perfil de
cliente— siguen fuera, y hay que sacar datos de ellas cada vez que se rehace el
conjunto de entrenamiento.

### RDS: exportar la instantánea, no consultar la base

La opción intuitiva es conectarse a la base y volcar el resultado de una consulta.
Funciona, y tiene dos problemas: la consulta compite por CPU y E/S con las
transacciones de producción, y una tabla que cambia mientras se lee da un resultado
que no corresponde a ningún instante.

RDS ofrece la alternativa: **exportar una instantánea a S3**. El servicio restaura
la instantánea en infraestructura aparte, extrae los datos y los escribe en S3 en
**Apache Parquet** comprimido. La instancia en producción no se entera.

```bash
aws rds start-export-task \
  --export-task-identifier kanan-historial-2026-09 \
  --source-arn arn:aws:rds:us-east-1:111111111111:snapshot:kanan-historial-2026-09-19 \
  --s3-bucket-name kanan-ml-dev-raw-us-east-1 \
  --s3-prefix fraude/2026/historial-rds/ \
  --iam-role-arn arn:aws:iam::111111111111:role/KananRdsExportRole-dev \
  --kms-key-id arn:aws:kms:us-east-1:111111111111:key/3f1c9e02-kanan-ml-dev \
  --export-only kanan.public.transacciones kanan.public.contracargos \
  --profile kanan-dev
```

Glosa:

- `--source-arn` es el ARN de una **instantánea**, no de la instancia. Vale una
  manual, una automática o una de AWS Backup.
- `--iam-role-arn` es un rol de ejecución en el sentido de la nota 1: su política de
  confianza nombra al principal de servicio `export.rds.amazonaws.com`, y su
  política de identidad permite escribir en el bucket.
- `--kms-key-id` es **obligatorio**: la exportación siempre se cifra, con una llave
  simétrica cuya política debe permitir al servicio `kms:CreateGrant` y
  `kms:DescribeKey`.
- `--export-only` limita la exportación a `base[.esquema][.tabla]`. Sin él sale todo.

Límites que deciden preguntas: máximo 5 exportaciones simultáneas por cuenta; no se
exportan vistas ni vistas materializadas; los objetos grandes de 500 MB o más hacen
fallar la exportación y las filas de 2 GB o más se omiten; y lo exportado **no se
puede restaurar de vuelta a RDS**.

Cuándo seguir consultando la base en vez de exportar: cuando hace falta una
transformación que el SQL resuelve y el volcado no (una agregación, un `JOIN`
selectivo), cuando se necesita un subconjunto pequeño y reciente, o cuando la
exportación completa es desproporcionada. La forma industrial de esa consulta
programada es [[05 - Catálogo y transformación a escala]].

### DynamoDB: exportar a S3 sin gastar capacidad

En DynamoDB el mecanismo natural de recorrer una tabla entera es `Scan`, y consume
unidades de capacidad de lectura: recorrer la tabla de perfiles para armar un
dataset compite con las consultas que el sistema de fraude hace en línea.

La exportación a S3 no consume capacidad de lectura ninguna, porque no lee la
tabla: lee las copias de seguridad continuas. Por eso su requisito es que la tabla
tenga **PITR** (*point-in-time recovery*) activado, y por eso se puede pedir el
estado de la tabla en cualquier instante dentro de la ventana de PITR.

```bash
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:111111111111:table/kanan-perfil-cliente \
  --s3-bucket kanan-ml-dev-raw-us-east-1 \
  --s3-prefix fraude/2026/perfil-dynamodb/ \
  --s3-sse-algorithm KMS \
  --s3-sse-kms-key-id arn:aws:kms:us-east-1:111111111111:key/3f1c9e02-kanan-ml-dev \
  --export-format DYNAMODB_JSON \
  --export-time 2026-09-19T06:00:00Z \
  --export-type FULL_EXPORT \
  --profile kanan-dev
```

`--export-format` admite `DYNAMODB_JSON` o `ION`; ninguno de los dos es columnar, así
que lo exportado suele ser la entrada de una conversión posterior, no el dataset
final. `--export-type` admite `FULL_EXPORT` (por defecto) o `INCREMENTAL_EXPORT`, y
este último exige además `--incremental-export-specification` con la ventana
temporal: es la forma de traer solo lo que cambió desde la exportación de ayer. La
operación es asíncrona y sin garantía de tiempo de finalización, así que un pipeline
no debe asumir que termina en un plazo fijo.

La tercera vía —capturar los cambios según ocurren, en lugar de exportarlos después—
es [[04 - Ingesta en streaming]].

### Lo que está fuera de AWS

Dos servicios aparecen en el temario solo cuando el enunciado dice «centro de datos
propio»:

- **AWS DataSync** mueve datos entre almacenamiento local (NFS, SMB, HDFS, S3
  compatible) y AWS (S3, EFS, FSx). Es un agente que se instala en la red del
  cliente y ejecuta tareas de transferencia programadas, con verificación de
  integridad y transferencia incremental. Señal: *migrar* o *sincronizar
  periódicamente* un volumen grande.
- **AWS Storage Gateway**, en su variante *Amazon S3 File Gateway*, expone un bucket
  de S3 como un recurso compartido NFS o SMB dentro del centro de datos, con caché
  local. Señal: las aplicaciones locales *siguen trabajando* contra un directorio y
  lo que hay detrás debe ser S3. No es una migración, es un acceso permanente.
- Cuando el volumen es de decenas o cientos de terabytes y la red no da para la
  ventana disponible, la respuesta es el envío físico con la familia **Snow**.

La confusión que el examen busca: DataSync y Storage Gateway resuelven el problema
de *estar fuera de AWS*; Transfer Acceleration resuelve el problema de *estar lejos
de la región*, con los datos ya viajando a S3 por Internet. No son intercambiables.

---

## Cuando lo que falla es la capacidad

La nota 2 dejó un error de cuota, `ResourceLimitExceeded`, que llega en la llamada
y se arregla pidiendo aumento en Service Quotas. Los de esta sección son distintos:
no hay ninguna cuota que subir, hay un recurso saturado.

| Síntoma | Causa | Qué tocar |
| --- | --- | --- |
| `503 Slow Down` al escribir o leer en ráfaga | S3 todavía no ha particionado ese prefijo para la carga nueva | reintentos con espera exponencial (la CLI y boto3 ya los hacen), repartir las claves entre más prefijos, subir la carga de forma gradual |
| La subida de 500 GB tarda horas y no satura la red | pocas conexiones en paralelo, partes de 8 MB | `max_concurrent_requests` y `multipart_chunksize`; si el origen está lejos, evaluar Transfer Acceleration |
| El job pasa más tiempo en `Downloading` que entrenando | modo File con un dataset grande | FastFile, o `ShardedByS3Key` para que cada instancia baje su parte |
| El job muere descargando | el volumen EBS no cabe los datos | subir `VolumeSizeInGB`, repartir, o cambiar de modo |
| La lectura desde el contenedor va a tirones | volumen `gp2` pequeño con IOPS atadas al tamaño, o EFS en `Bursting` con los créditos agotados | `gp3` con línea base fija; en EFS, throughput `Elastic` |
| FSx for Lustre tarda muchísimo la primera vez | el sistema de archivos se crea y se indexa el repositorio vinculado; con millones de objetos puede ser cosa de una hora | crearlo una vez y reutilizarlo entre jobs; aumentar las IOPS de metadatos |
| Con `ShardedByS3Key`, algunas instancias no hacen nada | hay menos objetos que instancias | más objetos, o menos instancias |
| El entrenamiento distribuido se atasca leyendo millones de JPEG | una petición por archivo | consolidar en archivos grandes, o FSx for Lustre |

---

## Lo que queda decidido para el fraude

- **Dónde.** S3, clase Standard, en `us-east-1`, con los buckets de la convención.
  El crudo de años anteriores pasa a Glacier Flexible Retrieval por ciclo de vida, y
  se acepta que restaurarlo cuesta horas.
- **Layout.** `fraude/2026/parquet/mes=AAAA-MM/`, dentro del prefijo `fraude/2026/*`
  que la política de la nota 1 permite. Objetos de unos cientos de MB: suficientes
  para no pagar una petición por migaja, suficientemente numerosos para repartir
  entre instancias.
- **Formato.** Parquet con Snappy, row groups de 200.000 filas, esquema explícito.
  Los 500 GB de CSV quedan en unos 240 GB, y leer las ocho columnas del modelo mueve
  del orden de 50 GB en vez de 500.
- **Modo de entrada.** FastFile. El arranque deja de depender del tamaño y el
  volumen del job baja a 50 GB. FSx for Lustre se queda en reserva para cuando
  empiecen los barridos de hiperparámetros sobre el mismo dataset.
- **Una consecuencia incómoda y honesta.** El XGBoost integrado de la nota 2 consume
  `text/csv` o `text/libsvm`, no Parquet. La capa Parquet sirve para explorar,
  construir rasgos y entrenar con código propio ([[13 - Script mode]]); el canal del
  algoritmo integrado se alimenta de un CSV materializado desde Parquet con las ocho
  columnas y la etiqueta primera. Que el almacén analítico y el canal del algoritmo
  tengan formatos distintos no es un error de diseño: es la consecuencia de que uno
  lo lee un motor de consultas y el otro un contenedor que espera texto.

---

## Preguntas de práctica

### Dificultad media

**1.** Un equipo entrena un modelo sobre 400 GB de archivos Parquet en S3. Cada
archivo pesa unos 300 MB. El entrenamiento en sí dura 30 minutos, pero el job tarda
más de una hora de principio a fin. El código de entrenamiento abre los archivos con
la biblioteca estándar y no se puede modificar en este trimestre. ¿Qué cambio reduce
más el tiempo total?

A. Cambiar el canal a modo Pipe.
B. Cambiar el canal a modo FastFile.
C. Aumentar `VolumeSizeInGB` al doble.
D. Habilitar S3 Transfer Acceleration en el bucket.

<details><summary>Solución</summary>

**B.** El tiempo perdido es la descarga previa del modo File. FastFile monta el
prefijo y las lecturas se resuelven contra S3 bajo demanda, sin tocar el programa:
sigue abriendo archivos normales. Con archivos de 300 MB está además en el rango
donde FastFile rinde bien.

**A** falla por el requisito explícito: Pipe entrega un flujo secuencial por un FIFO
y obliga a reescribir la lectura. **C** no ataca el problema: el disco no es el
cuello de botella, el tiempo de transferencia sí, y un volumen mayor no descarga más
rápido. **D** resuelve distancia entre el cliente y la región; aquí el tráfico es
entre S3 y las instancias de entrenamiento dentro de la misma región.
</details>

---

**2.** Una aplicación escribe telemetría de 3.000 cajeros en S3 con claves de la
forma `telemetria/2026-09-19/atm-0001.json`, un objeto por cajero y minuto. Durante
las horas pico aparecen errores `503 Slow Down` en la escritura. ¿Cuál es la causa y
el remedio correcto?

A. Se superó el límite de objetos por bucket; hay que crear más buckets.
B. Se superó el límite de peticiones de escritura del prefijo de la fecha; conviene
   reordenar la clave para repartir la escritura entre más prefijos.
C. Se agotó la capacidad provisionada del bucket; hay que aumentarla en la consola.
D. Los objetos son demasiado pequeños; hay que activar multipart upload.

<details><summary>Solución</summary>

**B.** El límite es de al menos 3.500 escrituras por segundo y por prefijo
particionado. Todas las claves comparten `telemetria/2026-09-19/`, así que la carga
se concentra en un prefijo. Poner el identificador del cajero antes de la fecha
—`telemetria/atm-0001/2026-09-19/...`— reparte la escritura entre 3.000 prefijos.

**A** es falso: no hay límite de objetos por bucket. **C** es falso: S3 no tiene
capacidad provisionada que se configure. **D** confunde los mecanismos: multipart
sirve para objetos grandes y aquí aumentaría el número de peticiones, que es
justamente el problema.
</details>

---

**3.** (Respuesta múltiple: elija DOS.) Hay que construir un dataset de
entrenamiento a partir de una base RDS PostgreSQL de producción de 2 TB. El
requisito es no degradar el rendimiento de la base y obtener un estado consistente.
¿Qué dos afirmaciones sobre la exportación de una instantánea de RDS a S3 son
correctas?

A. La exportación escribe los datos en S3 en formato Apache Parquet.
B. La exportación se ejecuta contra la instancia activa y consume su CPU y su E/S.
C. La exportación requiere una llave de KMS y siempre cifra la salida.
D. Los datos exportados se pueden restaurar directamente a una instancia RDS nueva.
E. La exportación solo admite instantáneas manuales.

<details><summary>Solución</summary>

**A y C.** La salida es Parquet comprimido, y `--kms-key-id` es obligatorio.

**B** es exactamente lo contrario: el trabajo se hace sobre una restauración
separada de la instantánea, sin tocar la instancia en producción; esa es la razón de
elegir este mecanismo. **D** es falso y está documentado como limitación: lo
exportado no vuelve a RDS. **E** es falso: valen instantáneas manuales, automáticas
y de AWS Backup.
</details>

---

**4.** (Emparejamiento.) Asocie cada escenario con el formato más adecuado. Cada
formato se usa una sola vez.

| Escenario | |
| --- | --- |
| 1. Ingesta continua de eventos cuyo esquema gana campos cada trimestre y debe seguir siendo legible por consumidores antiguos | |
| 2. Consultas repetidas que leen 6 de 80 columnas sobre 3 TB de historial | |
| 3. Entrenamiento con un algoritmo integrado que consume registros por streaming | |
| 4. Intercambio de un extracto de 20 MB con un equipo externo que lo abrirá en una hoja de cálculo | |

Formatos: Parquet · CSV · Avro · RecordIO-protobuf

<details><summary>Solución</summary>

1 → **Avro**: formato de fila con esquema embebido y pensado para evolución de
esquema, que es literalmente el requisito.
2 → **Parquet**: columnar, lee solo los bloques de las columnas pedidas y se salta
row groups con las estadísticas.
3 → **RecordIO-protobuf**: el formato binario por registros que los algoritmos
integrados consumen, y el que hace posible el modo Pipe.
4 → **CSV**: no validado y poco eficiente, pero universal, y a 20 MB nada de eso
importa.
</details>

---

**5.** Una aseguradora en Madrid sube cada noche 300 GB de imágenes a un bucket en
`us-east-1` por Internet, y la transferencia no termina antes de la mañana aunque
sobra ancho de banda contratado. ¿Qué servicio o característica ataca la causa?

A. AWS DataSync.
B. Amazon S3 File Gateway.
C. S3 Transfer Acceleration.
D. Aumentar `multipart_threshold` a 5 GB.

<details><summary>Solución</summary>

**C.** El síntoma —ancho de banda disponible sin aprovechar en una transferencia
intercontinental— es el caso de uso canónico: el tráfico entra por la ubicación de
borde más cercana y cruza el océano por la red de AWS.

**A** es para mover datos entre almacenamiento local y AWS de forma gestionada y
programada; aquí ya existe un proceso de subida que funciona, y DataSync no cambia
la física del trayecto intercontinental de la misma forma. **B** expone S3 como NFS
o SMB local: resuelve *acceso*, no *velocidad de subida masiva*. **D** empeora las
cosas: subir el umbral a 5 GB hace que archivos de menos de 5 GB se suban en una
sola petición, sin paralelismo ni reintentos por partes.
</details>

---

### Dificultad alta

**6.** (Ordenamiento.) Un equipo quiere entrenar leyendo desde FSx for Lustre
vinculado a un prefijo de S3. Ordene los pasos.

1. Crear el sistema de archivos FSx for Lustre en una subred, con su grupo de
   seguridad.
2. Crear la *data repository association* entre una ruta del sistema de archivos y
   el prefijo de S3.
3. Obtener el `MountName` con `describe-file-systems`.
4. Lanzar el training job con un canal `FileSystemDataSource` y la configuración de
   red del sistema de archivos.
5. Conceder al rol de ejecución permisos de lectura sobre el sistema de archivos.

<details><summary>Solución</summary>

**1 → 2 → 3 → 5 → 4** (los pasos 3 y 5 son intercambiables entre sí, pero ambos van
después de 2 y antes de 4).

El orden está forzado por dependencias reales: la asociación necesita un sistema de
archivos existente; el `MountName` es un atributo del sistema de archivos ya creado
y forma parte del `DirectoryPath` del canal, así que hay que tenerlo antes de
componer la petición; y el permiso debe existir antes de que el job intente montar,
o el job falla. El paso 4 lleva además la configuración de red: sin ella el job no
alcanza el sistema de archivos.
</details>

---

**7.** Un training job con esta configuración termina en `Failed`. El
`SecondaryStatusTransitions` muestra `Starting` → `Downloading` → `Failed`, y el
`FailureReason` menciona falta de espacio en el disco.

```json
{
  "AlgorithmSpecification": {"TrainingInputMode": "File", "TrainingImage": "..."},
  "InputDataConfig": [{
    "ChannelName": "train",
    "DataSource": {"S3DataSource": {
      "S3DataType": "S3Prefix",
      "S3Uri": "s3://.../curated/train/",
      "S3DataDistributionType": "FullyReplicated"}}}],
  "ResourceConfig": {"InstanceType": "ml.m5.4xlarge", "InstanceCount": 8,
                     "VolumeSizeInGB": 100}
}
```

El canal `train` contiene 640 GB repartidos en 6.400 objetos. ¿Cuál es el cambio
mínimo que resuelve el fallo manteniendo las ocho instancias y el modo File?

A. Subir `VolumeSizeInGB` a 800.
B. Cambiar `S3DataDistributionType` a `ShardedByS3Key`.
C. Cambiar `InstanceCount` a 1 y `VolumeSizeInGB` a 700.
D. Añadir `CompressionType: "Gzip"` al canal.

<details><summary>Solución</summary>

**B.** Con `FullyReplicated` cada una de las ocho instancias intenta bajar los
640 GB completos en un volumen de 100 GB. Con `ShardedByS3Key` los 6.400 objetos se
reparten y cada instancia recibe unos 80 GB, que caben en 100. Además el reparto
divide por ocho el tiempo de descarga, y hay objetos de sobra para las ocho
instancias.

**A** funciona pero no es mínimo: paga ocho volúmenes de 800 GB para almacenar ocho
copias del mismo dataset, y no reduce el tiempo de descarga. **C** contradice el
enunciado, que exige mantener ocho instancias. **D** es inválido: `CompressionType`
solo se usa en modo Pipe, y en File la petición se rechaza; aunque se aceptara, no
cambia el tamaño de lo que ya está en S3.
</details>

---

**8.** (Respuesta múltiple: elija DOS.) ¿Qué dos afirmaciones sobre el modo Pipe son
correctas?

A. El programa de entrenamiento puede posicionarse en cualquier punto del flujo para
   releer registros de una época anterior.
B. `RecordWrapperType: "RecordIO"` hace que SageMaker envuelva cada objeto de S3 en
   un registro RecordIO cuando los datos están en crudo.
C. El tamaño del volumen de la instancia debe poder alojar el conjunto de datos
   completo.
D. `S3DataType: "AugmentedManifestFile"` solo puede usarse con el modo Pipe.
E. El modo Pipe es incompatible con `ShuffleConfig`.

<details><summary>Solución</summary>

**B y D.**

**A** describe justo lo que Pipe no permite: el FIFO es secuencial y de un solo
sentido, y esa restricción es la razón de que el algoritmo tenga que estar escrito
para este modo. **C** es falso y es una de sus ventajas: los datos no aterrizan en
el disco, que solo necesita espacio para el modelo y las salidas. **E** es falso e
invierte la realidad: `ShuffleConfig` no solo es compatible, sino que en Pipe el
barajado se rehace al principio de cada época.
</details>

---

### Mini caso de estudio

> Kanan quiere mejorar el modelo de documentos KYC. El dataset son 14 millones de
> imágenes de identificaciones, con un tamaño medio de 90 KB, hoy en
> `s3://kanan-ml-dev-raw-us-east-1/kyc/2026/imagenes/`, un objeto por imagen. El plan
> es un barrido de hiperparámetros: unos 40 entrenamientos sobre exactamente el mismo
> conjunto, cada uno de 12 épocas y unas dos horas, en instancias con GPU. El equipo
> probó un primer job en modo FastFile y midió que las GPU pasan la mayor parte del
> tiempo esperando datos. Todo debe permanecer en `us-east-1` y cifrado con la llave
> de Kanan.

**9.** ¿Qué configuración de almacenamiento y entrada resuelve mejor el problema
medido?

A. Modo File, subiendo `VolumeSizeInGB` hasta que quepan las imágenes.
B. Un sistema de archivos FSx for Lustre vinculado al prefijo de S3, con el canal
   apuntando a él mediante `FileSystemDataSource`.
C. Un sistema de archivos EFS con throughput `Bursting`, copiando ahí las imágenes.
D. Mantener FastFile y habilitar Transfer Acceleration en el bucket.

<details><summary>Solución</summary>

**B.** El síntoma —GPU ociosas con 14 millones de objetos de 90 KB— es el caso donde
FastFile se degrada: cada archivo cuesta al menos una petición, y el coste por
archivo domina. FSx for Lustre está diseñado para ese patrón, se vincula al prefijo
de S3 sin duplicar la gestión de los datos, y su coste se reparte entre los 40
entrenamientos, que además no vuelven a pagar el recorrido de los objetos desde cero.

**A** empeora el problema: descargar 14 millones de objetos antes de cada uno de los
40 jobs multiplica el tiempo muerto por 40. **C** cambia de sistema de archivos sin
atacar la causa —EFS no está pensado para este rendimiento— y `Bursting` añade el
riesgo de quedarse en la línea base a mitad del barrido. **D** no aplica: el
problema no es la distancia a la región, es el número de peticiones.
</details>

---

**10.** Además de la configuración anterior, el equipo quiere reducir el tiempo que
tarda el primer job en empezar y bajar el coste del barrido. ¿Qué dos medidas son
las adecuadas? (Elija DOS.)

A. Consolidar las imágenes en archivos contenedores de unos 150 MB antes de crear el
   sistema de archivos.
B. Crear el sistema de archivos una vez y reutilizarlo en los 40 entrenamientos, en
   lugar de crearlo y destruirlo por job.
C. Convertir las imágenes a Parquet.
D. Mover el prefijo de imágenes a S3 Glacier Flexible Retrieval entre barrido y
   barrido.
E. Cambiar el canal a modo Pipe con `CompressionType: "Gzip"`.

<details><summary>Solución</summary>

**A y B.**

**A** ataca la causa raíz por el otro lado: menos archivos y más grandes reducen el
trabajo de metadatos tanto al indexar el repositorio vinculado como al leer, y es la
recomendación estándar para datasets de archivos diminutos.
**B** es la economía del asunto: el arranque en frío de FSx —crear el sistema de
archivos e indexar millones de objetos— se paga una vez y se amortiza entre 40 jobs.

**C** no tiene sentido para imágenes: Parquet es columnar y su ventaja está en leer
algunas columnas de datos tabulares. **D** rompe el barrido: los objetos archivados
no se pueden leer sin restaurarlos, y la restauración tarda horas. **E** contradice
la decisión anterior —el canal ya no viene de S3— y además obligaría a reescribir la
lectura del entrenamiento.
</details>

---

## Términos que esta nota deja definidos

Objeto, clave y prefijo de S3 · límite de peticiones por prefijo particionado y
`503 Slow Down` · multipart upload y sus límites · `TransferConfig` y los ajustes
`s3.*` de la CLI · S3 Transfer Acceleration · clases de almacenamiento y
`RestoreObject` · EBS (`gp3`, `gp2`, `io2`/Provisioned IOPS, `st1`) · EFS y sus
modos de throughput · FSx for Lustre, *data repository association*,
`PerUnitStorageThroughput`, `MountName`, IOPS de metadatos · FSx for NetApp ONTAP ·
formatos validados y no validados · orientación a fila y a columna · Parquet,
*row group* y estadísticas por bloque · ORC · Avro · RecordIO-protobuf y
`write_numpy_to_dense_tensor` · compresión divisible · modos de entrada `File`,
`FastFile` y `Pipe` · `Channel.InputMode`, `CompressionType`, `RecordWrapperType`,
`ShuffleConfig` · `FileSystemDataSource` · `AugmentedManifestFile` ·
`start-export-task` de RDS · `export-table-to-point-in-time` de DynamoDB y PITR ·
DataSync · S3 File Gateway.
