---
tema: Credenciales, perfiles y configuración de la AWS CLI y boto3
corte-curricular: Segunda nota del temario. Se supone leída [[Recursos, ARN, principals, acciones y policies en AWS]] y nada más. El lector sabe qué son una cuenta, una región, un servicio, un recurso, un ARN, un principal, una acción y una policy, pero no ha instalado ninguna herramienta ni ha ejecutado una sola llamada a AWS.
prerrequisitos:
  [JSON, línea de comandos, variables de entorno, Python, la nota anterior]
se-introduce-aqui:
  [
    AWS CLI v2,
    botocore,
    boto3,
    client,
    --query,
    formatos de salida,
    credencial,
    aws configure,
    ~/.aws/credentials,
    ~/.aws/config,
    perfil,
    access key ID,
    secret access key,
    firma SigV4,
    credenciales temporales,
    session token,
    IAM Identity Center,
    aws sso login,
    aws login,
    credential_process,
    cadena de proveedores de credenciales,
    región por defecto,
    reintentos,
    Config de botocore,
    Session de boto3,
    resource,
    paginadores,
    waiters,
    ClientError,
    sts get-caller-identity,
  ]
no-se-usa-aqui:
  [
    mecanismo interno de asunción de roles,
    emisión de credenciales con STS (AssumeRole, GetSessionToken),
    PassRole,
    MFA en profundidad,
    federación OIDC,
    Secrets Manager,
    Parameter Store,
    CloudFormation,
    Terraform,
    CDK,
    VPC,
    KMS,
  ]
versiones: AWS CLI v2 ≥ 2.32, boto3 1.43 sobre Python 3.10+. Consultado en septiembre de 2026.
certificacion: MLA-C01
tags: [aws, cli, boto3, credenciales, iam, mla-c01, fundamentos]
---

# Credenciales, perfiles y configuración de la AWS CLI y boto3

La nota anterior explicó quién puede hacer qué dentro de una cuenta de AWS:
principals, acciones, recursos y las policies que los relacionan. Todo aquello
ocurre **dentro** de AWS y da por supuesto que AWS ya sabe quién está llamando.
Esta nota cubre el paso anterior, que ocurre en tu máquina: cómo se instala lo
que habla con AWS, y cómo se le dice con qué identidad hablar.

## 1. Las dos herramientas

### La AWS CLI

La **AWS CLI** (_AWS Command Line Interface_) es un programa de línea de
comandos llamado `aws`. Existe para que puedas ejecutar operaciones de AWS
—crear un recurso, consultarlo, borrarlo— sin abrir el navegador y sin escribir
un programa. Lo que hace con cada comando es traducirlo a una llamada HTTP a la
API del servicio, mandarla e imprimir la respuesta. No mantiene estado entre
invocaciones.

Su forma es invariable:

```
aws <servicio> <operación> [--parámetro valor ...]
```

`<servicio>` es el nombre corto —`s3`, `ec2`, `sagemaker`, `iam`— y
`<operación>` es el nombre de la operación de la API en minúsculas y con
guiones. `DescribeTrainingJob` se escribe `describe-training-job`. La
correspondencia es mecánica y no hay que memorizarla.

Hay dos versiones y no son intercambiables. La **versión 2** es la única que
debe instalarse hoy: se distribuye como un ejecutable que **trae su propio
intérprete de Python dentro**, de modo que no depende de la instalación de
Python del sistema ni la contamina. La **versión 1** entró en modo
mantenimiento el 15 de julio de 2026 —solo correcciones críticas— y pierde el
soporte el 15 de julio de 2027.

```bash
curl -fsSL https://awscli.amazonaws.com/v2/install.sh | bash
aws --version
```

`install.sh` es el instalador recomendado por AWS para Linux y macOS; la
documentación antigua usa en su lugar un `.zip` con `awscli-exe-linux-x86_64`,
que hace lo mismo. `aws --version` responde algo así:

```
aws-cli/2.36.44 Python/3.13.4 Linux/6.8.0-45-generic exe/x86_64
```

Esa línea dice tres cosas: que la herramienta existe, que el `2.` confirma que
no es la versión antigua, y que el Python que aparece es el suyo, el
empaquetado, no el del sistema. Por eso una AWS CLI v2 y un `pip` del sistema
no se estorban nunca.

Un detalle de manejo que ahorra horas: **toda la documentación de la CLI está
dentro de la CLI**. `aws help` lista los servicios, `aws sagemaker help` lista
las operaciones de SageMaker y `aws sagemaker create-training-job help` describe
cada parámetro con su tipo y sus valores admitidos.

### boto3

**boto3** es el SDK oficial de AWS para Python. Existe para el caso en que las
llamadas a AWS son parte de un programa y no el programa entero: expone las
mismas operaciones que la CLI como métodos de objetos Python, de forma que la
respuesta se pueda inspeccionar y usar en el código.

```bash
pip install boto3
```

Requiere Python 3.10 o superior; el soporte de Python 3.9 terminó el 29 de
abril de 2026. Por debajo se apoya en **botocore**, una biblioteca de más bajo
nivel que se instala sola como dependencia. El lector no la invoca casi nunca
—aparece en un `import` cuando hay que ajustar el cliente, y en el nombre de las
excepciones—, pero es la que construye las peticiones HTTP y convierte las
respuestas JSON en diccionarios de Python.

Esto no es trivia: **la AWS CLI v2 también está construida sobre botocore**,
sobre una copia propia empaquetada dentro del ejecutable. De ahí que las dos
herramientas se comporten igual en todo lo que esta nota trata, y que aprender
la configuración de una sea aprender la de la otra. Las diferencias que quedan
son pocas y están señaladas una por una más adelante.

Hay un tercer hecho que explica por qué ninguna de las dos se queda atrás
respecto de AWS. Cada servicio publica su API en un archivo de descripción —un
JSON con operaciones, parámetros y respuestas— y tanto los comandos de la CLI
como los métodos de boto3 **se generan a partir de esos archivos**. Nadie
escribe a mano `create_training_job`. El precio es que la biblioteca hereda las
rarezas de cada API: nombres en `PascalCase` dentro de código Python,
estructuras anidadas profundas, y ninguna validación semántica más allá de la
que haga el servicio al otro lado.

## 2. Qué herramienta para qué problema

Las dos hablan con la misma API, así que la elección no es de capacidad sino de
forma. La misma consulta sobre el estado de un _training job_ de SageMaker,
escrita de las dos maneras:

```bash
aws sagemaker describe-training-job \
    --training-job-name xgb-churn-2026-09-14
```

```python
import boto3

sm = boto3.client("sagemaker", region_name="us-east-1")
respuesta = sm.describe_training_job(
    TrainingJobName="xgb-churn-2026-09-14"
)
```

`boto3.client("sagemaker", ...)` — devuelve un objeto cuyos métodos son, uno a
uno, las operaciones de la API de ese servicio: `describe_training_job`,
`create_training_job`, `list_training_jobs`. El nombre del servicio es el mismo
string que la CLI pone después de `aws`.

`region_name="us-east-1"` — casi todos los servicios son regionales, y sin un
valor aquí la llamada puede fallar antes de salir de la máquina por no saber a
dónde dirigirse.

