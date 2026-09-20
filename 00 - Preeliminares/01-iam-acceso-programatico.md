---
tema: Acceso programático a AWS e IAM para trabajo de ML
nota: 1/29
certificacion: AWS Certified Machine Learning Engineer – Associate (MLA-C01)
tareas-de-la-guia:
  - "4.3 K: IAM roles, policies y groups que controlan el acceso a los servicios de AWS"
  - "4.3 S: mínimo privilegio sobre artefactos de ML"
  - "4.3 S: políticas y roles para usuarios y aplicaciones que interactúan con sistemas de ML"
  - "4.3 S: diagnóstico de problemas de seguridad (parte IAM)"
prerrequisitos:
  - cuenta de AWS, región, zona de disponibilidad
  - S3 y EC2 como conceptos
  - modelo de responsabilidad compartida
  - Python, JSON, terminal
no-se-usa-aqui:
  - SageMaker más allá de "servicio que necesita un rol"
  - KMS y cifrado
  - VPC, subredes y endpoints
  - CloudTrail
  - IAM Identity Center, federación SAML
  - Organizations y SCP más allá de su nombre
  - Glue, Lambda, CloudFormation, CDK
versiones:
  aws-cli: "2.36.49"
  boto3: "1.43.98"
  sagemaker-python-sdk: "3.22.1 (no se usa en esta nota)"
verificado: 2026-09-20
tags: [aws, iam, boto3, awscli, seguridad, mla-c01]
---

# Acceso programático a AWS e IAM para trabajo de ML

Toda interacción con AWS —consola, terminal, notebook, un servicio llamando a
otro— termina en una petición HTTPS al *endpoint* de un servicio, firmada
criptográficamente con unas credenciales. AWS hace con ella dos cosas
separadas: averigua **quién** la manda (autenticación) y decide **si la
ejecuta** (autorización). Las dos fallan de maneras distintas y se arreglan en
sitios distintos, y confundirlas es el origen de la mayoría de las horas
perdidas de un ML engineer que empieza en AWS.

Esta nota recorre primero la autenticación, que es corta y mecánica: cómo
poner credenciales en tu laptop y en tu código. El resto —la mayor parte— es
autorización: qué documentos consulta AWS, cómo se escriben para conceder lo
mínimo, y cómo se lee un rechazo para saber cuál de esos documentos lo produjo.

Todos los ejemplos ocurren en Kanan Financiera: cuenta `111111111111`
(`kanan-ml-dev`) para experimentación, cuenta `222222222222` (`kanan-ml-prod`)
para producción, región `us-east-1`, y dos ML engineers, `miguel.reyes` y
`ana.torres`.

---

## De `aws configure` a saber quién eres

La AWS CLI es un programa de línea de comandos que traduce lo que escribes en
la terminal a esas peticiones HTTPS firmadas. Hay dos series, v1 y v2; v2 es
la que se instala hoy y la única que recibe funcionalidad nueva, así que toda
esta serie usa v2. Se instala desde el paquete oficial del sistema operativo,
no con `pip` —`pip install awscli` instala la v1, que es una fuente de
confusión clásica—, y se comprueba así:

```
aws --version
```

```
aws-cli/2.36.49 Python/3.13.9 Linux/6.8.0 exe/x86_64
```

La CLI no tiene credenciales propias: las lee de tu disco. Para ponerlas ahí:

```
aws configure --profile kanan-dev
```

```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: json
```

`--profile kanan-dev` — un **perfil** es un conjunto con nombre de
credenciales y ajustes. Sin `--profile`, todo va a parar al perfil `default`,
y cuando mañana necesites hablar con la cuenta de producción no habrá forma de
distinguir una de otra salvo sobrescribiendo. Nombra siempre los perfiles.

`AWS Access Key ID` y `AWS Secret Access Key` — el par de **credenciales de
larga duración**: un identificador público y un secreto. Se generan en IAM
para un usuario y no caducan hasta que alguien las borra. El identificador de
una llave de usuario empieza siempre por `AKIA`; ese prefijo será útil más
adelante, cuando aparezcan credenciales que empiezan por otra cosa.

`Default region name` — la región a la que se dirigen las peticiones cuando el
comando no dice otra cosa. Sin este valor, la mayoría de los comandos fallan
con `You must specify a region`.

El comando escribe en dos archivos distintos, y el reparto no es intuitivo:

```
cat ~/.aws/credentials
```

```
[kanan-dev]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

```
cat ~/.aws/config
```

```
[profile kanan-dev]
region = us-east-1
output = json
```

Lo secreto va en `credentials`, lo demás en `config`. Y en `config` la sección
se llama `[profile kanan-dev]` mientras que en `credentials` se llama
`[kanan-dev]`, sin la palabra `profile`. No hay razón: es una asimetría
histórica de los archivos, y es la causa más común de un perfil que "no
existe" cuando sí existe.

### La llamada que siempre funciona

```
aws sts get-caller-identity --profile kanan-dev
```

```json
{
    "UserId": "AIDA2EXAMPLE4KANAN7Q",
    "Account": "111111111111",
    "Arn": "arn:aws:iam::111111111111:user/miguel.reyes"
}
```

STS es el AWS Security Token Service, el servicio que reparte credenciales y
responde a la pregunta "¿quién soy?". `get-caller-identity` es la única
operación de AWS que no requiere permiso alguno: contesta a cualquiera que
mande una firma válida. Eso la convierte en la primera herramienta de
diagnóstico de la nota: si devuelve un ARN, la autenticación funciona y todo
lo que falle a partir de aquí es autorización.

`Arn` — un **ARN** (*Amazon Resource Name*) es el identificador global y único
de cualquier cosa en AWS. Su forma es:

```
arn:partición:servicio:región:cuenta:recurso
```

En `arn:aws:iam::111111111111:user/miguel.reyes` la partición es `aws` (las
regiones de China y GovCloud usan otras), el servicio es `iam`, y el campo de
región está **vacío** porque IAM es global: un usuario no vive en `us-east-1`,
vive en la cuenta. Los ARN de S3 llevan vacíos los dos campos, región y
cuenta: `arn:aws:s3:::kanan-ml-dev-raw-us-east-1`, con tres dos puntos
seguidos. Esa forma con huecos desconcierta la primera vez y luego se
reconoce al vuelo; los ARN son el vocabulario con el que se escribe todo lo
demás en esta nota.

El `Arn` de la respuesta dice `user/miguel.reyes`. Un **usuario de IAM** es
una identidad creada dentro de una cuenta, con credenciales de larga duración
propias, pensada para representar a una persona o a una aplicación.

> Para personas, las llaves de larga duración son el camino heredado: las
> organizaciones grandes centralizan el acceso humano con IAM Identity Center,
> que reparte credenciales temporales en vez de llaves permanentes. Está fuera
> del alcance de esta serie; lo que MLA-C01 evalúa, y lo que esta nota enseña,
> es el modelo de IAM que hay debajo en cualquier caso.

### Dónde busca la CLI las credenciales, y en qué orden

Este orden explica el fallo más caro de la nota, así que conviene tenerlo
delante. La CLI se queda con el primer sitio donde encuentra credenciales:

1. opciones del propio comando (`--profile`);
2. variables de entorno: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`,
   `AWS_SESSION_TOKEN`;
3. el perfil indicado por `AWS_PROFILE`, o `default`;
4. los archivos `~/.aws/credentials` y `~/.aws/config`;
5. credenciales del entorno de ejecución: las que un contenedor o una máquina
   virtual de AWS exponen a lo que corre dentro de ellos.

**Qué se rompe si...**

- **tienes `AWS_ACCESS_KEY_ID` exportada en la terminal y además pasas
  `--profile kanan-dev`**: gana `--profile`, porque el paso 1 va antes que el
  2. Pero un script que *no* pase `--profile` usará la variable de entorno, y
  si esa variable apunta a producción, el script escribe en producción sin
  avisar. No hay error: hay datos en el bucket equivocado. Por eso el primer
  comando de cualquier script de esta serie es `aws sts get-caller-identity`.
- **borras la línea `region` del perfil**: `You must specify a region. You can
  also configure your region by running "aws configure"`.
- **la región del perfil es `us-west-2`**: `aws s3api list-objects-v2` sobre un
  bucket de `us-east-1` sigue funcionando, porque la CLI redirige, pero un
  `create` cualquiera crea el recurso en Oregón. Para Kanan, cuyos datos no
  pueden salir de `us-east-1`, eso es un incidente regulatorio, no un
  despiste. La configuración local no es defensa suficiente contra eso; el
  cierre efectivo se escribe del lado de los permisos, en la cuenta.

---

## La misma llamada desde Python

```python
import boto3

sesion = boto3.Session(profile_name="kanan-dev", region_name="us-east-1")
sts = sesion.client("sts")
print(sts.get_caller_identity()["Arn"])
```

```
arn:aws:iam::111111111111:user/miguel.reyes
```

`boto3.Session(profile_name=...)` — una sesión es el equivalente en Python de
`--profile`: guarda credenciales, región y ajustes, y de ella salen los
clientes. Se puede omitir y llamar directamente a `boto3.client("sts")`, que
usa el perfil `default`; en un equipo con varias cuentas eso es exactamente el
fallo silencioso del apartado anterior, escrito en Python. La sesión explícita
cuesta una línea y hace visible contra qué cuenta corre el script.

`sesion.client("sts")` — un **cliente** es un objeto con un método por cada
operación de la API de ese servicio. boto3 no lo escribe a mano: lo genera a
partir de la misma descripción de la API que usa la CLI. De ahí sale una
correspondencia que conviene interiorizar, porque permite traducir cualquier
ejemplo de la documentación entre las dos interfaces:

| CLI | boto3 |
|---|---|
| `aws sts get-caller-identity` | `client("sts").get_caller_identity()` |
| `aws s3api list-objects-v2` | `client("s3").list_objects_v2(...)` |
| `--bucket kanan-ml-dev-raw-us-east-1` | `Bucket="kanan-ml-dev-raw-us-east-1"` |
| `--max-keys 100` | `MaxKeys=100` |

El servicio es el primer argumento de `client`, el comando pasa a
`snake_case` y los parámetros pasan de `--kebab-case` a `PascalCase`. La
traducción es mecánica y vale para cualquier ejemplo de la documentación.

### `client` y `resource`

boto3 ofrece una segunda interfaz, `sesion.resource("s3")`, con objetos que
parecen más pythónicos (`bucket.objects.filter(...)`). La documentación oficial
de boto3 dice hoy, literalmente:

