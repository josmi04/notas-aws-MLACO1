# El primer día de Ana

*Simulación de onboarding: de pedir acceso a AWS a lanzar el primer job de SageMaker, usando IAM Identity Center.*

---

## Contexto

Ana es data scientist y hoy es su primer día en la empresa. Ya tiene laptop,
correo corporativo y acceso a Slack. Le falta una cosa para poder empezar a
trabajar: acceso a AWS, donde vive todo el pipeline de datos y los jobs de
entrenamiento en SageMaker.

La empresa usa **IAM Identity Center** para el acceso humano a AWS, así que
Ana no va a recibir ninguna clave de acceso por correo ni va a crear ningún
usuario IAM. Todo pasa por un portal único con su cuenta corporativa.

---

## 1. El mensaje a DevOps

Ana entra a Slack y escribe en el canal `#platform-devops`:

> **Ana** — Hola! Soy nueva en el equipo de Data Science, empiezo hoy. ¿Cómo
> consigo acceso a AWS para trabajar con SageMaker?

DevOps responde con tres datos, nada más:

> **DevOps** — Bienvenida! Usamos IAM Identity Center, no hace falta que
> crees nada. Necesitas:
> - URL del portal: `https://mi-empresa.awsapps.com/start`
> - Región del SSO: `us-east-1`
> - Ya te dimos acceso a la cuenta `123456789012` con el rol `DataScientist`
>
> Corre `aws configure sso` en tu terminal y te autenticas con tu cuenta
> corporativa (la misma de tu correo). Cualquier duda, aquí estamos.

Eso es todo lo que Ana necesita. No hay ninguna clave que copiar ni ningún
`.csv` de credenciales que guardar con cuidado.

---

## 2. Configurando el perfil

Ana abre la terminal:

```bash
aws configure sso
```

Y responde con los datos que le dio DevOps:

```
SSO session name (Recommended): mi-empresa
SSO start URL [None]: https://mi-empresa.awsapps.com/start
SSO region [None]: us-east-1
SSO registration scopes [None]: sso:account:access
```

Se le abre el navegador, entra con su correo corporativo y su contraseña de
siempre, y la CLI le muestra la cuenta y el rol a los que tiene acceso:

```
There are 1 AWS accounts available to you.
> Mi Empresa - Data, 123456789012
Using the role name "DataScientist"
```

Le pide un nombre de perfil y una región por defecto:

```
CLI default client Region [None]: us-east-1
CLI default output format [None]: json
CLI profile name [123456789012_DataScientist]: datos-dev
```

Con eso, `~/.aws/config` queda escrito automáticamente:

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

Ana nunca vio ni escribió una clave de acceso. No existe ningún
`aws_secret_access_key` en su laptop.

---

## 3. Primer login del día

Antes de poder usar el perfil, tiene que autenticarse:

```bash
aws sso login --profile datos-dev
```

Se le abre el navegador otra vez (esta vez más rápido, ya tiene sesión),
confirma, y la terminal responde:

```
Successfully logged into Start URL: https://mi-empresa.awsapps.com/start
```

Eso deja un testigo guardado en `~/.aws/sso/cache/`. A partir de aquí, todo
lo que corra con `--profile datos-dev` obtiene credenciales automáticamente,
sin que ella tenga que hacer nada más.

---

## 4. Comprobando que todo funciona

Un vistazo rápido para confirmar quién es y qué permisos tiene:

```bash
aws sts get-caller-identity --profile datos-dev
```

```json
{
    "UserId": "AROAEXAMPLE:ana.garcia",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/DataScientist/ana.garcia"
}
```

Y para no repetir `--profile datos-dev` en cada comando, lo exporta como
variable de entorno para esta sesión de terminal:

```bash
export AWS_PROFILE=datos-dev
```

Con eso, cualquier comando de la CLI o cualquier script de Python que use
`boto3` va a tomar este perfil sin necesidad de especificarlo cada vez.

---

## 5. Explorando qué hay ya montado