La correspondencia es exacta: mismo servicio, misma operación, mismo parámetro
con distinta tipografía. Lo que cambia es qué se puede hacer con la respuesta.
La CLI la imprime como JSON y uno queda en el mundo del _shell_: tuberías,
`grep`, `jq`. boto3 la devuelve como diccionario de Python y uno queda en
Python: condicionales sobre campos, `try`/`except`, funciones.

La CLI ofrece un mecanismo para no salir a `jq` en los casos fáciles:

```bash
aws sagemaker describe-training-job \
    --training-job-name xgb-churn-2026-09-14 \
    --query 'TrainingJobStatus' \
    --output text
```

`--query 'TrainingJobStatus'` — acepta una expresión en **JMESPath**, un
lenguaje de consulta sobre JSON, y recorta la respuesta antes de imprimirla.
`--output text` — imprime `Completed` a secas, sin comillas ni llaves, listo
para meter en una variable de _shell_; sin él saldría `"Completed"`, con las
comillas dentro de la variable. `--output` admite además `json` (el valor por
omisión), `table`, `yaml` y `yaml-stream`.

Con eso, el criterio de elección se enuncia sin ambigüedad.

**La CLI gana en lo puntual.** Mirar si algo existe, ver en qué estado está,
crear un recurso una vez, borrar lo que quedó encendido. Un comando de una
línea contra media docena de líneas de Python más el intérprete.

**La CLI gana en la orquestación de archivos.** `aws s3 cp`, `aws s3 sync` y
`aws s3 ls` son comandos de alto nivel que por dentro encadenan muchas
operaciones —trocear un archivo grande, subirlo en partes en paralelo, comparar
fechas para copiar solo lo que cambió—. Subir un dataset de 40 GB es `aws s3
cp`; escribir eso en Python es reimplementar un programa que ya existe.

**boto3 gana en cuanto hay lógica.** «Si el estado es `Failed`, léeme el motivo,
y si el motivo es una cuota, espera y reintenta con otro tipo de instancia» es
un programa. La frontera práctica es reconocible: cuando un _script_ de _shell_
necesita su tercera variable intermedia o su segundo `if` anidado sobre un campo
de JSON, ya debería ser Python.

**boto3 gana cuando AWS es una parte de un programa y no el programa.** Un
entrenamiento que lee de S3, transforma con pandas, ajusta con scikit-learn y
escribe métricas de vuelta no tiene motivo para salir al sistema operativo a
invocar `aws` a mitad de camino: se pierde la estructura de la respuesta, se
pierde el manejo de errores por excepciones y se gana una dependencia del
binario instalado en la máquina.

**Ninguna de las dos gana cuando el problema es infraestructura reproducible.**
Crear recursos que deben existir siempre, idénticos, en tres entornos, y poder
destruirlos y recrearlos, es el terreno de _infrastructure as code_: se declara
el estado deseado y la herramienta calcula las llamadas. Un _script_ de boto3
que crea recursos es imperativo y no sabe qué hacer la segunda vez que se
ejecuta. Es el tema de [[Infrastructure as code en AWS: CloudFormation, CDK y
Terraform]].

Queda una cuarta interfaz: la **consola**, la interfaz web de AWS, donde se
entra con usuario y contraseña desde el navegador. Es insuperable para explorar
un servicio que no conoces y para leer gráficas, y es la peor opción para
cualquier cosa que haya que hacer dos veces, porque no deja rastro
reproducible.

## 3. El primer comando no funciona: de dónde sale una credencial

Con las dos herramientas instaladas, el primer comando que se escriba falla
antes de salir de la máquina:

```bash
aws s3 ls
```

```
Unable to locate credentials. You can configure credentials by running "aws configure".
```

Ni la CLI ni boto3 vienen con nada tuyo dentro. AWS no sabe quién eres: recibe
peticiones de todo internet y la primera cosa que hace con cualquiera de ellas
es comprobar de qué principal viene. Una petición que no lo demuestre se
rechaza sin mirar nada más, y solo cuando lo demuestra pasan a evaluarse las
policies de la nota anterior.

Lo que demuestra esa identidad es una **credencial**: unas cadenas de texto que
AWS te entrega y que la herramienta usa en cada llamada. Tres cosas hay que
saber de ella antes que cualquier otra.

**Vale para toda la cuenta, no para un servicio.** No existe una credencial de
S3 y otra de SageMaker. La misma sirve para hablar con los dos, y lo que cambia
entre uno y otro no es la credencial sino el permiso, que se decide con
policies. Un `AccessDenied` no significa que te falte una credencial distinta.

**La consigues tú, desde la consola.** Entras a la consola con tu usuario y
contraseña, abres IAM, eliges tu usuario, y en la pestaña de credenciales de
seguridad pides una clave de acceso nueva. AWS te muestra entonces dos cadenas
—y solo entonces, porque la segunda no se vuelve a mostrar nunca—:

```
AKIAIOSFODNN7EXAMPLE
wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**Se las das a la herramienta una vez y no vuelves a escribirlas.** Ese es todo
el gesto, y es el que resuelve el fallo de arriba.

### aws configure: dárselas a la herramienta

```bash
aws configure
```

El comando hace cuatro preguntas. Las dos primeras son las cadenas que acabas de
copiar de la consola; la tercera es contra qué región trabajas por omisión; la
cuarta, cómo quieres que se imprima la respuesta.

```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: json
```

A partir de ahí `aws s3 ls` imprime tus _buckets_, y el fragmento de Python de
la sección 2 se ejecuta sin que haya que tocarle una línea: boto3 lee lo mismo
que la CLI acaba de escribir.

### Dónde acabaron: los dos archivos

`aws configure` no guarda nada en un sitio secreto. Reparte las cuatro
respuestas entre dos archivos de texto dentro de un directorio `.aws` en tu
_home_, y saber cuál es cuál es lo que permite luego editarlos a mano, copiarlos
de máquina o diagnosticar por qué la herramienta hace algo inesperado.

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

```ini
# ~/.aws/config
[default]
region = us-east-1
output = json
```

El reparto es el que sugieren los nombres. En `credentials` va lo que
identifica; en `config` va todo lo demás. La separación existe para que el
segundo se pueda compartir, versionar o pegar en un ticket sin peligro mientras
el primero no sale de la máquina. Ambos están en formato INI: secciones entre
corchetes y pares `nombre = valor`.

Una advertencia de higiene que se paga cara si se ignora: `credentials` es un
archivo de texto plano con un secreto dentro. Debe tener permisos restringidos
(`chmod 600`), no debe copiarse a una imagen de contenedor, y `.aws/` merece
estar en el `.gitignore` global de la máquina. Las claves tampoco se pegan en un
cuaderno de Jupyter: un cuaderno guarda lo que se escribió en él, y los
cuadernos se comparten.

### El perfil: varias cuentas en la misma máquina

Esa sección `[default]` que apareció en los dos archivos es un **perfil**: un
conjunto de respuestas con nombre a las preguntas de con qué identidad, contra
qué región y en qué formato. Existe porque casi nadie trabaja contra una sola
cuenta: hay una de desarrollo, una de producción y a menudo una de datos, cada
una con su identidad, y hacen falta a la vez en la misma máquina. Un perfil por
cuenta resuelve eso sin borrar y reescribir el archivo cada vez.

Se crea con el mismo comando, poniéndole nombre:

```bash
aws configure --profile datos-dev
```

Y aquí aparece la única asimetría de formato de la plataforma, que hay que
conocer porque se equivoca todo el mundo una vez: en `credentials` la sección se
llama igual que el perfil, y en `config` lleva delante la palabra `profile`.

```ini
# ~/.aws/credentials          # ~/.aws/config
[default]                     [default]
...                           ...