> The AWS Python SDK team does not intend to add new features to the resources
> interface in boto3. Existing interfaces will continue to operate during
> boto3's lifecycle.

Es decir: sigue funcionando, no va a crecer, y los servicios y operaciones
nuevos solo aparecen en `client`. Como buena parte del trabajo de ML toca APIs
recientes, esta serie usa `client` en todos los ejemplos. Si heredas código con
`resource`, no hace falta reescribirlo; si escribes código nuevo, no lo uses.

### Respuestas que vienen a trozos: el paginador

```python
s3 = sesion.client("s3")
paginador = s3.get_paginator("list_objects_v2")

total = 0
for pagina in paginador.paginate(
    Bucket="kanan-ml-dev-raw-us-east-1",
    Prefix="fraude/2026/09/",
):
    total += len(pagina.get("Contents", []))

print(total)
```

`get_paginator("list_objects_v2")` — muchas operaciones de AWS truncan la
respuesta y devuelven un testigo (`NextContinuationToken`, `NextToken`, `Marker`
según el servicio) para pedir el resto. `ListObjectsV2` devuelve como máximo
1000 claves. Un bucket de transacciones de Kanan tiene millones, así que un
`s3.list_objects_v2(...)` a secas devuelve mil y **no da ningún error**: el
script cuenta mal y nadie se entera. El paginador encapsula ese bucle.

`pagina.get("Contents", [])` — cuando una página no tiene objetos, la respuesta
**no trae la clave `Contents` vacía: no trae la clave**. `pagina["Contents"]`
levanta `KeyError` sobre un prefijo sin resultados. Es la forma de fallar más
frecuente del código de S3 recién escrito.

### Recursos que tardan en existir: el *waiter*

```python
s3.create_bucket(Bucket="kanan-ml-dev-artifacts-us-east-1")
s3.get_waiter("bucket_exists").wait(
    Bucket="kanan-ml-dev-artifacts-us-east-1",
    WaiterConfig={"Delay": 5, "MaxAttempts": 20},
)
```

`get_waiter("bucket_exists")` — un *waiter* es un bucle de sondeo ya escrito:
repite una operación de consulta hasta que la respuesta indica el estado
esperado, o hasta agotar los intentos. Existe porque muchas APIs de AWS son
asíncronas: la llamada de creación devuelve en milisegundos y el recurso tarda
en estar utilizable. S3 es el caso más leve; los casos dramáticos son los
trabajos que tardan minutos u horas, y ahí el *waiter* es la diferencia entre
un script y un `time.sleep(600)` puesto a ojo ([[Anatomía de un job de
SageMaker]]).

`WaiterConfig={"Delay": 5, "MaxAttempts": 20}` — sondea cada 5 segundos hasta
20 veces, es decir, se rinde a los 100 segundos con
`WaiterError: Max attempts exceeded`. Sin este argumento se usan los valores
por defecto del servicio, que para trabajos largos se quedan cortos.

### Cuando AWS dice que no

Todo error que AWS devuelve al cliente llega a Python como una única excepción,
`ClientError`, con el detalle dentro:

```python
from botocore.exceptions import ClientError

try:
    s3.get_object(
        Bucket="kanan-ml-dev-raw-us-east-1",
        Key="fraude/2026/09/no-existe.parquet",
    )
except ClientError as error:
    print(error.response["Error"]["Code"])
    print(error.response["Error"]["Message"])
    print(error.response["ResponseMetadata"]["HTTPStatusCode"])
```

```
NoSuchKey
The specified key does not exist.
404
```

`error.response["Error"]["Code"]` — la cadena corta y estable con la que se
ramifica el código (`NoSuchKey`, `NoSuchBucket`, `ValidationException`,
`ThrottlingException`, `ResourceLimitExceeded`). El `Message` es prosa que AWS
puede reescribir entre versiones; nunca se compara por igualdad. Un
`except ClientError` que no mira el `Code` está tratando igual "no existe",
"no tienes permiso" y "vuelve a intentarlo dentro de un rato", que son tres
situaciones con tres respuestas distintas.

`ResponseMetadata` — además del error trae `RequestId`, el identificador que el
soporte de AWS pide cuando el error no tiene explicación.

Un detalle que cuesta una tarde la primera vez: `head_object` sobre la misma
clave inexistente no devuelve `NoSuchKey` sino `Code` igual a `"404"` y
`Message` igual a `"Not Found"`. Una respuesta HTTP `HEAD` no tiene cuerpo, y
sin cuerpo no hay XML donde viaje el código detallado; solo queda el estado
HTTP. Si ramificas por `Code` sobre `head_object`, compara con `"404"`.

### Cuándo CLI y cuándo boto3

No son dos sabores de lo mismo. La característica del problema que decide es
**qué pasa entre una llamada y la siguiente**:

- **Nada, o casi nada → CLI.** Inspeccionar, crear un recurso, comprobar un
  estado. Además la CLI trae `--query` (filtro JMESPath sobre la respuesta),
  `--output table|text|json` y `--dry-run` en varios servicios, que en Python
  tendrías que escribir.
- **Lógica condicional, transformación o acumulación → boto3.** En cuanto la
  respuesta de una llamada decide la siguiente, o hay que cruzar dos
  respuestas, el bucle en `bash` sobre `aws ... | jq` se vuelve frágil: cada
  iteración arranca un proceso nuevo, firma de nuevo, y el manejo de errores se
  reduce al código de salida.
- **Integración dentro de una aplicación Python o de un paso de un pipeline →
  boto3**, por lo mismo y porque ahí ya estás en Python.
- **Infraestructura que debe ser reproducible → ninguno de los dos.** Ni la CLI
  ni boto3 saben qué creaste ayer, así que un script que se ejecuta dos veces
  falla la segunda con `EntityAlreadyExists`. Eso es [[IaC]].

Regla operativa para esta serie: los recursos de IAM se crean una vez y se
revisan mucho, así que aquí verás sobre todo CLI, con la versión en boto3
cuando la diferencia sea instructiva.

---

## Usuario, grupo y rol

`miguel.reyes` y `ana.torres` necesitan exactamente los mismos permisos sobre
los buckets de desarrollo. Conceder esos permisos dos veces, una por usuario,
significa que dentro de seis meses habrá dos conjuntos de permisos distintos,
porque alguien actualizará uno y olvidará el otro.

```
aws iam create-group --group-name KananMLEngineers-dev --profile kanan-dev
```

```json
{
    "Group": {
        "Path": "/",
        "GroupName": "KananMLEngineers-dev",
        "GroupId": "AGPA2EXAMPLE4KANANGR",
        "Arn": "arn:aws:iam::111111111111:group/KananMLEngineers-dev",
        "CreateDate": "2026-09-20T18:04:11+00:00"
    }
}
```

```
aws iam add-user-to-group --group-name KananMLEngineers-dev --user-name miguel.reyes --profile kanan-dev
aws iam add-user-to-group --group-name KananMLEngineers-dev --user-name ana.torres  --profile kanan-dev
```

Un **grupo** de IAM es un contenedor de permisos con una lista de usuarios
dentro. No es una identidad: no tiene credenciales, no aparece nunca como quien
hace una llamada, y no se puede "entrar" en un grupo. Lo único que hace es que
los permisos que le cuelgas valgan para todos sus miembros. IAM
limita a 10 los documentos de permisos que se pueden colgar de un grupo, y ese
tope se alcanza antes de lo que parece en cuanto el equipo crece.

La tercera forma de identidad es la que domina el trabajo de ML. Un **rol** de
IAM es una identidad **sin credenciales permanentes**, que existe para que
alguien la use temporalmente. Quien la usa —una persona de otra cuenta, un
servicio de AWS, una máquina— recibe **credenciales temporales**: el mismo par
de llave e identificador de siempre, más un tercer valor, el *session token*,
y una fecha de expiración. El identificador de una credencial temporal empieza
por `ASIA` en vez de `AKIA`; es la forma rápida de saber, mirando un archivo de
configuración ajeno, si alguien dejó ahí una llave permanente.

Por qué el rol y no el usuario, en tres consecuencias prácticas:

- **No hay nada que rotar ni que filtrar.** Una llave `AKIA` en un repositorio
  sirve hasta que alguien la revoque. Unas credenciales `ASIA` caducan solas.
- **Un servicio no puede tener usuario.** Cuando el entrenamiento corra en una
  máquina administrada por AWS, no habrá dónde poner un archivo
  `~/.aws/credentials`, ni quién lo escriba.
- **El permiso se concede por tarea, no por persona.** Los mismos dos ingenieros
  pueden tener permisos distintos según qué rol estén usando en ese momento.

Para que un rol sirva de algo hacen falta antes los documentos que dicen qué
puede hacer cada identidad.

---

## Qué hay dentro de una política de identidad

Con el grupo creado y los dos usuarios dentro, esto es lo que ocurre al
listar el bucket de datos curados:

```python
for pagina in paginador.paginate(Bucket="kanan-ml-dev-curated-us-east-1"):
    print(len(pagina.get("Contents", [])))
```

```
botocore.exceptions.ClientError: An error occurred (AccessDenied) when calling
the ListObjectsV2 operation: User: arn:aws:iam::111111111111:user/miguel.reyes
is not authorized to perform: s3:ListBucket on resource:
"arn:aws:s3:::kanan-ml-dev-curated-us-east-1" because no identity-based policy
allows the s3:ListBucket action
```

El mensaje trae las cuatro piezas con las que AWS toma la decisión, y merece
leerse despacio porque su estructura se repite en todos los rechazos de la
nota: **quién** (`User: arn:aws:iam::111111111111:user/miguel.reyes`), **qué
acción** (`s3:ListBucket`), **sobre qué recurso**
(`arn:aws:s3:::kanan-ml-dev-curated-us-east-1`) y **por qué**
(`because no identity-based policy allows`). Esa última coletilla es la que
dice dónde está el arreglo.