Antes de lanzar nada, Ana revisa qué recursos existen ya en la cuenta, porque
en una empresa con equipo de plataforma casi nunca se empieza desde cero.

Busca el rol de ejecución de SageMaker (normalmente lo crea Infraestructura,
no cada data scientist):

```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'SageMaker')].RoleName"
```

```json
[
    "SageMakerExecutionRole-DataTeam"
]
```

Y revisa qué buckets de S3 tiene disponibles para el equipo de datos:

```bash
aws s3 ls | grep datos
```

```
2024-01-15 09:12:33 mi-empresa-datos-dev
```

Dentro, ya hay una carpeta de entrada con datos de ejemplo:

```bash
aws s3 ls s3://mi-empresa-datos-dev/entrenamientos/entrada/
```

```
2026-09-10 11:02:07     284112 dataset.csv
```

---

## 6. Primer job de SageMaker

Con el rol y el bucket identificados, Ana escribe un script mínimo en Python
usando el SDK de SageMaker:

```python
import boto3
import sagemaker
from sagemaker.estimator import Estimator

# boto3 usa las credenciales del perfil activo (AWS_PROFILE=datos-dev)
boto_session = boto3.Session()
sagemaker_session = sagemaker.Session(boto_session=boto_session)

role = "arn:aws:iam::123456789012:role/SageMakerExecutionRole-DataTeam"

estimator = Estimator(
    image_uri="123456789012.dkr.ecr.us-east-1.amazonaws.com/mi-imagen-entrenamiento:latest",
    role=role,
    instance_count=1,
    instance_type="ml.m5.xlarge",
    output_path="s3://mi-empresa-datos-dev/entrenamientos/salida/",
    sagemaker_session=sagemaker_session,
)

estimator.fit({"train": "s3://mi-empresa-datos-dev/entrenamientos/entrada/"})
```

Lo corre:

```bash
python entrenar.py
```

```
INFO:sagemaker:Creating training-job with name: mi-imagen-entrenamiento-2026-09-15-14-32-07-812
...............
2026-09-15 14:35:41 Completed - Training job completed
```

Ningún paso de todo esto involucró una clave de acceso permanente.

---

## 7. El testigo caduca

Unas horas después, Ana quiere revisar el resultado del job y corre otro
comando:

```bash
aws sagemaker describe-training-job --training-job-name mi-imagen-entrenamiento-2026-09-15-14-32-07-812
```

Pero recibe un error:

```
The SSO session associated with this profile has expired or is otherwise
invalid. To refresh this SSO session run aws sso login with the corresponding
profile.
```

No es un fallo, es el diseño funcionando como debe: el testigo tenía una
vida útil corta, y al vencer deja de servir para nada. Ana simplemente repite:

```bash
aws sso login --profile datos-dev
```

Y sigue trabajando. Si en algún momento quiere cerrar sesión explícitamente
—por ejemplo, al terminar el día en una laptop compartida—, puede correr:

```bash
aws sso logout
```

Eso borra el testigo de `~/.aws/sso/cache/` y cualquier comando futuro le
va a pedir autenticarse de nuevo.

---

## Resumen del día

| Paso | Comando | Qué logra |
|---|---|---|
| 1 | `aws configure sso` | Crea el perfil en `~/.aws/config`, sin secretos |
| 2 | `aws sso login --profile datos-dev` | Autentica y guarda un testigo temporal |
| 3 | `export AWS_PROFILE=datos-dev` | Evita repetir `--profile` en cada comando |
| 4 | `aws sts get-caller-identity` | Confirma identidad y permisos |
| 5 | `python entrenar.py` | Lanza el primer job de SageMaker |
| 6 | `aws sso login` (al expirar) | Renueva el acceso cuando el testigo caduca |

En ningún momento Ana tuvo, copió o guardó una clave de acceso permanente. Si
alguna vez pierde la laptop, no hay nada en `~/.aws/` que alguien pueda usar
directamente: el testigo caduca solo, y sin él, el perfil no sirve para nada.