[datos-dev]                   [profile datos-dev]
...                           ...
```

La excepción es `default`, que va sin prefijo en los dos. La razón es histórica
y no tiene ninguna lógica que ayude a recordarla; lo que sí ayuda es el síntoma:
si escribes `[datos-dev]` en `config` y ejecutas cualquier comando con
`--profile datos-dev`, la CLI responde `ProfileNotFound: The config profile
(datos-dev) could not be found` aunque el perfil esté ahí delante, escrito.

Un perfil se elige de dos maneras. Con `--profile`, que vale para un comando:

```bash
aws sagemaker list-training-jobs --profile datos-dev
```

y con la variable de entorno `AWS_PROFILE`, que vale para todo lo que se ejecute
después en esa terminal, incluidos los programas de Python:

```bash
export AWS_PROFILE=datos-dev
```

Cuando algo no sale como esperabas, el comando que contesta es este:

```bash
aws configure list --profile datos-dev
```

```
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                 datos-dev           manual    --profile
access_key     ****************MPLE shared-credentials-file
secret_key     ****************EKEY shared-credentials-file
    region                  us-east-1      config-file    ~/.aws/config
```

Muestra qué valores está usando la CLI ahora mismo y, en la columna `Location`,
de dónde ha sacado cada uno. Esa columna es el contenido real del comando y
resuelve la mitad de las sorpresas, porque el archivo no es el único sitio de
donde pueden salir. `aws configure list-profiles`, más modesto, solo lista los
perfiles que existen.

## 4. Qué es una credencial por dentro

Las dos cadenas de la sección anterior no viajan a AWS tal cual, y entender qué
viaja en su lugar explica tres errores que de otro modo parecen brujería.

Toda petición a una API de AWS viaja firmada, salvo en el caso deliberado del
acceso anónimo que apareció en la nota anterior. El mecanismo se llama
**Signature Version 4**, abreviado **SigV4**: la herramienta construye una
representación canónica de la petición —método HTTP, ruta, parámetros, ciertas
cabeceras, un resumen del cuerpo y la marca de tiempo—, calcula sobre ella un
código de autenticación con una clave derivada del secreto, de la fecha, de la
región y del servicio, y envía el resultado en una cabecera. El servicio, que
puede derivar la misma clave, repite el cálculo y compara.

De ahí salen los tres hechos prácticos. **El secreto nunca viaja**: lo que viaja
es una firma derivada de él, así que interceptar la petición no revela la clave.
**La firma está atada a esa petición y a ese instante**: no se puede reutilizar,
y si el reloj de tu máquina se desvía varios minutos del de AWS las peticiones
empiezan a rechazarse. **La firma es todo lo que AWS necesita para saber quién
llama**: por eso el único trabajo de configuración es decidir de dónde sale el
material con el que se firma.

Ese material tiene dos formas, y se distinguen a simple vista.

**Las credenciales de larga duración** son el par que ya viste: un **access key
ID**, que identifica la clave y viaja en claro dentro de la petición, y una
**secret access key**, el secreto con el que se firma. Se crean sobre un usuario
IAM y son la única forma que no caduca: valen hasta que alguien las desactive.
El prefijo `AKIA` del identificador las delata. AWS permite como máximo dos
pares activos por usuario, precisamente para poder rotarlos sin cortar el
servicio —se crea el segundo, se despliega, se comprueba y se borra el primero—.

**Las credenciales temporales** son un trío: los dos campos anteriores más un
**session token**, una cadena larga que acompaña a la firma y que el servicio
usa para saber de qué sesión proceden. Caducan, típicamente entre minutos y
horas. Su identificador empieza por `ASIA`, y ese prefijo distingue el trío del
par de un vistazo.

El trío no se teclea a mano: lo produce un servicio y lo renueva la biblioteca.
Cuando aun así aparece escrito en alguna parte, el error clásico del primer día
es copiar solo las dos primeras líneas: sin el session token la firma se rechaza
con un error de autenticación que no dice nada sobre el campo que falta.

> **Caja negra.** Quién emite las credenciales temporales, qué se le pide y
> cuánto duran es el contenido de [[Roles de IAM, STS y credenciales
> temporales]]. Aquí solo se usa su contrato: _existen proveedores que entregan
> a la biblioteca un trío con fecha de caducidad, y la biblioteca pide otro
> antes de que expire_. Eso basta para toda esta nota.

### Por qué el par permanente es el problema

El par `AKIA` que configuraste en la sección 3 es el camino más corto para
empezar y el que AWS desaconseja para seguir. Es un secreto que no caduca, que
concede exactamente los permisos de su dueño y que funciona desde cualquier
punto de internet. Esas tres propiedades juntas lo convierten en el activo más
peligroso de una cuenta: cuando uno aparece en un repositorio público, en un
cuaderno subido a GitHub, en una imagen de contenedor o en una captura de
pantalla, no hay ventana de exposición limitada, sirve hasta que alguien se dé
cuenta. La minería de criptomonedas a costa de cuentas ajenas vive de eso.

De ahí la postura oficial de AWS, que es también la del examen: **las
credenciales de larga duración son el último recurso**. Para personas,
identidades federadas que producen credenciales temporales. Para código que
corre dentro de AWS, roles, y ninguna credencial en ninguna parte. Un par `AKIA`
solo se justifica cuando ninguna de las dos cosas es posible, y entonces se
rota. Las dos secciones siguientes son, respectivamente, cómo se deja de tener
claves en el archivo y cómo decide la herramienta de dónde las toma.

Un caso particular que el examen premia si se reconoce: **el usuario raíz no
debe tener claves de acceso en absoluto**. Si las tiene, se borran. No hay
ningún escenario legítimo en el que la respuesta correcta sea crear una clave de
acceso para la cuenta raíz.

## 5. Perfiles que no contienen claves

Si el par permanente es el problema, la salida no es dejar de usar perfiles: es
usar perfiles que **describen cómo conseguir una credencial** en lugar de
contenerla. El archivo deja de guardar un secreto y pasa a guardar una receta;
quien lo lea sin permiso no encuentra nada aprovechable, y las credenciales que
la receta produce caducan solas.

Son cuatro formas, todas viven en `~/.aws/config`, y se eligen por quién eres y
qué tiene montado tu organización.

### IAM Identity Center

**AWS IAM Identity Center** es el servicio con el que una empresa da acceso a
sus empleados: un portal web único donde cada uno entra con las credenciales
corporativas y elige a qué cuenta de AWS y con qué rol quiere trabajar. Lo monta
el equipo de plataforma, no tú; lo que a ti te llega es la dirección del portal.
Es la forma recomendada por AWS para el acceso humano y sustituye a la práctica
de crear un usuario IAM por persona en cada cuenta.

Con esa dirección en la mano, el asistente hace el resto:

```bash
aws configure sso
```

```
SSO session name (Recommended): mi-empresa
SSO start URL [None]: https://mi-empresa.awsapps.com/start
SSO region [None]: us-east-1
SSO registration scopes [None]: sso:account:access
```

Abre el navegador, te autentica, lista las cuentas y los roles a los que tienes
acceso, y escribe en `~/.aws/config`:

```ini
[sso-session mi-empresa]
sso_start_url = https://mi-empresa.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access