Una **política** es un documento JSON que concede o niega permisos. La que
falta aquí es una **política de identidad**: la que se adjunta a un usuario, un
grupo o un rol, y que responde a la pregunta "¿qué puede hacer este principal?".
Un **principal** es quien hace la llamada: el usuario, el rol o el servicio que
firmó la petición.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListarLosBucketsDeMLDev",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": [
        "arn:aws:s3:::kanan-ml-dev-raw-us-east-1",
        "arn:aws:s3:::kanan-ml-dev-curated-us-east-1",
        "arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1"
      ]
    },
    {
      "Sid": "LeerYEscribirObjetosDeMLDev",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": [
        "arn:aws:s3:::kanan-ml-dev-raw-us-east-1/*",
        "arn:aws:s3:::kanan-ml-dev-curated-us-east-1/*",
        "arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1/*"
      ]
    }
  ]
}
```

`"Version": "2012-10-17"` — no es la versión de tu documento: es la versión del
lenguaje de políticas, y esa fecha es la única válida para políticas nuevas. El
nombre es engañoso y no hay nada que decidir aquí; se copia tal cual. Omitirlo
no da error al crear la política, pero desactiva funcionalidad del lenguaje
—entre otras, las variables de política—, así que siempre va.

`Statement` — una lista de bloques independientes. AWS los evalúa todos, no se
detiene en el primero que encaja.

`Sid` — identificador libre dentro del documento. Es opcional y puramente
humano, con una excepción que lo hace valioso: aparece en la salida del
simulador de la última sección, que dice qué bloque decidió. Sin `Sid`, el
simulador te devuelve números de línea.

`Effect`, `Action`, `Resource` — quién concede qué sobre qué. `Action` usa la
forma `servicio:Operación` y admite comodines (`s3:Get*`, `s3:*`). `Resource`
es una lista de ARN, también con comodines.

**Y ahora lo que el examen pregunta una y otra vez:** hay dos bloques porque hay
dos formas de ARN. `arn:aws:s3:::mi-bucket` designa *el bucket*;
`arn:aws:s3:::mi-bucket/*` designa *los objetos dentro del bucket*. Son
recursos distintos y las acciones se reparten entre ellos:

| Acción | Recurso que necesita |
|---|---|
| `s3:ListBucket`, `s3:GetBucketLocation` | el bucket, sin `/*` |
| `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` | los objetos, con `/*` |

Ahora se crea y se adjunta:

```
aws iam create-policy \
  --policy-name KananMLEngineerS3-dev \
  --policy-document file://kanan-ml-engineer-s3-dev.json \
  --profile kanan-dev

aws iam attach-group-policy \
  --group-name KananMLEngineers-dev \
  --policy-arn arn:aws:iam::111111111111:policy/KananMLEngineerS3-dev \
  --profile kanan-dev
```

`file://` — con dos barras y ruta relativa al directorio actual. Es una
convención de la CLI, no del shell; `--policy-document kanan-...json` pasa el
nombre del archivo como si fuera el JSON y devuelve
`Error parsing parameter '--policy-document': Invalid JSON received`.

`create-policy` devuelve el ARN de la política, que es lo que `attach-group-policy`
necesita. Una política así, con nombre y ARN propios, es una **política
gestionada**: existe como objeto independiente y se puede adjuntar a varias
identidades. Frente a ella, una **política en línea** (`put-group-policy`,
`put-role-policy`) es un documento incrustado dentro de una identidad, sin ARN,
que desaparece con ella.

| | Gestionada | En línea |
|---|---|---|
| Reutilizable en varias identidades | sí | no |
| Tiene ARN y versiones | sí (hasta 5 versiones) | no |
| Tamaño máximo del documento | 6 144 caracteres | 2 048 (usuario), 5 120 (grupo), 10 240 (rol) |
| Se borra al borrar la identidad | no | sí |

Y entre las gestionadas hay dos clases: las **gestionadas por AWS**, que AWS
escribe, mantiene y actualiza (`arn:aws:iam::aws:policy/...`), y las
**gestionadas por el cliente**, como la que acabas de crear
(`arn:aws:iam::111111111111:policy/...`). La diferencia importa y tiene sección
propia más abajo.

Regla práctica: gestionada por el cliente por defecto; en línea solo cuando el
permiso no tiene sentido fuera de esa identidad concreta —el caso típico es el
permiso a medida de un rol de ejecución—, porque así no queda huérfana cuando
se borra el rol.

**Qué se rompe si...**

- **pones `arn:aws:s3:::kanan-ml-dev-curated-us-east-1/*` en el bloque de
  `ListBucket`**: `AccessDenied ... no identity-based policy allows the
  s3:ListBucket action`. Idéntico al error de partida, y por eso desconcierta:
  parece que la política no se aplicó, cuando sí se aplicó y no encaja.
- **pones el ARN sin `/*` en el bloque de `GetObject`**: la misma forma de
  error, ahora sobre una clave concreta.
- **escribes `"Action": "s3:listbucket"`**: las acciones distinguen mayúsculas.
  No hay error al crear la política: el bloque queda ahí sin conceder nada nunca.
- **el documento supera 6 144 caracteres**: `LimitExceeded: Maximum policy size
  of 6144 bytes exceeded`. La salida es partirlo en dos políticas gestionadas,
  no comprimir el JSON.

---

## `Condition`: recortar el permiso al prefijo y a la región

La política anterior concede los tres buckets enteros a los dos ingenieros. Eso
no es mínimo privilegio: el bucket de datos curados de Kanan contiene también
los documentos de KYC, que llevan PII y que el equipo de fraude no necesita.
El recorte natural sería por ARN, y para los objetos funciona; para listar, no,
porque el recurso de `ListBucket` es el bucket entero y no admite prefijo.

Ahí entra el cuarto elemento de un bloque de política:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListarSoloElPrefijoDeFraude",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::kanan-ml-dev-curated-us-east-1",
      "Condition": {
        "StringLike": {"s3:prefix": ["fraude/2026/*"]},
        "StringEquals": {"aws:RequestedRegion": "us-east-1"}
      }
    },
    {
      "Sid": "LeerObjetosDelPrefijoDeFraude",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::kanan-ml-dev-curated-us-east-1/fraude/2026/*",
      "Condition": {
        "StringEquals": {"aws:RequestedRegion": "us-east-1"}
      }
    },
    {
      "Sid": "NadaFueraDeTLS",
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {"Bool": {"aws:SecureTransport": "false"}}
    }
  ]
}
```

Un bloque `Condition` tiene tres niveles: **operador** → **clave de condición**
→ **valores**. `StringLike` es el operador, `s3:prefix` la clave,
`["fraude/2026/*"]` los valores. Y la lógica entre ellos no es simétrica, lo
cual es una fuente constante de políticas que conceden de más:

- varias claves dentro de un mismo `Condition`, o varios operadores, se
  combinan con **Y**: aquí hay que cumplir el prefijo **y** la región;
- varios valores para una misma clave se combinan con **O**:
  `["fraude/2026/*", "fraude/2025/*"]` deja listar cualquiera de los dos.

`s3:prefix` — clave de condición **de servicio**: la aporta S3 y solo existe
en las llamadas de S3. Se compara contra el parámetro `Prefix` de la petición,
y la documentación es explícita en que aplica a `s3:ListBucket` (la operación
`ListObjectsV2`). No aplica a `GetObject`, y ese es el punto que el examen
prueba: **en `ListBucket` el recorte por carpeta se hace con `s3:prefix`; en
`GetObject` se hace con el ARN del recurso.** Escribir `s3:prefix` en el bloque
de `GetObject` no produce error al guardar la política: produce una condición
que nunca se cumple, porque la clave no está en el contexto de la petición, y
por tanto un `AccessDenied` permanente.

`aws:RequestedRegion` — clave de condición **global**: empieza por `aws:`, la
aporta IAM y está disponible en las peticiones a *endpoints* regionales de
cualquier servicio. Compara la región a la que se dirigió la llamada. Esto es
la restricción regulatoria de Kanan escrita como permiso: aunque alguien
configure mal un perfil, las llamadas a otra región no se autorizan.

`Bool` con el valor `"false"` entre comillas — los valores de una política son
siempre cadenas, incluso los booleanos. `"aws:SecureTransport": false` sin
comillas es JSON válido y política inválida.

`Effect: Deny` en el tercer bloque — un bloque puede negar, no solo conceder, y
un `Deny` derrota a cualquier `Allow`. Ese es el mecanismo que sostiene la
sección "Cómo decide AWS"; aquí basta con saber que este bloque cierra la
puerta a HTTP en claro pase lo que pase en el resto de la política.

Comprobación desde la terminal:

```
aws s3api list-objects-v2 \
  --bucket kanan-ml-dev-curated-us-east-1 \
  --prefix fraude/2026/09/ \
  --profile kanan-dev
```

Devuelve las claves. Y sin el prefijo:

```
aws s3api list-objects-v2 --bucket kanan-ml-dev-curated-us-east-1 --profile kanan-dev
```

```
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:iam::111111111111:user/miguel.reyes is not authorized to perform:
s3:ListBucket on resource: "arn:aws:s3:::kanan-ml-dev-curated-us-east-1"
because no identity-based policy allows the s3:ListBucket action
```

Sin `--prefix`, la petición no lleva la clave `s3:prefix`, la condición no se
cumple, el bloque no concede, y no hay otro bloque que conceda: denegación. El
mensaje es indistinguible del de no tener la política en absoluto, y aprender
a sospechar de una condición ante este mensaje ahorra mucho tiempo.

Otro operador que conviene conocer por su rareza: `Null` comprueba si una clave
**está o no está presente** en la petición, no su valor.
`"Null": {"s3:prefix": "false"}` significa "la petición trae algún prefijo,
sea cual sea". Es la forma de exigir que quien liste el bucket se limite a
alguna carpeta sin decir cuál.

**Qué se rompe si...**

- **el ingeniero abre la consola de S3 y navega al bucket**: la consola lista la
  raíz sin prefijo, así que ve `AccessDenied` aunque su trabajo por código
  funcione. Es el reporte de incidencia más frecuente tras aplicar esta
  política, y la respuesta correcta no es ampliar el permiso sino darle el
  enlace directo a la carpeta.
- **añades `"StringEquals": {"s3:prefix": "fraude/2026/"}`** en vez de
  `StringLike` con comodín: solo se autoriza listar exactamente esa carpeta, no
  `fraude/2026/09/`.
- **alguien copia datos con una herramienta que usa `s3:ListBucketVersions`**:
  la política no la menciona, así que falla. `s3:prefix` también aplica a esa
  acción, pero hay que concederla aparte.

---

## Cruzar de dev a prod: la política de confianza

Un ingeniero de Kanan trabaja en `kanan-ml-dev` y a veces necesita mirar algo
en `kanan-ml-prod`. Darle un segundo usuario, con un segundo par de llaves, en
la segunda cuenta, multiplica por dos las llaves que rotar y deja dos
identidades que auditar por persona. La alternativa es que su identidad de
desarrollo tome prestada, durante una hora, una identidad de producción.

Eso exige dos documentos, en dos cuentas distintas, y entender que son
distintos es la mitad de IAM.

Hasta ahora, toda política respondía "¿qué puede hacer este principal?". Un rol
necesita además responder a otra pregunta: "¿**quién** puede usar este rol?".
Ese segundo documento es la **política de confianza** (*trust policy*) del rol,
y es lo que distingue a un rol de un usuario. Se escribe en el mismo lenguaje,
pero tiene un elemento que las políticas de identidad no pueden tener:
`Principal`.

En la cuenta de producción, `222222222222`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ConfiarEnLosUsuariosDeLaCuentaDeDev",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::111111111111:root"},
      "Action": "sts:AssumeRole",
      "Condition": {
        "ArnLike": {"aws:PrincipalArn": "arn:aws:iam::111111111111:user/*"}
      }
    }
  ]
}
```

```
aws iam create-role \
  --role-name KananMLReadArtifacts-prod \
  --assume-role-policy-document file://confianza-dev-a-prod.json \
  --profile kanan-prod-admin

aws iam attach-role-policy \
  --role-name KananMLReadArtifacts-prod \
  --policy-arn arn:aws:iam::222222222222:policy/KananLecturaArtefactos-prod \
  --profile kanan-prod-admin
```

`"Principal": {"AWS": "arn:aws:iam::111111111111:root"}` — la palabra `root`
aquí **no significa el usuario raíz** de la cuenta, aunque lo parezca. Es la
forma de escribir "la cuenta `111111111111` entera", y lo que expresa es una
delegación: producción confía en que el administrador de desarrollo decida qué
identidades de su cuenta pueden asumir el rol. Este es probablemente el nombre
peor elegido de IAM y no hay nada que entender detrás: se memoriza.

`"Action": "sts:AssumeRole"` — la acción que se concede en una política de
confianza es siempre la de asumir el rol. No se ponen aquí los permisos del
rol: esos van en `attach-role-policy`, en un documento separado, la **política
de permisos**. Dos documentos, dos preguntas: quién entra, y qué puede hacer
una vez dentro.

`"ArnLike": {"aws:PrincipalArn": "arn:aws:iam::111111111111:user/*"}` — la
delegación en bloque a una cuenta entera es amplia; esta condición la acota a
los usuarios de esa cuenta. `aws:PrincipalArn` es la clave global que contiene
el ARN de quien llama.

`--profile kanan-prod-admin` — nota el cambio de perfil: estos dos comandos se
ejecutan **contra la cuenta de producción**, con credenciales de alguien que
administra IAM allí. El ingeniero de desarrollo no puede crearse a sí mismo la
confianza; ese es justamente el punto.

Con eso, producción ya confía. Falta el otro lado: desarrollo tiene que
permitir a sus usuarios salir. En `111111111111`, adjunta al grupo:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CruzarAProdSoloParaLeerArtefactos",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::222222222222:role/KananMLReadArtifacts-prod"
    }
  ]
}
```

**Las dos mitades son obligatorias y fallan distinto.** Si falta la política de
identidad en dev, el error nombra a tu usuario y dice
`is not authorized to perform: sts:AssumeRole`. Si falta la confianza en prod,
el error es `AccessDenied ... User: arn:aws:iam::111111111111:user/miguel.reyes
is not authorized to perform: sts:AssumeRole on resource:
arn:aws:iam::222222222222:role/KananMLReadArtifacts-prod`, y hay que ir a
mirar el rol de la otra cuenta.

### El cruce, paso a paso

```mermaid
sequenceDiagram
    participant U as miguel.reyes en dev
    participant P as politica de identidad en dev
    participant S as STS AssumeRole
    participant T as politica de confianza del rol en prod
    participant A as API de S3 en prod

    U->>S: sts:AssumeRole sobre KananMLReadArtifacts-prod
    S->>P: ¿dev permite salir?
    alt no hay Allow de sts:AssumeRole en dev
        P-->>U: AccessDenied (no identity-based policy allows)
    else dev permite
        S->>T: ¿prod confía en este principal?
        alt el Principal o la condición no encajan
            T-->>U: AccessDenied sobre el ARN del rol
        else prod confía
            S-->>U: Credentials (ASIA..., SessionToken, Expiration)
            U->>A: llamada firmada con las credenciales temporales
            A-->>U: respuesta segun la politica de permisos del rol
        end
    end
```

**Lectura.** El viaje tiene dos puertas y la primera está en tu propia cuenta:
antes de que producción opine, IAM en desarrollo tiene que permitirte llamar a
`sts:AssumeRole`. Solo si pasas esa puerta, STS consulta la política de
confianza del rol en la otra cuenta. Lo que vuelve no es un permiso: es un
juego de credenciales nuevas, con caducidad. A partir de ahí tus llaves `AKIA`
de desarrollo no intervienen en nada; todas las peticiones a producción se
firman con las `ASIA` que devolvió STS, y lo que puedas hacer con ellas lo
decide la política de permisos del rol, no la tuya. Cuando expiren, las
llamadas empiezan a fallar con `ExpiredToken`, que es un error de
autenticación y no de autorización: no se arregla con más permisos, se arregla
volviendo a asumir.

### En la terminal

```
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/KananMLReadArtifacts-prod \
  --role-session-name miguel-revision-artefactos \
  --duration-seconds 3600 \
  --profile kanan-dev
```

```json
{
    "Credentials": {
        "AccessKeyId": "ASIA2EXAMPLE4KANAN7Q",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
        "SessionToken": "IQoJb3JpZ2luX2VjEExample...",
        "Expiration": "2026-09-20T19:34:41+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA2EXAMPLE4KANANRO:miguel-revision-artefactos",
        "Arn": "arn:aws:sts::222222222222:assumed-role/KananMLReadArtifacts-prod/miguel-revision-artefactos"
    }
}
```

`--role-session-name` — obligatorio, de 2 a 64 caracteres. Se pega al ARN de la
sesión y es lo que queda registrado en la auditoría, así que un nombre que
identifique a la persona y el motivo vale mucho más que `sesion1`. Es la única
pista de quién estaba detrás de un rol compartido ([[Observabilidad]]).

`--duration-seconds` — entre 900 (15 min) y 43 200 (12 h), por defecto 3 600.
Además está topado por el `MaxSessionDuration` del propio rol, que por defecto
es 1 hora: pedir más que eso devuelve `ValidationError: The requested
DurationSeconds exceeds the MaxSessionDuration set for this role`. Y si quien
asume el rol ya está usando un rol (encadenamiento de roles), el máximo baja a
1 hora sin importar la configuración.

`AssumedRoleUser.Arn` — fíjate en dos cosas: el servicio es `sts`, no `iam`, y
la forma es `assumed-role/NombreDelRol/NombreDeSesion`. Esa es la cadena que
verás en las auditorías y la que devuelve `get-caller-identity` mientras usas
el rol. En cambio, cuando nombres ese rol **dentro de una política**, se usa su
ARN de IAM: `arn:aws:iam::222222222222:role/KananMLReadArtifacts-prod`. Dos
formas para la misma cosa según el contexto.

Pegar esas tres cadenas a mano en variables de entorno funciona una vez. La
forma sostenible es declarar el cruce como un perfil, en `~/.aws/config`:

```
[profile kanan-prod-lectura]
role_arn = arn:aws:iam::222222222222:role/KananMLReadArtifacts-prod
source_profile = kanan-dev
role_session_name = miguel-revision-artefactos
region = us-east-1
```

```
aws sts get-caller-identity --profile kanan-prod-lectura
```

```json
{
    "UserId": "AROA2EXAMPLE4KANANRO:miguel-revision-artefactos",
    "Account": "222222222222",
    "Arn": "arn:aws:sts::222222222222:assumed-role/KananMLReadArtifacts-prod/miguel-revision-artefactos"
}
```

`source_profile` — el perfil cuyas credenciales se usan para llamar a
`AssumeRole`. La CLI hace la llamada por ti, guarda las credenciales
temporales en caché y **las renueva sola** cuando caducan. Ese es el argumento
decisivo frente a copiar el JSON a mano: un script largo que asume el rol al
principio y corre dos horas fallará a mitad con credenciales pegadas, y no
fallará con perfil.

`external_id` es un cuarto ajuste que aparece en este mismo sitio cuando quien
asume el rol es un tercero ajeno a la organización: un secreto compartido que
la política de confianza exige. En un cruce entre dos cuentas propias no se
usa.

### En boto3

```python
import boto3

sesion_prod = boto3.Session(profile_name="kanan-prod-lectura", region_name="us-east-1")
s3_prod = sesion_prod.client("s3")
```

Esa es la forma corta, y aprovecha la misma caché y la misma renovación
automática que la CLI, porque el archivo de configuración es el mismo. La
forma larga solo hace falta cuando el perfil no puede existir —por ejemplo,
dentro de un contenedor construido sin `~/.aws`—:

```python
sts = boto3.Session(profile_name="kanan-dev").client("sts")

respuesta = sts.assume_role(
    RoleArn="arn:aws:iam::222222222222:role/KananMLReadArtifacts-prod",
    RoleSessionName="miguel-revision-artefactos",
)
c = respuesta["Credentials"]

sesion_prod = boto3.Session(
    aws_access_key_id=c["AccessKeyId"],
    aws_secret_access_key=c["SecretAccessKey"],
    aws_session_token=c["SessionToken"],
    region_name="us-east-1",
)
```

`aws_session_token=c["SessionToken"]` — el tercer valor es el que distingue
estas credenciales de unas permanentes. Omitirlo produce
`InvalidClientTokenId: The security token included in the request is invalid`,
un mensaje que suena a llave mal copiada y en realidad significa "le pasaste
llaves temporales sin su token".

Esta sesión **no se renueva**: cuando pase la hora, todo cliente creado a
partir de ella empieza a devolver `ExpiredToken`. Si el proceso vive más que la
sesión, hay que volver a llamar a `assume_role`.

---

## Cuando quien asume el rol es un servicio

Un entrenamiento no corre en tu laptop: corre en una máquina que AWS levanta,
con un contenedor dentro, y ese contenedor necesita leer datos de S3 y escribir
el modelo. No hay dónde poner un `~/.aws/credentials`, y aunque lo hubiera,
meter llaves permanentes en una imagen de contenedor es exactamente lo que
nadie quiere.

La solución es la misma pieza de la sección anterior con un principal distinto:
un rol cuya política de confianza, en vez de nombrar una cuenta, nombra un
**servicio**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SageMakerPuedeAsumirEsteRol",
      "Effect": "Allow",
      "Principal": {"Service": "sagemaker.amazonaws.com"},
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {"aws:SourceAccount": "111111111111"}
      }
    }
  ]
}
```

```
aws iam create-role \
  --role-name KananSageMakerExecutionRole-dev \
  --assume-role-policy-document file://confianza-sagemaker.json \
  --profile kanan-dev
```

`{"Service": "sagemaker.amazonaws.com"}` — un **principal de servicio**: la
identidad con la que un servicio de AWS actúa. La cadena es fija por servicio y
no se inventa; se busca en la documentación del servicio.

Un rol con esta confianza es un **rol de ejecución**: el rol que un servicio
asume *en tu nombre* para hacer el trabajo que le encargaste. Todo lo que el
trabajo pueda tocar lo decide la política de permisos de este rol, no la tuya.
Es la pieza que hace que la sección siguiente exista.

`aws:SourceAccount` — el principal de servicio es común a todos los clientes
de AWS. Sin condición, la confianza dice "cualquier invocación de SageMaker
puede asumir este rol", incluida una lanzada desde la cuenta de otra empresa
que de algún modo consiguiera referenciarlo. Esta condición exige que la
llamada se origine en la cuenta de Kanan. Es el remedio estándar al problema
del *confused deputy*: un servicio legítimo usado como intermediario por un
tercero. `aws:SourceArn` hace lo mismo con más precisión, acotando al ARN del
recurso concreto que origina la llamada.

Y los permisos, en un documento aparte. Aquí el rol de ejecución de fraude en
desarrollo: lee lo curado, escribe artefactos, nada más.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LeerDatosCuradosDeFraude",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::kanan-ml-dev-curated-us-east-1",
        "arn:aws:s3:::kanan-ml-dev-curated-us-east-1/fraude/*"
      ]
    },
    {
      "Sid": "EscribirArtefactosDelModelo",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1/fraude/*"
    }
  ]
}
```

El mismo rol, creado desde Python, para ver la correspondencia con la CLI:

```python
import json

iam = boto3.Session(profile_name="kanan-dev").client("iam")

confianza = {
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "sagemaker.amazonaws.com"},
        "Action": "sts:AssumeRole",
        "Condition": {"StringEquals": {"aws:SourceAccount": "111111111111"}},
    }],
}

iam.create_role(
    RoleName="KananSageMakerExecutionRole-dev",
    AssumeRolePolicyDocument=json.dumps(confianza),
    Description="Rol de ejecucion para entrenamientos de fraude en dev",
    MaxSessionDuration=3600,
)
```

`AssumeRolePolicyDocument=json.dumps(confianza)` — el parámetro espera una
**cadena**, no un diccionario. Pasarle el `dict` directamente produce
`ParamValidationError: Invalid type for parameter AssumeRolePolicyDocument,
value: {...}, type: <class 'dict'>, valid types: <class 'str'>`. Es el error
más repetido al pasar de la CLI (donde el JSON llega desde un archivo) a boto3.

Este rol todavía no lo usa nadie. Quien lo va a usar es el servicio, y para eso
hay que entregárselo.

### El mismo mecanismo fuera de SageMaker

Cuando el código corre en una máquina EC2, el rol se le entrega a la máquina a
través de un **instance profile**: un envoltorio de un rol que se asocia a la
instancia. El código que corre dentro no ve ninguna llave; lo que ocurre es que
llega al paso 5 de la cadena de resolución de la primera sección —credenciales
del entorno de ejecución— y las obtiene del propio entorno, renovadas
automáticamente. Por eso `boto3.client("s3")` sin perfil ni credenciales
funciona dentro de una instancia y falla en tu laptop: no es magia, es el
último eslabón de una lista que ya conoces.

Un notebook administrado por AWS funciona igual: su identidad es un rol de
ejecución, no un perfil. Si abres una terminal en él y ejecutas
`aws sts get-caller-identity`, obtendrás un ARN de la forma
`assumed-role/...`, y cualquier `AccessDenied` que veas ahí se arregla en la
política de ese rol, no en la de tu usuario. Cómo se configura ese entorno es
[[Anatomía de un job de SageMaker]].

---

## `iam:PassRole`: entregar un rol es un permiso aparte

> **Caja negra.** `aws sagemaker create-training-job` lanza un entrenamiento.
> Recibe un nombre, el ARN de un rol de ejecución, una imagen de contenedor y
> rutas de S3; devuelve el ARN del trabajo y empieza a ejecutarlo. Qué hace por
> dentro, qué significa cada campo y cómo se elige la instancia es
> [[Anatomía de un job de SageMaker]]. Copia el JSON tal cual: aquí solo
> interesa que es una llamada que **lleva dentro el ARN de un rol**.

```json
{
  "TrainingJobName": "fraude-xgb-2026-09-20-01",
  "RoleArn": "arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev",
  "AlgorithmSpecification": {
    "TrainingImage": "683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.7-1",
    "TrainingInputMode": "File"
  },
  "OutputDataConfig": {
    "S3OutputPath": "s3://kanan-ml-dev-artifacts-us-east-1/fraude/"
  },
  "ResourceConfig": {
    "InstanceType": "ml.m5.xlarge",
    "InstanceCount": 1,
    "VolumeSizeInGB": 30
  },
  "StoppingCondition": {"MaxRuntimeInSeconds": 3600}
}
```

```
aws sagemaker create-training-job --cli-input-json file://training-job.json --profile kanan-dev
```

```
An error occurred (AccessDeniedException) when calling the CreateTrainingJob
operation: User: arn:aws:iam::111111111111:user/miguel.reyes is not authorized
to perform: iam:PassRole on resource:
arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev because no
identity-based policy allows the iam:PassRole action
```

Lo razonable, llegado aquí, es pensar que sobra un permiso: el rol existe, su
política de confianza ya autoriza a SageMaker a asumirlo, y el ingeniero tiene
permiso para crear trabajos de entrenamiento. ¿Qué más hace falta?

Lo que falla en ese razonamiento es suponer que el trabajo corre con **tus**
permisos. No: corre con los del rol. Si entregar un rol fuera gratis, cualquiera
con permiso para lanzar un entrenamiento podría entregarle el rol de
administración de la cuenta y ejecutar código arbitrario con él. El permiso
para crear el trabajo y el permiso para decidir *con qué identidad* corre son
dos cosas distintas, y AWS las separa: `iam:PassRole` es el permiso de
entregar un rol concreto a un servicio concreto.

`iam:PassRole` no es una operación de la API: nadie llama nunca a `PassRole`.
Es una comprobación que hace el servicio receptor antes de aceptar el ARN. Una
consecuencia práctica que el examen ha preguntado: **no genera evento de
auditoría propio** ([[Observabilidad]]); lo que se ve en el registro es el
`CreateTrainingJob` que lo provocó.

El arreglo mínimo, añadido a la política del grupo:

```json
{
  "Sid": "EntregarSoloElRolDeEjecucionDeMLAServicioSageMaker",
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev",
  "Condition": {
    "StringEquals": {"iam:PassedToService": "sagemaker.amazonaws.com"}
  }
}
```

`Resource` — el ARN del rol que se puede entregar. Este es el sitio donde vive
el mínimo privilegio de esta sección: `"Resource": "*"` deja entregar
cualquier rol de la cuenta, incluido el de administración, y convierte el
permiso de lanzar entrenamientos en un permiso de administración encubierto.
Un patrón por prefijo, `arn:aws:iam::111111111111:role/KananSageMaker*`,
es el término medio razonable cuando hay varios roles de ejecución.

`iam:PassedToService` — clave de condición que solo existe para `PassRole` y
que se compara con el principal de servicio que acabará asumiendo el rol. Con
ella, aunque el ingeniero encontrara otra API que acepte este mismo rol, solo
podría entregárselo a SageMaker.

Y hace falta **además** el permiso sobre la operación en sí:

```json
{
  "Sid": "LanzarYConsultarEntrenamientos",
  "Effect": "Allow",
  "Action": [
    "sagemaker:CreateTrainingJob",
    "sagemaker:DescribeTrainingJob",
    "sagemaker:ListTrainingJobs",
    "sagemaker:StopTrainingJob"
  ],
  "Resource": "*"
}
```

Dos permisos, dos fallos distintos, y distinguirlos por el mensaje es lo que
el examen evalúa: si falta `sagemaker:CreateTrainingJob`, el error nombra esa
acción; si falta `iam:PassRole`, el error nombra `iam:PassRole` y, sobre todo,
**el recurso que menciona es un rol, no un trabajo**. Ese detalle basta para
diagnosticar sin leer nada más.

### `iam:PassRole` frente a `sts:AssumeRole`

| | `sts:AssumeRole` | `iam:PassRole` |
|---|---|---|
| Qué haces | usar el rol tú mismo | dar el rol a un servicio para que lo use él |
| Quién acaba actuando | tú, con las credenciales del rol | el servicio |
| ¿Es una llamada a la API? | sí, a STS | no, es una comprobación |
| Dónde se concede | política de identidad **y** confianza del rol | política de identidad de quien entrega |
| Error típico | `not authorized to perform: sts:AssumeRole on resource: <rol>` | `not authorized to perform: iam:PassRole on resource: <rol>` |

Regla mnemotécnica: `AssumeRole` te cambia de identidad; `PassRole` no cambia
la tuya, se la da a otro.

---

## La *bucket policy*: el recurso también vota

El equipo de producción necesita leer los artefactos que produce desarrollo.
Ya existe el rol `KananMLReadArtifacts-prod`, pero los artefactos están en un
bucket de la cuenta `111111111111`, y una política de identidad escrita en
producción no puede conceder acceso a un recurso de otra cuenta por sí sola.

La segunda familia de políticas es la que se adjunta **al recurso**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProdPuedeLeerLosArtefactosDeFraude",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::222222222222:role/KananMLReadArtifacts-prod"
      },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1",
        "arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1/fraude/*"
      ]
    },
    {
      "Sid": "SoloPrincipalesDeKanan",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1",
        "arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalAccount": ["111111111111", "222222222222"]
        }
      }
    }
  ]
}
```

```
aws s3api put-bucket-policy \
  --bucket kanan-ml-dev-artifacts-us-east-1 \
  --policy file://politica-bucket-artefactos.json \
  --profile kanan-dev
```

Una **política basada en recurso** se adjunta al recurso en vez de a la
identidad. En S3 se llama *bucket policy*, hay una sola por bucket, y
`put-bucket-policy` la **reemplaza entera**: no añade. Sobrescribir sin leer
antes con `get-bucket-policy` es una forma barata de dejar sin acceso a medio
equipo.

`Principal` — obligatorio aquí, y prohibido en una política de identidad. Tiene
sentido: una política de identidad ya sabe a quién se aplica, porque está
adjunta a él; una política de recurso tiene que decirlo.

`"Principal": "*"` con `Effect: Deny` — "cualquiera". Combinado con
`StringNotEquals` sobre `aws:PrincipalAccount`, el bloque dice: todo lo que no
venga de las dos cuentas de Kanan queda denegado, pase lo que pase. Esto vale
la pena subrayarlo porque sorprende: ese `Deny` también alcanza a un
administrador de la propia cuenta si llamara desde una tercera cuenta, y un
`Deny` mal acotado en una *bucket policy* puede dejar el bucket inaccesible
para todo el mundo, incluido quien lo escribió. El único que puede deshacerlo
entonces es el usuario raíz de la cuenta propietaria.

### Cuándo conviene cada mecanismo

| Situación | Mecanismo |
|---|---|
| Dar acceso a muchos principales de tu propia cuenta | política de identidad |
| Dar acceso a un recurso concreto a otra cuenta | *bucket policy*, o rol asumido |
| Que el acceso quede documentado en el recurso, para auditoría | *bucket policy* |
| Que quien accede actúe con una identidad de tu cuenta | rol asumido |
| Un tercero externo con el que no compartes IAM | rol asumido con `external_id` |

El criterio práctico: si lo que cambia es **quién** entra, la *bucket policy*
escala mal (un bloque por principal) y el rol escala bien (un rol, muchos
principales confiados). Si lo que cambia es **qué recurso**, es al revés.

---

## Cómo decide AWS: deny explícito, allow, deny implícito

Ya hay sobre la mesa políticas de identidad, políticas de confianza, políticas
de recurso y bloques `Deny`. La pregunta que queda es qué ocurre cuando dos de
ellas dicen cosas distintas.

```mermaid
flowchart TD
    A["peticion firmada: principal, accion, recurso, contexto"] --> B{"hay algun Deny explicito aplicable"}
    B -- si --> Z["AccessDenied: with an explicit deny in ..."]
    B -- no --> C{"principal y recurso en la misma cuenta"}
    C -- "misma cuenta" --> D{"Allow en politica de identidad O en politica de recurso"}
    C -- "cuentas distintas" --> E{"Allow en politica de identidad Y en politica de recurso"}
    D -- si --> Y["la operacion se ejecuta"]
    E -- si --> Y
    D -- no --> X["AccessDenied: because no ... policy allows the action"]
    E -- no --> X
```

**Lectura.** Se entra por arriba con la petición firmada y sus cuatro datos:
quién, qué acción, sobre qué recurso, y el contexto (región, prefijo, si hubo
TLS: todo lo que las condiciones pueden mirar). Lo primero que AWS hace no es
buscar un permiso, sino buscar una prohibición: **un `Deny` explícito en
cualquier política aplicable gana siempre**, y ninguna cantidad de `Allow` lo
revierte. Ese es el motivo por el que el bloque `NadaFueraDeTLS` de la sección
de condiciones es efectivo aunque el resto de la política sea generosa, y el
motivo por el que el `Deny` de la *bucket policy* anterior bloquea incluso a
un administrador con permisos totales: `AdministratorAccess` concede
`"Action": "*"` sobre `"Resource": "*"`, que es un `Allow`, y un `Allow` no
vence a un `Deny`.

Si no hay prohibición, hace falta un permiso, y aquí es donde la cuenta
importa. Dentro de una misma cuenta basta con que **una** de las dos familias
conceda: por eso una *bucket policy* que menciona a tu usuario funciona aunque
su política de identidad no diga nada de ese bucket. Entre cuentas distintas
hacen falta **las dos**: la política de identidad en la cuenta que llama y la
política de recurso en la cuenta que posee el recurso. Ese "Y" es la respuesta
a la mitad de las preguntas de examen sobre acceso entre cuentas, y explica
por qué el acceso de producción a los artefactos de desarrollo necesitó tanto
la política del rol en `222222222222` como la *bucket policy* en
`111111111111`.

Si nadie concede, el resultado es la **denegación implícita**: AWS niega por
defecto, sin que exista ningún documento que diga "no". El mensaje entonces es
`because no ... policy allows the action`, que es literalmente eso: no hay
prohibición, hay ausencia de permiso.

Dos filtros más pueden intervenir antes de que un `Allow` surta efecto, y se
nombran aquí porque aparecen en los mensajes de error y en el examen:

- un **permissions boundary** es una política que se adjunta a un usuario o rol
  y que **no concede nada**: fija el techo de lo que sus políticas pueden
  llegar a conceder. Sirve para delegar la creación de identidades sin delegar
  la capacidad de crearlas más poderosas que uno mismo.
- una **service control policy (SCP)** es una política de AWS Organizations
  que fija ese mismo techo para una cuenta entera. Tampoco concede nada. Una
  SCP que no incluya `us-east-1` hace que ninguna identidad de la cuenta pueda
  operar allí, por administrador que sea.

Ninguna de las dos se configura en esta serie; lo que hay que reconocer es su
huella en un rechazo: `because no permissions boundary allows` y
`because no service control policy allows` significan que el permiso que
falta **no está en la política que estás mirando**.

---

## `AmazonSageMakerFullAccess` y el atajo que ya no está

La vía rápida para que un rol de ejecución funcione es adjuntarle una política
gestionada por AWS:

```
aws iam attach-role-policy \
  --role-name KananSageMakerExecutionRole-dev \
  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess \
  --profile kanan-dev
```

Funciona, y para una cuenta de juguete es aceptable. Para Kanan no lo es, y las
razones concretas son mejores que el eslogan de "mínimo privilegio":

- **Alcanza a muchos servicios vecinos.** Además de `sagemaker:*`, concede
  acciones sobre CloudWatch, ECR, Glue, Athena, Redshift, Step Functions,
  Lambda, Secrets Manager, Cognito, EC2 (interfaces de red, grupos de
  seguridad) y más. Un rol de ejecución de un entrenamiento de fraude no
  necesita nada de eso.
- **Su acceso a S3 va por nombre, no por bucket.** Las acciones sobre objetos
  se conceden sobre los patrones `arn:aws:s3:::*SageMaker*`, `*Sagemaker*`,
  `*sagemaker*` y `*aws-glue*`. Dos consecuencias, y las dos malas: concede
  lectura y escritura en **cualquier** bucket de la cuenta cuyo nombre
  contenga "sagemaker", lo haya creado quien lo haya creado; y **no concede
  nada** sobre `kanan-ml-dev-curated-us-east-1`, porque ese nombre no encaja
  con ningún patrón. Adjuntarla y creer que el rol ya puede leer los datos de
  Kanan lleva a un `AccessDenied` desconcertante.
- **Las acciones a nivel de bucket sí van sobre `"Resource": "*"`.**
  `ListBucket`, `GetBucketLocation` y `CreateBucket` se conceden sobre todos
  los buckets de la cuenta.
- **Y lo más grave: incluye `iam:PassRole` sobre `arn:aws:iam::*:role/*`**,
  condicionado a `iam:PassedToService: sagemaker.amazonaws.com`. Es decir,
  quien tenga esta política puede entregar **cualquier rol de la cuenta** a
  SageMaker. Con la sección anterior en la cabeza, eso es la vía de escalada de
  privilegios del apartado de `PassRole`, servida por defecto.

Cuándo es razonable: una cuenta de experimentación desechable, sin datos
reales, mientras se aprende. Cuándo no: cualquier cuenta con PII, es decir,
todas las de Kanan. El camino correcto es el de esta nota: una política
gestionada por el cliente, escrita a partir de lo que el trabajo necesita de
verdad.

> **Servicio que cambió.** SageMaker Role Manager era el atajo que generaba
> roles a partir de *personas* (científico de datos, MLOps, auditor) y
> *actividades*. Hoy **no admite clientes nuevos**: quienes ya lo usaban siguen
> operando, pero no recibe funcionalidad nueva, y las construcciones `Persona`
> y `Activity` de su biblioteca de CDK están marcadas como obsoletas en favor
> de las construcciones estándar de IAM. Los roles que creó siguen siendo roles
> de IAM normales y no hay que migrar nada. Si en el examen aparece Role
> Manager como la forma de generar permisos por persona para SageMaker, esa es
> la respuesta que el temario espera; lo que hoy ocupa su lugar es IAM a mano
> y [[IaC]].

---

## Leer un `AccessDenied` y averiguar qué falta

Los mensajes de rechazo de AWS no son prosa libre: siguen plantillas fijas, y
la coletilla final identifica el tipo de documento que hay que tocar. Esta
tabla es la herramienta de diagnóstico más rentable de la nota.

| Final del mensaje | Qué significa | Dónde se arregla |
|---|---|---|
| `because no identity-based policy allows the <acción> action` | nadie concedió; denegación implícita | política de identidad del principal |
| `with an explicit deny in an identity-based policy: <arn>` | hay un `Deny` y el mensaje nombra la política | ese documento concreto |
| `because no resource-based policy allows the <acción> action` | el recurso no te menciona | política del recurso |
| `with an explicit deny in a resource-based policy` | `Deny` en la política del recurso | *bucket policy* u homóloga |
| `because no permissions boundary allows the <acción> action` | el techo del principal no lo incluye | permissions boundary |
| `with an explicit deny in a service control policy: <arn>` | lo bloquea la organización | fuera de tu cuenta; hay que escalar |
| `because no service control policy allows the <acción> action` | la organización no lo habilita | ídem |

Dos observaciones que ahorran tiempo. La primera: cuando el mensaje nombra un
ARN de política, ya tienes el archivo que editar, y ese caso es el fácil. La
segunda: el mensaje **siempre dice la acción exacta**, y muy a menudo la acción
que falta no es la que creías. Un `aws s3 cp` que falla puede necesitar
`s3:PutObject`, y un `aws s3 sync` necesita además `s3:ListBucket`; el mensaje
lo dice, y leerlo entero evita conceder `s3:*` por frustración.

El orden en que conviene mirar:

1. `aws sts get-caller-identity` — confirma **quién** es el principal. Buena
   parte de los `AccessDenied` son en realidad "estoy usando el perfil
   equivocado".
2. Lee la coletilla del mensaje y elige la fila de la tabla.
3. Si es una denegación implícita sobre una política que juras haber escrito,
   sospecha de tres cosas en este orden: la forma del ARN (`/*` o sin `/*`),
   una `Condition` que la petición no cumple, y las mayúsculas del nombre de la
   acción.
4. Si sigue sin verse, simula.

### Simular antes de tocar nada

```
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::111111111111:user/miguel.reyes \
  --action-names iam:PassRole \
  --resource-arns arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev \
  --profile kanan-dev
```

```json
{
    "EvaluationResults": [
        {
            "EvalActionName": "iam:PassRole",
            "EvalResourceName": "arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev",
            "EvalDecision": "implicitDeny",
            "MatchedStatements": [],
            "MissingContextValues": ["iam:PassedToService"]
        }
    ]
}
```

`simulate-principal-policy` evalúa las políticas de un principal **sin
ejecutar la operación**: no crea nada, no borra nada, y no necesita que el
recurso exista.

`EvalDecision` — tres valores: `allowed`, `implicitDeny` (nadie concedió) y
`explicitDeny` (alguien prohibió). Es la misma distinción de la sección
anterior, ahora como dato.

`MatchedStatements` — los bloques que decidieron, identificados por su `Sid`
cuando lo tienen. Vacío aquí porque ninguna política concede todavía. Cuando
hay varias políticas adjuntas, esta lista responde a "¿cuál de las siete es la
que está mandando?".

`MissingContextValues` — **la línea más útil de la salida**. Dice qué claves de
condición necesitaba evaluar y no tenía. Aparece `iam:PassedToService` porque
la política que escribimos la exige y el simulador no sabe a qué servicio se
iba a entregar el rol. Se le pasa con `--context-entries`:

```
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::111111111111:user/miguel.reyes \
  --action-names iam:PassRole \
  --resource-arns arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev \
  --context-entries "ContextKeyName=iam:PassedToService,ContextKeyValues=sagemaker.amazonaws.com,ContextKeyType=string" \
  --profile kanan-dev
```

Con la política de `PassRole` ya adjunta, `EvalDecision` pasa a `allowed`. La
misma herramienta está en la consola de IAM como *Policy Simulator*, con la
misma información; para trabajo repetido, la CLI se automatiza y la consola no.

Dos límites que conviene saber antes de confiar en el resultado: el simulador
**no evalúa políticas de recurso cuando el origen es un rol**, así que un
acceso entre cuentas por *bucket policy* no se puede comprobar así; y su
veredicto ignora cualquier condición cuyo valor no le hayas dado.

### Recortar lo que ya concediste de más

El problema inverso —una política demasiado ancha que nadie se atreve a
estrechar— tiene herramienta propia: IAM registra qué servicios ha usado
realmente cada identidad y cuándo.

```
aws iam generate-service-last-accessed-details \
  --arn arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev \
  --profile kanan-dev
```

```json
{"JobId": "a1b2c3d4-0000-4444-8888-examplejobid"}
```

```
aws iam get-service-last-accessed-details --job-id a1b2c3d4-0000-4444-8888-examplejobid --profile kanan-dev
```

La llamada es en dos tiempos —una lanza el análisis y devuelve un identificador,
la otra recoge el resultado— porque el análisis recorre meses de actividad y no
cabe en una respuesta síncrona. La respuesta lista, por servicio, la fecha del
último acceso autenticado y quién lo hizo. Un servicio sin `LastAuthenticated`
después de varios meses es un permiso que sobra. Es la forma defendible de
estrechar una política ancha hacia una propia: mirar qué se usó, no adivinar.

Hay una tercera herramienta, **IAM Access Analyzer**, que va un paso más allá:
genera un borrador de política a partir de la actividad registrada de una
identidad durante un periodo, y además señala qué recursos de la cuenta son
accesibles desde fuera de ella. La actividad de la que se alimenta sale del
registro de llamadas de la cuenta, que es [[Observabilidad]]; ahí se ve también
cómo encontrar el evento concreto que produjo un rechazo, con su `RequestId`,
su principal y sus parámetros.

---

## Diccionario de nombres

| Nombre | Qué es en realidad | De dónde viene |
|---|---|---|
| ARN | identificador global y único de un recurso, con campos vacíos para los servicios que no los usan | *Amazon Resource Name* |
| STS | el servicio que reparte credenciales temporales y responde "quién soy" | *Security Token Service* |
| `"Version": "2012-10-17"` | versión del **lenguaje** de políticas, no de tu documento; hoy es la única válida | la fecha en que se fijó esa gramática |
| `Sid` | etiqueta libre de un bloque; aparece en la salida del simulador | *statement identifier* |
| `Principal` con `"...:root"` | la cuenta entera, **no** el usuario raíz | histórico, sin justificación |
| `AKIA...` / `ASIA...` | prefijo de una llave permanente / de una temporal | convención de prefijos de IAM |
| `assumed-role/Rol/Sesion` | forma del ARN mientras usas un rol; servicio `sts`, no `iam` | — |
| `iam:PassRole` | permiso, no operación: nadie lo llama, lo comprueba el servicio receptor | — |
| *trust policy* | responde "quién puede usar este rol", frente a la de permisos, que responde "qué puede hacer" | — |
| *waiter* | bucle de sondeo ya escrito, con `Delay` y `MaxAttempts` | — |

## Recursos del caso creados en esta nota

| Recurso | Nombre | Cuenta |
|---|---|---|
| Grupo | `KananMLEngineers-dev` | 111111111111 |
| Usuarios | `miguel.reyes`, `ana.torres` | 111111111111 |
| Política gestionada | `KananMLEngineerS3-dev` | 111111111111 |
| Rol de ejecución | `KananSageMakerExecutionRole-dev` | 111111111111 |
| Rol entre cuentas | `KananMLReadArtifacts-prod` | 222222222222 |
| *Bucket policy* | sobre `kanan-ml-dev-artifacts-us-east-1` | 111111111111 |
| Perfiles de CLI | `kanan-dev`, `kanan-prod-lectura`, `kanan-prod-admin` | — |

## Punteros

[[Anatomía de un job de SageMaker]] · [[Protección de datos]] · [[Red]] ·
[[IaC]] · [[Observabilidad]]

---

## Preguntas de práctica

### Dificultad media

**1.** Una ingeniera recibe este error al ejecutar
`aws s3api list-objects-v2 --bucket kanan-ml-dev-curated-us-east-1`:

```
User: arn:aws:iam::111111111111:user/ana.torres is not authorized to perform:
s3:ListBucket on resource: "arn:aws:s3:::kanan-ml-dev-curated-us-east-1"
because no identity-based policy allows the s3:ListBucket action
```

Su política contiene un bloque `Allow` con `"Action": "s3:*"` y
`"Resource": "arn:aws:s3:::kanan-ml-dev-curated-us-east-1/*"`. ¿Cuál es la
causa?

A. Falta `s3:GetBucketLocation` en el bloque.
B. El ARN con `/*` designa los objetos; `s3:ListBucket` actúa sobre el bucket y
necesita el ARN sin `/*`.
C. La política no se ha propagado y hay que esperar.
D. Falta un bloque `Deny` explícito que habilite la acción.

<details><summary>Solución</summary>

**B.** `s3:ListBucket` es una acción a nivel de bucket: su recurso es
`arn:aws:s3:::kanan-ml-dev-curated-us-east-1`, sin sufijo. El comodín `s3:*`
concede la acción, pero sobre un recurso que nunca coincide, y el resultado es
una denegación implícita, que es exactamente lo que dice el mensaje.

A falla porque `GetBucketLocation` no interviene en `ListObjectsV2` y, de
hacerlo, el mensaje la nombraría. C falla porque el mensaje describe una
ausencia de permiso, no una condición transitoria. D falla porque un `Deny`
nunca habilita nada: solo prohíbe.
</details>

**2.** Un ML engineer con permiso `sagemaker:CreateTrainingJob` lanza un
entrenamiento pasando `KananSageMakerExecutionRole-dev` y recibe:

```
not authorized to perform: iam:PassRole on resource:
arn:aws:iam::111111111111:role/KananSageMakerExecutionRole-dev
```

¿Qué hay que añadir, y dónde?

A. `sts:AssumeRole` sobre el rol, en la política de identidad del ingeniero.
B. Un bloque que confíe en `sagemaker.amazonaws.com`, en la política de
confianza del rol.
C. `iam:PassRole` sobre el ARN del rol, en la política de identidad del
ingeniero.
D. `iam:PassRole` sobre el ARN del ingeniero, en la política de permisos del
rol de ejecución.

<details><summary>Solución</summary>

**C.** El permiso que falta es el de *entregar* el rol, y se concede a quien
hace la llamada: en su política de identidad, con el ARN del rol en `Resource`.
Conviene además condicionarlo con `iam:PassedToService`.

A falla porque el ingeniero no va a usar el rol, lo va a entregar; quien lo
asume es el servicio. B describe algo necesario que ya existe —el rol se creó
con esa confianza— y cuya ausencia produciría un error distinto, en el momento
en que el servicio intentara asumirlo. D invierte los papeles: la política de
permisos del rol dice qué puede hacer el rol una vez asumido, no quién puede
entregarlo.
</details>

**3.** (Respuesta múltiple: elige **2**.) Un usuario de `kanan-ml-dev` debe
poder asumir el rol `KananMLReadArtifacts-prod` de `kanan-ml-prod`. ¿Qué dos
cosas son imprescindibles?

A. Una política de identidad en `111111111111` que permita `sts:AssumeRole`
sobre el ARN del rol.
B. Una política de confianza en el rol de `222222222222` cuyo `Principal`
incluya la cuenta o el usuario de `111111111111`.
C. Un usuario de IAM con el mismo nombre en `222222222222`.
D. Una *bucket policy* en los buckets de producción que nombre al usuario.
E. Que ambas cuentas pertenezcan a la misma organización.

<details><summary>Solución</summary>

**A y B.** El acceso entre cuentas exige las dos mitades: permiso para salir en
la cuenta de origen y confianza para entrar en la de destino. Faltando A el
error nombra `sts:AssumeRole` con la coletilla de política de identidad;
faltando B, el error apunta al ARN del rol de la otra cuenta.

C es innecesario: el rol no requiere una identidad espejo. D resuelve otro
problema —acceso a objetos concretos sin asumir rol— y no habilita
`AssumeRole`. E es falso: dos cuentas sin relación organizativa pueden
confiarse mutuamente.
</details>

**4.** Un script de mantenimiento corre desde la laptop de un ingeniero y
escribe en producción, aunque él cree haberlo apuntado a desarrollo. El script
usa `boto3.client("s3")` sin más, y en la terminal está exportada
`AWS_ACCESS_KEY_ID` con llaves de producción. ¿Qué lo explica y cuál es la
corrección mínima?

A. boto3 lee siempre el perfil `default`; hay que cambiar el `default`.
B. Las variables de entorno preceden a los archivos de configuración; crear la
sesión con `profile_name` explícito y verificar con `get_caller_identity`.
C. boto3 mezcla ambas fuentes; hay que borrar `~/.aws/credentials`.
D. Es un fallo de permisos; hay que denegar la escritura en producción.

<details><summary>Solución</summary>

**B.** En la cadena de resolución, las variables de entorno van antes que los
archivos de configuración, así que el cliente sin perfil toma las llaves
exportadas. La corrección mínima es una sesión explícita y una comprobación de
identidad al arrancar.

A es falso: el `default` solo se usa si no hay variables de entorno. C es
desproporcionado y no ataca la causa. D describe una mitigación que también
convendría tener, pero el problema es de credenciales, no de permisos: con
llaves de producción legítimas, denegar la escritura rompería a quien sí debe
escribir.
</details>

**5.** (Emparejamiento.) Asocia cada final de mensaje con el documento que hay
que revisar.

| | Mensaje |
|---|---|
| 1 | `because no identity-based policy allows the s3:GetObject action` |
| 2 | `with an explicit deny in a resource-based policy` |
| 3 | `because no permissions boundary allows the s3:GetObject action` |
| 4 | `because no service control policy allows the s3:GetObject action` |

| | Documento |
|---|---|
| W | la política adjunta al bucket |
| X | la política adjunta al usuario, grupo o rol |
| Y | el techo definido en AWS Organizations |
| Z | el techo adjunto a la propia identidad |

<details><summary>Solución</summary>

**1-X, 2-W, 3-Z, 4-Y.**

El par que más se confunde es 3 y 4: un *permissions boundary* se adjunta a una
identidad concreta y lo administra la propia cuenta; una SCP se aplica a la
cuenta entera y se administra desde la organización, fuera del alcance del
ingeniero. En ambos casos el permiso puede estar concedido en la política de
identidad y aun así no surtir efecto, que es lo que hace estos dos mensajes
desconcertantes.
</details>

### Dificultad alta

**6.** (Ordenamiento.) Ordena los pasos para que un ingeniero de
`kanan-ml-dev`, que todavía no tiene ningún permiso de SageMaker, pueda lanzar
un entrenamiento que lea de `kanan-ml-dev-curated-us-east-1` y escriba en
`kanan-ml-dev-artifacts-us-east-1`.

1. Adjuntar al grupo del ingeniero un bloque con `iam:PassRole` sobre el ARN
del rol, condicionado con `iam:PassedToService`.
2. Crear el rol con una política de confianza cuyo `Principal` sea
`{"Service": "sagemaker.amazonaws.com"}`.
3. Adjuntar al grupo del ingeniero `sagemaker:CreateTrainingJob`.
4. Adjuntar al rol una política de permisos con lectura sobre el bucket curado
y escritura sobre el de artefactos.
5. Llamar a `create-training-job` con el ARN del rol.

<details><summary>Solución</summary>

**2 → 4 → 3 → 1 → 5.**

El rol tiene que existir antes de que se le adjunten permisos (2 antes que 4) y
antes de que su ARN se pueda nombrar en el `Resource` de `PassRole` (2 antes
que 1). Los pasos 3 y 1 son intercambiables entre sí, pero ambos van antes del
5, y ahí está el punto: faltando el 3, el error nombra
`sagemaker:CreateTrainingJob`; faltando el 1, nombra `iam:PassRole` **y el
recurso que menciona es un rol**. El orden equivocado que más se ve es 5 antes
que 4: la llamada se acepta, el trabajo arranca y falla más tarde, ya dentro,
cuando el contenedor intenta leer S3 con un rol sin permisos.
</details>

**7.** El responsable de cumplimiento aplica esta *bucket policy* a
`kanan-ml-dev-artifacts-us-east-1`:

```json
{
  "Sid": "SoloDesdeKanan",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1/*",
  "Condition": {
    "StringNotEquals": {"aws:PrincipalAccount": "111111111111"}
  }
}
```

Al día siguiente, `KananMLReadArtifacts-prod` deja de poder leer los
artefactos, y un administrador de `111111111111` con `AdministratorAccess` sí
puede. ¿Cuál es la mejor explicación?

A. `AdministratorAccess` incluye una excepción para políticas de recurso.
B. El `Deny` alcanza a todo principal cuya cuenta no sea `111111111111`; el rol
de producción pertenece a `222222222222`, y para el administrador la condición
no se cumple.
C. Las *bucket policies* no se aplican a roles, solo a usuarios.
D. Falta `s3:ListBucket`, que es lo que el rol de producción usa.

<details><summary>Solución</summary>

**B.** El bloque niega a cualquier principal que no pertenezca a la cuenta de
desarrollo. El rol de producción vive en `222222222222`, así que
`StringNotEquals` se cumple y el `Deny` se aplica; ningún `Allow`, ni en la
política del rol ni en otro bloque del bucket, lo revierte. Para el
administrador de `111111111111` la condición no se cumple, el `Deny` no aplica,
y su `Allow` de identidad basta, porque principal y recurso están en la misma
cuenta.

A es inventado; de hecho un `Deny` de *bucket policy* sí bloquea a un
administrador cuando la condición le alcanza. C es falso. D no explicaría ni
que antes funcionara ni que el administrador sí pueda; además, con `Resource`
terminado en `/*` este bloque ni siquiera evalúa las acciones a nivel de
bucket.
</details>

**8.** (Respuesta múltiple: elige **2**.) Hay que permitir que
`KananMLEngineers-dev` liste y lea **solo** `fraude/2026/` dentro de
`kanan-ml-dev-curated-us-east-1`, sin ver el resto del bucket. ¿Qué dos
afirmaciones son correctas?

A. El recorte de `s3:ListBucket` se consigue con una `Condition` sobre
`s3:prefix`, porque su recurso es el bucket entero.
B. El recorte de `s3:GetObject` se consigue poniendo
`arn:aws:s3:::kanan-ml-dev-curated-us-east-1/fraude/2026/*` en `Resource`.
C. Una `Condition` con `s3:prefix` en el bloque de `s3:GetObject` recorta el
acceso a los objetos de esa carpeta.
D. Basta con poner el prefijo en `Resource` del bloque de `s3:ListBucket`.
E. `s3:prefix` es una clave de condición global, aplicable a cualquier
servicio.

<details><summary>Solución</summary>

**A y B.** Son las dos mitades del mismo recorte, hechas con mecanismos
distintos porque las dos acciones actúan sobre recursos distintos: `ListBucket`
sobre el bucket, y por tanto solo se puede acotar por condición; `GetObject`
sobre los objetos, y por tanto se acota por ARN.

C falla porque `s3:prefix` solo aplica a `ListBucket` (y a
`ListBucketVersions`): en un bloque de `GetObject` la clave no está en el
contexto de la petición, la condición nunca se cumple y el permiso queda
muerto. D falla porque el recurso de `ListBucket` es el bucket; un ARN con
prefijo no coincide nunca. E falla porque `s3:prefix` la aporta S3 —las
globales empiezan por `aws:`—.
</details>

### Mini caso de estudio

Kanan quiere que el pipeline de producción lea los artefactos de modelos que
produce desarrollo. El diseño acordado es: un rol
`KananMLReadArtifacts-prod` en `222222222222`, con una política de permisos que
concede `s3:GetObject` y `s3:ListBucket` sobre
`arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1` y `.../fraude/*`. El bucket
está en `111111111111` y su *bucket policy* está vacía. El rol se asume sin
problemas: `aws sts get-caller-identity` con el perfil correspondiente devuelve
un ARN `assumed-role/KananMLReadArtifacts-prod/...`. Pero toda llamada a S3
devuelve `AccessDenied`.

**9.** ¿Cuál es la causa?

A. La política de permisos del rol necesita también `s3:GetBucketLocation`.
B. Principal y recurso están en cuentas distintas, y en ese caso hacen falta
`Allow` en la política de identidad **y** en la política del recurso; la
*bucket policy* está vacía.
C. El rol no puede usarse para acceder a S3, solo a servicios de la misma
cuenta.
D. Falta `sts:AssumeRole` en la política del rol.

<details><summary>Solución</summary>

**B.** Dentro de una misma cuenta basta con que conceda una de las dos
familias; entre cuentas distintas hacen falta las dos. La política del rol
concede desde el lado de producción, pero el bucket, que es de desarrollo, no
menciona a nadie. El arreglo es una *bucket policy* en
`kanan-ml-dev-artifacts-us-east-1` con `Principal` igual al ARN del rol.

A no explicaría un `AccessDenied` sobre `GetObject`, y el mensaje nombraría esa
otra acción. C es falso. D confunde dos momentos: `AssumeRole` ya funcionó, lo
demuestra el `get-caller-identity` con ARN de rol asumido.
</details>

**10.** Añadida la *bucket policy* correcta, las lecturas de objetos funcionan,
pero `aws s3 ls s3://kanan-ml-dev-artifacts-us-east-1/fraude/` sigue fallando.
La *bucket policy* concede `s3:GetObject` y `s3:ListBucket` con
`"Resource": "arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1/fraude/*"`. ¿Qué
falta?

A. Añadir `arn:aws:s3:::kanan-ml-dev-artifacts-us-east-1` (sin `/*`) al
`Resource`, porque `s3:ListBucket` se concede sobre el bucket.
B. Cambiar `s3:ListBucket` por `s3:ListObjects`.
C. Añadir una `Condition` con `s3:prefix` igual a `fraude/*`.
D. Aumentar `--duration-seconds` al asumir el rol.

<details><summary>Solución</summary>

**A.** Es el mismo error de la pregunta 1, ahora en una política de recurso: el
ARN con `/*` designa objetos, y `s3:ListBucket` necesita el ARN del bucket. Que
`GetObject` funcione y `ls` no es la firma característica de este fallo.

B nombra una acción que no existe en el lenguaje de políticas de IAM: la
operación de la API se llama `ListObjectsV2`, pero el permiso es
`s3:ListBucket`, y esa asimetría es precisamente lo que el distractor explota.
C es una restricción adicional que no concede nada por sí sola; con el
`Resource` mal puesto seguiría fallando. D pertenece a un problema de
autenticación, y el síntoma sería `ExpiredToken`, no `AccessDenied`.
</details>