[profile datos-dev]
sso_session = mi-empresa
sso_account_id = 123456789012
sso_role_name = DataScientist
region = us-east-1
output = json
```

Merece la pena comparar esto con el archivo de la sección 3 y ver lo que ya
**no** hay: ni `aws_access_key_id`, ni `aws_secret_access_key`, ni ningún otro
secreto. `[sso-session ...]` es un tipo de sección distinto de un perfil:
describe el portal una vez y se comparte entre todos los perfiles que la nombren
con `sso_session`. Cada perfil dice únicamente en qué cuenta y con qué rol
quiere trabajar.

A cambio, el uso diario tiene un gesto más:

```bash
aws sso login --profile datos-dev
```

Abre el navegador, te autentica y deja un testigo en `~/.aws/sso/cache/`. A
partir de ahí, todos los comandos y todos los programas de Python que usen ese
perfil obtienen credenciales sin intervención. Cuando el testigo caduca —horas o
días, según lo configure la organización— el siguiente comando falla pidiendo
que vuelvas a entrar, y se repite el `aws sso login`. `aws sso logout` borra el
testigo.

Que haya que volver a entrar cada cierto tiempo no es una molestia del diseño:
es el diseño. Un portátil robado con un perfil de Identity Center caduca solo.

### aws login

Si trabajas en una cuenta propia o pequeña, sin Identity Center montado, desde
la versión 2.32 de la CLI hay una alternativa que no exige infraestructura
ninguna y que sirve para desarrollo local:

```bash
aws login
```

Abre el navegador y te autentica con las mismas credenciales con las que
entrarías a la consola —el usuario y la contraseña que ya usaste en la sección 3
para crear la clave de acceso—. El perfil que deja escrito es este:

```ini
[default]
login_session = arn:aws:iam::123456789012:user/ana
region = us-east-1
```

Otra vez, ningún secreto en el archivo: solo el ARN de la identidad con la que
se entró. Las credenciales que produce son temporales, la sesión dura como
máximo doce horas, se guardan en `~/.aws/login/cache` y la CLI las renueva sola
mientras siga viva. `aws logout` la cierra.

Dos condiciones. La identidad que entra necesita tener adjunta la _AWS managed
policy_ `SignInLocalDevelopmentAccess` —una de esas policies mantenidas por AWS
de la nota anterior—, salvo el usuario raíz, que no la necesita y que de todas
formas no debería usarse para esto. Y si la organización tiene Identity Center,
lo correcto es `aws configure sso`, no esto.

### Un perfil que asume un rol

Las dos formas anteriores resuelven cómo entras tú. Esta resuelve otra cosa: que
una vez dentro pases a actuar con una identidad distinta, que es lo que ocurre
cuando tu cuenta de trabajo no es la cuenta donde están los datos. El perfil
dice de dónde salir y a qué rol pasar:

```ini
[profile base]
region = us-east-1

[profile produccion]
role_arn = arn:aws:iam::210987654321:role/AnalistaDeDatos
source_profile = base
region = us-east-1
```

`role_arn` es el ARN del rol de destino —la notación de la nota anterior— y
`source_profile` nombra el perfil cuyas credenciales se usan para pedir el
cambio. Al invocar cualquier cosa con `--profile produccion`, la biblioteca
resuelve primero las credenciales de `base`, con ellas obtiene credenciales
temporales del rol, y firma con estas últimas. El lector no hace nada: escribe
`--profile produccion` y ya está. Qué ocurre exactamente en ese cambio de
identidad es [[Roles de IAM, STS y credenciales temporales]].

Hay variantes que se ven en archivos ajenos: `credential_source` en vez de
`source_profile` cuando las credenciales de partida no vienen de otro perfil
sino del entorno o de la propia máquina; `mfa_serial` para exigir un segundo
factor al asumir; `duration_seconds` para pedir otra duración;
`role_session_name` para que quede etiquetado en los registros quién asumió.

### Un programa externo

La última forma existe para las organizaciones que ya tienen su propio sistema
de identidad o su bóveda de secretos y no quieren pasar por ninguna de las tres
anteriores. El perfil no dice cómo conseguir la credencial: dice qué programa
preguntárselo.

```ini
[profile corporativo]
credential_process = /usr/local/bin/mis-credenciales --entorno prod
region = us-east-1
```

Ese programa lo escribe la empresa, y la biblioteca lo ejecuta cada vez que
necesita credenciales. El contrato es que escriba en su salida estándar un JSON
con esta forma exacta:

```json
{
  "Version": 1,
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "SessionToken": "...",
  "Expiration": "2026-09-14T21:00:00Z"
}
```

`Version` vale siempre `1`. `SessionToken` se omite si las credenciales son
permanentes. `Expiration`, en formato RFC 3339, es lo que le dice a la
biblioteca cuándo volver a ejecutar el programa: **sin `Expiration` las
credenciales se tratan como permanentes y no se renuevan nunca**, que es el
fallo silencioso típico de una integración casera —funciona una hora y luego
empieza a devolver errores de token expirado sin que nada haya cambiado—.

Tiene una implicación de seguridad que la documentación subraya: ese programa se
ejecuta con tus privilegios cada vez que algo necesita credenciales, de modo que
quien pueda escribir en tu `~/.aws/config` puede ejecutar lo que quiera en tu
máquina.

### El patrón común

Las cuatro comparten estructura: el perfil no guarda un secreto, guarda una
receta, y el resultado de aplicarla son credenciales temporales que la
biblioteca renueva. Esa es la razón de fondo por la que un programa bien escrito
para AWS **nunca menciona credenciales**: si el entorno está bien configurado,
`boto3.client("s3")` funciona, y si mañana ese entorno cambia de Identity Center
a un rol asumido, el programa no cambia ni una línea.

| Ajuste en `~/.aws/config` | Qué contiene en realidad | De dónde viene el nombre |
|---|---|---|
| `sso_session` | El nombre de la sección `[sso-session ...]` que describe el portal, no una sesión abierta | Del servicio IAM Identity Center, antes llamado AWS SSO |
| `sso_account_id`, `sso_role_name` | Qué cuenta y qué rol elegir de los que el portal ofrezca | — |
| `login_session` | El ARN de la identidad con la que se entró por navegador; sirve de etiqueta, no de credencial | De `aws login` |
| `role_arn` + `source_profile` | El rol al que pasar y el perfil del que salir | — |
| `credential_process` | La ruta de un ejecutable que imprime un JSON de credenciales | Del proceso hijo que la biblioteca lanza |

## 6. La cadena de proveedores de credenciales

A estas alturas las credenciales pueden estar en bastantes sitios: escritas en
el código, en variables de entorno, en los dos archivos de la sección 3, detrás
de cualquiera de las cuatro recetas de la sección 5, o en ninguna parte porque
la máquina las recibe sola. Falta la regla que decide cuál gana cuando hay más
de una, que es exactamente lo que ocurre en toda máquina que se lleva usando
unos meses.

La regla es que **se recorre una lista fija de proveedores en orden y se usa el
primero que devuelva algo**. No se combinan, no se toma el identificador de un
sitio y el secreto de otro, no hay heurística: el primero que conteste gana y el
recorrido termina. La lista documentada para boto3:

1. Credenciales pasadas como argumentos en el código de boto3, al construir el
   cliente.
2. Variables de entorno: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y
   `AWS_SESSION_TOKEN`.
3. Perfil que asume un rol (`role_arn` con `source_profile` o
   `credential_source`).
4. Perfil que asume un rol presentando el testigo de un proveedor de identidad
   externo; el cómo es [[Federación de identidades en AWS]].
5. Perfil de IAM Identity Center (`sso_session`, `sso_account_id`,
   `sso_role_name`).
6. El archivo `~/.aws/credentials`.
7. Sesión iniciada con `aws login` (`login_session`).
8. El archivo `~/.aws/config`.
9. Los archivos heredados de boto2 (`~/.boto`), que solo existen por
   compatibilidad.
10. Credenciales de contenedor, servidas por una dirección local dentro de ECS y
    EKS.
11. Credenciales de instancia, servidas por el servicio de metadatos de una
    máquina virtual de EC2.

La de la AWS CLI es casi la misma, con la diferencia natural de que arriba del
todo están las opciones de línea de comandos —`--profile` y las demás— en lugar
de los argumentos de Python, y con `credential_process` situado entre el archivo
de credenciales y el de configuración.

Los dos extremos explican todo lo demás.

**El extremo de arriba es un antipatrón.** Que se puedan pasar claves como
argumentos no significa que deba hacerse:

```python
# No hagas esto.
s3 = boto3.client(
    "s3",
    aws_access_key_id="AKIAIOSFODNN7EXAMPLE",
    aws_secret_access_key="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
)
```

Ese código pone un secreto permanente en el archivo fuente, que es lo que acaba
en el repositorio; ata el programa a una identidad concreta, de modo que no se
puede mover a otro entorno sin editarlo; e impide cualquier renovación
automática. Existe para casos legítimos y raros —una herramienta que gestiona
credenciales de terceros, un test contra un servicio simulado— y para casi todo
lo demás es el error que el examen espera que reconozcas.

**El extremo de abajo es la respuesta correcta casi siempre.** Los dos últimos
proveedores no leen ningún archivo: preguntan a la infraestructura. Una máquina
de EC2 a la que se le ha asociado un rol expone, en una dirección local a la que
solo se llega desde dentro de la propia máquina, un pequeño servicio que entrega
credenciales temporales de ese rol; un contenedor en ECS o EKS recibe lo mismo
por un endpoint equivalente cuya dirección le llega en una variable de entorno.
La biblioteca las encuentra sola, las usa y las renueva, y en la máquina no hay
ningún archivo con secretos que robar.

Por eso la frase operativa de toda la nota es: **si el código corre dentro de
AWS, no configures credenciales**. Asóciale un rol al recurso de cómputo y deja
que la cadena haga su trabajo.

```mermaid
flowchart TD
    A["boto3 o la CLI necesitan firmar una petición"] --> B{"¿Credenciales en el código<br/>o en la línea de comandos?"}
    B -->|sí| Z["Se usan y se acaba la búsqueda"]
    B -->|no| C{"¿Variables de entorno<br/>AWS_ACCESS_KEY_ID...?"}
    C -->|sí| Z
    C -->|no| D{"¿El perfil activo describe<br/>una receta: rol, SSO, proceso?"}
    D -->|sí| E["Se ejecuta la receta<br/>y se obtienen credenciales temporales"]
    E --> Z
    D -->|no| F{"¿Hay claves en<br/>~/.aws/credentials o ~/.aws/config?"}
    F -->|sí| Z
    F -->|no| G{"¿Estamos dentro de AWS?<br/>endpoint local de contenedor o de instancia"}
    G -->|sí| H["Credenciales temporales del rol<br/>asociado al cómputo"]
    H --> Z
    G -->|no| I["Unable to locate credentials"]
```

**Lectura.** El dibujo es una cascada, no un árbol de decisión inteligente: cada
rombo pregunta lo mismo —«¿hay algo aquí?»— y la primera respuesta afirmativa
termina el recorrido. Las dos cajas que producen credenciales temporales, la
receta del perfil y la dirección local de la infraestructura, son las únicas
deseables; las que devuelven claves escritas a mano están más arriba
precisamente porque son explícitas, y esa prioridad causa el error más
frustrante de todos: un `export AWS_ACCESS_KEY_ID=...` olvidado en la terminal
gana a cualquier perfil, por bien configurado que esté, y `--profile produccion`
se ignora en silencio. El nodo final es el mensaje con el que empezó la sección
3, y ahora se lee como lo que es: no «no tengo permiso», sino «no he encontrado
con qué identidad hablar».

Nótese que toda esta cadena es **anterior** a la nota previa. Primero se resuelve
qué credencial se usa, después se firma, después AWS identifica al principal y
solo entonces evalúa las policies. `Unable to locate credentials` y
`AccessDenied` están separados por tres pasos: el primero es un problema de tu
máquina, el segundo es un problema de permisos en la cuenta.

## 7. La configuración que no son credenciales

De las cuatro preguntas que hizo `aws configure`, dos eran la identidad y las
otras dos no. Esta sección es esas otras dos, más un tercer ajuste que no
pregunta nadie y que decide si tus trabajos largos terminan.

### La región

Un recurso creado en una región no existe en otra, así que la herramienta
necesita saber a qué región dirigirse antes de poder mandar nada. Lo resuelve
por precedencia, de más fuerte a más débil: lo que se diga en la propia llamada
—`--region` en la CLI, `region_name` en boto3—, luego la variable de entorno,
luego el ajuste `region` del perfil activo. Si nada de eso existe, la llamada
falla antes de salir de la máquina con `NoRegionError: You must specify a
region.`

Aquí hay una divergencia entre las dos herramientas que produce un fallo
silencioso. **La AWS CLI v2 reconoce `AWS_REGION` y `AWS_DEFAULT_REGION`**, y la
primera gana sobre la segunda. **boto3 documenta solo `AWS_DEFAULT_REGION`**. La
consecuencia concreta: en una terminal con `export AWS_REGION=eu-west-1`, `aws
s3 ls` lista los _buckets_ de Irlanda y el script de Python de al lado sigue
hablando con la región del perfil. Dos costumbres lo evitan: exportar las dos
variables, y ser explícito en el código.

```python
sm = boto3.client("sagemaker", region_name="us-east-1")
```

Un matiz que ahorra perplejidad: los servicios globales —IAM entre ellos— no
fallan por falta de región, porque no la usan. Un programa puede funcionar
perfectamente con IAM y romperse en la primera línea que toque SageMaker.

### La salida y las consultas

La cuarta pregunta de `aws configure` quedó guardada como `output`, y se puede
cambiar por comando con `--output` o por terminal con `AWS_DEFAULT_OUTPUT`.
`table` es para leer con los ojos; `text` es para leer con un _script_; `json` es
para pasarlo a otro programa.

`--query` filtra **del lado del cliente**: la respuesta completa llega
igualmente por la red y la CLI la recorta antes de imprimirla. Importa cuando se
listan miles de elementos, porque no ahorra ni tráfico ni tiempo de servicio;
para eso están los parámetros de filtrado que cada operación ofrezca.

### Los reintentos

Una llamada puede fallar por motivos transitorios: un error de red, un 500 del
servicio o, muy habitualmente en cargas de machine learning, un _throttling_ —el
servicio rechaza la petición porque estás llamando demasiado rápido—. Las
bibliotecas reintentan solas, con espera exponencial, y el comportamiento se
configura con `retry_mode` y `max_attempts`.

Hay tres modos. `legacy` es el antiguo, con una lista corta de errores que
considera reintentables. `standard` es el moderno: más tipos de error
reconocidos, incluidos los de _throttling_, y comportamiento consistente con el
resto de SDKs. `adaptive` añade a `standard` un limitador de ritmo del lado del
cliente que se adapta a las respuestas del servicio; es experimental.

Y aquí la asimetría que importa: **la AWS CLI v2 usa `standard` por omisión y
boto3 sigue usando `legacy`** (AWS ha anunciado el cambio de boto3 a `standard`
para finales de 2026). El mismo trabajo escrito en Python reintenta peor que
escrito en la terminal, salvo que se le pida lo contrario:

```python
import boto3
from botocore.config import Config

config = Config(
    region_name="us-east-1",
    retries={"total_max_attempts": 10, "mode": "standard"},
)

sm = boto3.client("sagemaker", config=config)
```

`from botocore.config import Config` — la ruta sorprende: el objeto vive en
botocore, la capa de abajo, no en boto3. Es el sitio donde se ajusta todo lo que
es comportamiento del cliente y no parámetro de la operación: reintentos,
tiempos de espera, número de conexiones simultáneas, _proxies_, opciones
específicas de S3.

`retries={"total_max_attempts": 10, ...}` — dentro de `Config`, el campo
`max_attempts` cuenta **reintentos**, sin contar la llamada inicial, mientras
que el ajuste homónimo del archivo de configuración y la variable
`AWS_MAX_ATTEMPTS` cuentan **intentos totales**. Es una trampa real: los mismos
diez en un sitio y en otro significan diez y once. `total_max_attempts` cuenta
siempre el total y elimina la ambigüedad.

El equivalente en el archivo, que afecta a las dos herramientas a la vez:

```ini
[profile datos-dev]
region = us-east-1
retry_mode = standard
max_attempts = 10
```

Para trabajo de machine learning esto no es cosmética. Un _script_ que lanza
cientos de llamadas a SageMaker o que lista un _bucket_ con millones de objetos
se encuentra con _throttling_ antes o después, y la diferencia entre `legacy` y
`standard` es la diferencia entre un pipeline que falla a mitad y uno que no se
entera.

## 8. boto3 por dentro: sesión, cliente y lo que hay alrededor

La sección 2 mostró `boto3.client("sagemaker")` y nada más. Eso basta para un
programa que habla con una cuenta; en cuanto hay dos cuentas, listas largas o
esperas, aparecen cuatro piezas más que están en todo el código de AWS que vas a
leer.

### La sesión

Una **Session** es el objeto que guarda una configuración resuelta: un perfil,
una región y unas credenciales. Todo cliente nace de una sesión; si no se crea
ninguna explícitamente, boto3 crea y reutiliza una por detrás —la _sesión por
omisión_—, y por eso la forma corta de la sección 2 funciona.

Crearla a mano sirve para lo que `--profile` hace en la terminal, con la ventaja
de que dos sesiones pueden vivir a la vez en el mismo proceso:

```python
import boto3

dev = boto3.Session(profile_name="datos-dev")
prod = boto3.Session(profile_name="produccion")

s3_dev = dev.client("s3")
s3_prod = prod.client("s3")
```

`boto3.Session(profile_name=...)` — es la forma de elegir perfil desde Python,
equivalente a `--profile`. Nota que los clientes se piden a la sesión
(`dev.client`), no a `boto3`: pedírselo a `boto3` devolvería un cliente de la
sesión por omisión y las dos identidades serían la misma sin que nada avisara.

Ese fragmento es la respuesta a un problema común —copiar datos entre cuentas,
comparar dos entornos— y la demostración de para qué sirven los perfiles: dos
identidades distintas, vivas a la vez, sin una sola clave en el código.
`Session` acepta además los mismos argumentos de credenciales que el cliente, y
ocupa por tanto el mismo escalón de arriba de la cadena de la sección 6, con el
mismo desaconsejo.

La sesión expone lo que resolvió, que es oro puro para diagnosticar:

```python
sesion = boto3.Session(profile_name="datos-dev")

print(sesion.region_name)          # 'us-east-1'
print(sesion.available_profiles)   # ['default', 'datos-dev', 'produccion']

credenciales = sesion.get_credentials()
print(credenciales.method)         # 'sso', 'env', 'iam-role', ...
print(credenciales.access_key)     # el ID; el secreto no se imprime nunca
```

`credenciales.method` — es el nombre del proveedor de la sección 6 que ganó la
carrera. Cuando un programa hace algo inesperado, esa línea contesta la pregunta
correcta en un segundo.

### El otro cliente

Al leer código ajeno aparecerá una segunda interfaz, el **resource**, que
envuelve al cliente en objetos con aire de dominio:

```python
s3 = boto3.resource("s3")
bucket = s3.Bucket("datos-ml")
for objeto in bucket.objects.filter(Prefix="crudos/"):
    print(objeto.key, objeto.size)
```

Se lee mejor, no hay discusión. Pero su estado oficial es explícito: AWS **no va
a añadir características nuevas** a la interfaz de _resources_; las existentes
seguirán funcionando durante la vida de boto3, y para las capacidades nuevas de
cualquier servicio hay que usar el _client_. En la práctica: reconócelo al leer,
y escribe _client_ al escribir. Además, la mayoría de los servicios modernos
—SageMaker entre ellos— nunca tuvieron _resource_.

### Paginadores

Las operaciones que listan cosas no devuelven todo: devuelven una página y un
testigo para pedir la siguiente. Escribir ese bucle a mano es tedioso y se hace
mal, así que boto3 lo trae hecho:

```python
sm = boto3.client("sagemaker", region_name="us-east-1")

paginador = sm.get_paginator("list_training_jobs")
for pagina in paginador.paginate(StatusEquals="Completed"):
    for job in pagina["TrainingJobSummaries"]:
        print(job["TrainingJobName"], job["TrainingEndTime"])
```

`sm.get_paginator("list_training_jobs")` — el argumento es el nombre del método
en Python, con guiones bajos, no el de la CLI. `paginador.paginate(...)` recibe
los mismos parámetros que la operación original y devuelve un iterable de
páginas; el testigo de continuación lo gestiona él y corta cuando no hay más.
Sin paginador, `sm.list_training_jobs()` devuelve solo la primera página y el
programa parece funcionar: es el fallo silencioso clásico, un informe que
siempre reporta el mismo centenar de trabajos.

En la CLI el recorrido es automático —`aws sagemaker list-training-jobs` ya pasa
por todas las páginas— y se puede desactivar con `--no-paginate` o limitar con
`--max-items`. Es el punto donde la comparación de la sección 2 se decide por el
tamaño del problema: para mirar, la CLI; para procesar diez mil elementos con
lógica, el paginador.

### Waiters

Casi todas las operaciones interesantes de machine learning son asíncronas:
`CreateTrainingJob` devuelve en cuanto el trabajo se ha aceptado, no cuando ha
terminado de entrenar. Escribir el bucle de sondeo a mano —con su `sleep`, su
espera creciente y su límite de intentos— es innecesario:

```python
esperador = sm.get_waiter("training_job_completed_or_stopped")
esperador.wait(TrainingJobName="xgb-churn-2026-09-14")
```

`get_waiter("training_job_completed_or_stopped")` — el nombre dice los dos
estados finales que dan por terminada la espera, y eso es deliberado: un
_waiter_ que solo esperase `Completed` se quedaría colgado para siempre con un
trabajo detenido. `client.waiter_names` lista los que tiene cualquier cliente;
SageMaker ofrece también `processing_job_completed_or_stopped`,
`transform_job_completed_or_stopped`, `endpoint_in_service` y
`endpoint_deleted`. La CLI expone lo mismo como `aws sagemaker wait
training-job-completed-or-stopped --training-job-name ...`.

Un _waiter_ no es magia: por dentro sondea con un intervalo y un número máximo
de intentos, y falla si se agotan. Pero está afinado por AWS para cada operación
y no se olvida de manejar los estados de error.

### Errores

Los errores del servicio llegan como excepciones, y distinguirlos es la mitad de
por qué se escribe esto en Python y no en _shell_:

```python
import botocore.exceptions

try:
    sm.describe_training_job(TrainingJobName="no-existe")
except botocore.exceptions.ClientError as error:
    codigo = error.response["Error"]["Code"]
    mensaje = error.response["Error"]["Message"]
    print(codigo, "|", mensaje)
```

`ClientError` envuelve cualquier error devuelto por una API, así que una sola
cláusula `except` los atrapa todos y hay que mirar dentro para decidir.
`error.response["Error"]["Code"]` es una cadena corta y estable
—`ValidationException`, `ResourceNotFound`, `AccessDeniedException`,
`ThrottlingException`— y es lo que hay que comparar, nunca el texto de
`Message`, que cambia entre versiones del servicio.

## 9. Dónde corre el código: cuatro entornos y una regla

Todo lo anterior describe una máquina: la tuya. Pero el mismo programa acaba
ejecutándose en sitios donde no hay nadie para teclear `aws configure`, y la
respuesta correcta cambia en cada uno.

**Tu portátil.** Es el único caso donde hay algo que configurar. Si la
organización tiene IAM Identity Center, `aws configure sso` y un perfil por
cuenta; si no lo tiene, `aws login`; y solo si ninguna de las dos es posible, el
par permanente de la sección 3, con la obligación de rotarlo. En los tres casos
el resultado es un perfil con nombre y la costumbre de trabajar con `--profile`
o `AWS_PROFILE` en vez de con un `default` ambiguo, que es lo que impide el
accidente clásico de ejecutar en producción lo que se creía estar ejecutando en
desarrollo.

**Un servidor o un sistema de integración continua fuera de AWS.** No hay
navegador ni persona que autentique, así que las tres primeras recetas no
sirven. Lo correcto es que el sistema externo presente una identidad propia y la
cambie por credenciales temporales de un rol de tu cuenta, sin ninguna clave
permanente en ningún sitio: es el proveedor número 4 de la sección 6, y el cómo
pertenece a [[Federación de identidades en AWS]]. Si eso no es posible, un par
permanente guardado en el gestor de secretos de ese sistema —nunca en el
repositorio, nunca en el archivo de definición del _pipeline_— y rotado.

**Cómputo dentro de AWS: una máquina de EC2, un contenedor, una función de
Lambda.** No se configura nada. Se le asocia un rol al recurso y la cadena
encuentra credenciales temporales por sí sola en los dos últimos proveedores.
Cualquier respuesta de examen que proponga copiar claves a una instancia de EC2
es incorrecta por construcción, y el enunciado suele delatarse solo: cuando
aparecen «sin almacenar credenciales», «rotación automática» o «mínimo
privilegio», está pidiendo un rol.

**SageMaker.** Funciona igual que el anterior, con vocabulario propio. Todo lo
que SageMaker ejecuta por ti lleva asociado un rol: la instancia de cuaderno, el
espacio de SageMaker Studio, el contenedor que entrena, el que procesa datos y
el que sirve un _endpoint_ de inferencia. Ese rol es el **execution role** que ya
apareció en la nota anterior, y el entorno expone una dirección local que
entrega credenciales temporales suyas. La consecuencia sorprende al llegar de
otro entorno: en un cuaderno de SageMaker esto funciona sin haber ejecutado
`aws configure`, sin archivos en `~/.aws` y sin variables de entorno.

```python
import boto3

s3 = boto3.client("s3")
s3.download_file("datos-ml", "crudos/2026/train.parquet", "train.parquet")
```

No hay `region_name` y tampoco hace falta: el entorno viene con la región del
propio dominio o de la instancia ya puesta.

En ese mismo entorno aparece una biblioteca más que conviene situar, porque se
confunde con boto3 constantemente. El **SageMaker Python SDK** —`pip install
sagemaker`— está construido _encima_ de boto3 y ofrece abstracciones de alto
nivel propias del oficio: estimadores, _predictors_, _pipelines_. boto3 te deja
llamar a `create_training_job` con sus cuarenta parámetros; el SageMaker Python
SDK te deja describir un entrenamiento en unas pocas líneas. No compiten. En
esta nota importa por una sola función:

```python
import sagemaker

rol = sagemaker.get_execution_role()
print(rol)   # arn:aws:iam::123456789012:role/service-role/AmazonSageMaker-...
```

`get_execution_role()` devuelve el ARN del rol con el que está corriendo el
entorno actual, preguntándoselo al propio entorno. Dentro de SageMaker funciona;
en tu portátil no tiene a quién preguntar y falla. Esa asimetría es el origen
del error más repetido por quien empieza: un cuaderno copiado de un tutorial que
se ejecuta en local y muere en la tercera línea. La solución no es pelearse con
la función, sino escribir el ARN del rol explícitamente cuando se trabaja fuera.

De ahí la regla que resume la sección: **las credenciales se configuran en un
único sitio, el portátil de una persona. En todo lo demás se asocian roles.** Si
un diseño te obliga a guardar claves en algo que corre dentro de AWS, el diseño
está mal.

```mermaid
flowchart LR
    subgraph Fuera["Fuera de AWS"]
        P["Portátil<br/>perfil con receta:<br/>SSO, aws login, rol"]
        CI["CI externo<br/>identidad federada"]
    end
    subgraph Dentro["Dentro de AWS"]
        E["EC2 / contenedor / Lambda<br/>rol asociado al cómputo"]
        SM["SageMaker<br/>execution role"]
    end
    P --> API["APIs de AWS<br/>peticiones firmadas"]
    CI --> API
    E --> API
    SM --> API
    API --> EV["Evaluación de policies<br/>de la nota anterior"]
```

**Lectura.** Las cuatro cajas de la izquierda son las cuatro situaciones, y lo
que tienen en común es lo que importa: todas terminan produciendo una petición
firmada con credenciales temporales, y solo las dos de fuera de AWS han
necesitado que alguien configure algo. La flecha final es la frontera entre esta
nota y la anterior: hasta el nodo `API` el problema es _quién eres y cómo lo
demuestras_, que es configuración de tu entorno; a partir de ahí es _qué te
dejan hacer_, que vive en la cuenta. Los dos errores canónicos, `Unable to
locate credentials` y `AccessDenied`, caen uno a cada lado de esa flecha.

## 10. Diagnóstico

Casi todo lo que falla en este terreno se resuelve con tres preguntas, en este
orden.

**¿Con qué identidad estoy hablando?** AWS Security Token Service —`sts`, el
servicio emisor de credenciales temporales que la nota anterior nombró al leer
la _trust policy_ de un rol— ofrece `GetCallerIdentity`, que devuelve quién eres
según AWS y que, de forma deliberada, **no requiere ningún permiso**: sirve
incluso cuando todo lo demás responde `AccessDenied`.

```bash
aws sts get-caller-identity
```

```json
{
  "UserId": "AROAEXAMPLEID:mi-sesion",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/DataScientist/mi-sesion"
}
```

```python
print(boto3.client("sts").get_caller_identity()["Arn"])
```

Ese ARN dice la cuenta en la que estás y con qué identidad actúas. Si dice
`user/ana` estás usando un usuario IAM; si dice `assumed-role/...` estás dentro
de un rol. Cuando algo da `AccessDenied`, esta es la primera línea que hay que
ejecutar, porque lo habitual es descubrir que las policies estaban bien y la
identidad era otra.

**¿De dónde salieron esas credenciales?** `aws configure list` con su columna
`Location`, y `sesion.get_credentials().method` en boto3. Si hace falta más
detalle, `--debug` en la CLI imprime el recorrido completo de la cadena,
incluido qué intentó y qué descartó.

**¿Contra qué región?** `aws configure list` la muestra; en boto3,
`sesion.region_name`.

Con eso, el catálogo de errores se lee solo.

- `Unable to locate credentials` — la cadena llegó al final sin encontrar nada.
  No es un problema de permisos. Comprueba el perfil activo, y si es de Identity
  Center, que la sesión no haya caducado.
- `ProfileNotFound` — casi siempre la asimetría de la sección 3: falta la palabra
  `profile` delante del nombre de la sección en `~/.aws/config`.
- `NoRegionError` — región ausente; recuerda la divergencia entre `AWS_REGION` y
  `AWS_DEFAULT_REGION`.
- `ExpiredToken` / `The security token included in the request is expired` — una
  credencial temporal caducada: `aws sso login` otra vez, o `aws login`.
- `InvalidClientTokenId` y `SignatureDoesNotMatch` — problemas de la credencial
  misma: claves borradas o desactivadas, un secreto mal copiado (un espacio de
  más al pegar), un session token olvidado, o el reloj del sistema desviado.
- `AccessDenied` — la otra nota: la identidad es correcta y las policies dicen
  que no.

Y el error que no da error, el peor de todos: **las variables de entorno que
alguien exportó y olvidó**, sobre todo si están en el `.bashrc`. El síntoma es
que `aws configure list` muestra `Type: env` donde esperabas
`shared-credentials-file`; la cura es
`unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN`.

Un último comando, útil y peligroso a partes iguales:

```bash
aws configure export-credentials --profile datos-dev --format env
```

Resuelve las credenciales del perfil por la cadena completa y las imprime como
sentencias de _shell_. Sirve para dar credenciales a una herramienta de terceros
que no sabe leer perfiles, o para pasarlas a un contenedor local. Y sirve, con
la misma facilidad, para volcar un secreto a la pantalla, al historial de la
terminal o a un registro.

## 11. Qué pide de todo esto el MLA-C01

El examen no pregunta por la sintaxis del archivo INI. Pregunta por decisiones,
y son cuatro.

**Credenciales de larga duración en cómputo es siempre la opción incorrecta.**
Si hay un _training job_, una instancia, un contenedor o una función de por
medio, la respuesta es un rol. Pistas estables del enunciado: «sin almacenar
credenciales de larga duración», «que rote automáticamente», «el equipo de
seguridad exige».

**Para personas, identidades federadas mediante IAM Identity Center**, no un
usuario IAM por científico de datos en cada cuenta. Pistas: «varios equipos»,
«varias cuentas», «el directorio corporativo».

**Mínimo privilegio**, que en la nota anterior era un criterio para escribir
policies y aquí se vuelve un criterio para elegir identidad: un cuaderno de
exploración y un _endpoint_ de producción no deben correr con el mismo rol,
aunque sea más cómodo.

**Los secretos que no son de AWS** —la contraseña de una base de datos, la clave
de una API de un tercero— no se resuelven con nada de esta nota. No van a
variables de entorno del contenedor ni al código: van a un servicio de gestión
de secretos y el rol recibe permiso para leerlos. Es el tema de [[Secrets
Manager, Parameter Store y la gestión de secretos en pipelines de ML]], y lo
relevante aquí es no confundir los dos problemas.

Y una distinción de vocabulario que el examen aprovecha: configurar credenciales
es decidir **con qué identidad** habla tu código, mientras que las policies
deciden **qué puede hacer** esa identidad. Un escenario en el que alguien «no
puede leer el _bucket_» se resuelve en un sitio o en el otro según qué error
concreto se esté viendo, y el enunciado siempre lo dice.

## 12. Lo que queda fuera

- [[Roles de IAM, STS y credenciales temporales]] — qué ocurre exactamente
  cuando un perfil con `role_arn` cambia de identidad, qué tiene que decir la
  _trust policy_ para permitirlo, cuánto duran las credenciales que se reciben y
  cómo se le entrega un rol a un servicio para que trabaje por ti.
- [[Federación de identidades en AWS]] — el sistema externo que presenta su
  propia identidad, y el detalle de IAM Identity Center.
- [[Infrastructure as code en AWS: CloudFormation, CDK y Terraform]] — la cuarta
  interfaz que la sección 2 dejó apuntada.
- [[Secrets Manager, Parameter Store y la gestión de secretos en pipelines de ML]]
  — los secretos que no son credenciales de AWS.
- [[El execution role de SageMaker]] — donde esta nota y la anterior se juntan
  en el único rol que vas a escribir cien veces.
