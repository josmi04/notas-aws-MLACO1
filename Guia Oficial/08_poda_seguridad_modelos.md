# Capítulo 8 — Seguridad de modelos

**Objetivos del examen cubiertos:** Dominio 4 (Monitoreo, mantenimiento y seguridad de soluciones de ML) · Tarea 4.3: Proteger recursos de AWS.

> **Nota de vigencia (septiembre de 2026)**
> - **Amazon SageMaker Role Manager** no admite nuevos clientes desde el 30-07-2026. Los clientes existentes pueden seguir usándolo, pero no recibirá funciones nuevas. Como reemplazo, AWS propone roles IAM creados directamente con políticas administradas de SageMaker AI (p. ej., `AmazonSageMakerFullAccess`), *permission sets* de IAM Identity Center, IAM Access Analyzer para generar políticas de mínimo privilegio y plantillas de infraestructura como código. **SageMaker Model Monitor**, que aparece en la tabla de actividades de Role Manager, tampoco admite nuevos clientes desde esa fecha.
> - El **MLA-C01 en inglés** se aplica por última vez el 28-09-2026, y desde el 29-09-2026 lo reemplaza el **MLA-C02**. Los fundamentos de este capítulo (IAM, VPC, cifrado y trazabilidad) siguen vigentes. **Amazon Kendra**, que se menciona en Role Manager, queda fuera del alcance del C02.

---

## 1. Principios de diseño de seguridad

La seguridad es un pilar del AWS Well-Architected Framework y debe considerarse en todas las fases del ciclo de vida de ML. Este enfoque se llama **seguridad por diseño** (*security by design*) y sirve para cualquier ciclo de desarrollo de software, no solo el de ML. AWS ofrece muchos servicios para construir arquitecturas de ML seguras por diseño. Los principios siguientes ayudan a elegir el servicio adecuado para cada componente, y toda solución en AWS debería poder relacionarse con uno o varios de ellos.

### 1.1 Implementar una base sólida de identidad

Los servicios de identidad de AWS giran en torno a **AWS Identity and Access Management (IAM)**. Este servicio controla de forma segura el acceso a servicios y recursos: permite crear usuarios y grupos y asignarles permisos que permiten o deniegan el acceso. Esta base descansa en dos principios:

- **Mínimo privilegio** (*least privilege*): cada identidad (usuario, aplicación o sistema) recibe solo los permisos mínimos que necesita para su función. Así baja el riesgo de actividad maliciosa o de mal uso accidental y, si las credenciales se ven comprometidas, el daño posible queda acotado.
- **Separación de funciones** (*separation of duties*): las responsabilidades se reparten entre varias identidades o sistemas para que ninguno controle todas las operaciones críticas. Esto crea controles cruzados, reduce el riesgo de amenazas internas y mejora la rendición de cuentas.

Más adelante veremos cómo aplicarlos a cargas de ML.

### 1.2 Aplicar seguridad en todas las capas

Este principio corresponde a la **defensa en profundidad** (*defense in depth*): varias capas de controles protegen identidades, infraestructura y datos. Si una capa falla, las demás siguen protegiendo, lo que retrasa y mitiga los ataques y da más tiempo para detectarlos y responder. En una aplicación web expuesta a internet (Figura 8.1 del libro), las capas van de afuera hacia adentro:

| Capa | Servicio | Función |
|---|---|---|
| Borde | Amazon CloudFront | CDN que distribuye contenido globalmente y protege contra DDoS |
| Borde | AWS WAF | Protege la aplicación de exploits y ataques web comunes |
| Aplicación | Application Load Balancer (ALB) | Reparte el tráfico entrante entre destinos, como instancias EC2 o contenedores |
| Subred | Network ACL (NACL) | Firewall de subred que permite o deniega tráfico según reglas |
| Recurso | Security group | Firewall virtual del tráfico entrante y saliente de cada recurso |

La lista puede seguir con **Amazon GuardDuty** (detección de amenazas), **AWS KMS** (cifrado de datos) y **Amazon Inspector** (escaneo continuo de instancias EC2 e imágenes de contenedores en busca de vulnerabilidades de software y exposición de red no deseada).

### 1.3 Habilitar la trazabilidad

La trazabilidad viene del principio de **no repudio**: nadie puede negar una acción o transacción después de realizarla, porque hay prueba del origen y de la integridad de los datos (definición del NIST: <https://csrc.nist.gov/glossary/term/non_repudiation>). Habilitarla significa registrar y monitorear el entorno AWS para rastrear hasta su origen cada acción de usuarios, aplicaciones y sistemas. Así se sabe qué pasó y quién fue el responsable, lo que ayuda a detectar y responder a incidentes, a rendir cuentas y al análisis forense.

| Servicio | Aporte a la trazabilidad |
|---|---|
| AWS CloudTrail | Registra las llamadas a la API de la cuenta: quién las hizo, cuándo y qué acción ejecutó. Es la base de la auditoría, el cumplimiento y la gestión de riesgos (capítulo 7). Si los logs se envían a Amazon S3, se conservan a largo plazo y pueden usarse en análisis forenses |
| AWS Config | Monitorea y registra los cambios de configuración de los recursos para evaluar el cumplimiento y facilitar auditorías |
| Amazon GuardDuty | Detecta amenazas y comportamientos no autorizados con ML, detección de anomalías e inteligencia de amenazas integrada |
| Amazon CloudWatch | Complementa a CloudTrail con monitoreo y alertas en tiempo real: recoge métricas y logs y define alarmas para detectar patrones y anomalías |
| Amazon EventBridge | Reúne eventos de muchas fuentes en un solo lugar para procesarlos y disparar respuestas automáticas |

En ML, la trazabilidad también incluye monitorear el entrenamiento y la inferencia. Si integras EventBridge en los flujos de ML, los eventos relevantes se capturan, se vigilan y se atienden.

### 1.4 Proteger los datos (en reposo, en uso y en tránsito)

Proteger los datos significa garantizar su **confidencialidad, integridad y disponibilidad**. Esta es la **tríada CIA**, la base de toda estrategia de seguridad de la información. En ML es aún más importante, porque se manejan grandes volúmenes de datos sensibles en todas las etapas del ciclo de vida.

| Estado | Definición | Servicios |
|---|---|---|
| En reposo | Datos guardados en discos u otros medios de almacenamiento | **AWS KMS** crea y gestiona las claves de cifrado y se integra de forma nativa con S3, EBS, RDS, DynamoDB y muchos otros servicios. **AWS Secrets Manager** guarda secretos (credenciales de bases de datos, API keys) cifrados con KMS |
| En uso | Datos que una aplicación está procesando | **AWS Nitro Enclaves** crea entornos de cómputo aislados, protegidos por hardware, para datos muy sensibles. **IAM** controla qué usuarios y aplicaciones pueden acceder a los datos |
| En tránsito | Datos que viajan por la red | **AWS Certificate Manager (ACM)** aprovisiona, gestiona y despliega certificados SSL/TLS. Se usa con CloudFront, Elastic Load Balancing y API Gateway para evitar accesos no autorizados y escuchas |

> **Almacenamiento integrado con SageMaker AI (capítulo 2).** Amazon S3, Amazon EFS y Amazon FSx for Lustre se integran de forma nativa con SageMaker AI, pero no todos cifran en reposo por defecto de la misma forma. **S3** cifra cada objeto nuevo automáticamente (SSE-S3), y **FSx for Lustre** siempre cifra en reposo, sin configuración adicional. En **EFS**, en cambio, el cifrado en reposo se elige al crear el sistema de archivos: la consola lo activa por defecto, pero con CLI, API o SDK hay que habilitarlo explícitamente, y después no se puede cambiar.

### 1.5 Automatizar los procesos de seguridad

Con **Policy-as-Code**, las políticas y configuraciones de seguridad se escriben como código: se versionan, se prueban y se despliegan igual que el código de la aplicación. Así se aplican de forma automática y consistente en todo el entorno, disminuye el error humano y se puede responder rápido a las amenazas. AWS Config y AWS IAM permiten automatizar la gestión y la aplicación de las políticas para que infraestructura e identidades cumplan los requisitos de la organización y las normas.

Si se integra en los pipelines de MLOps (capítulo 6), Policy-as-Code aplica los controles en cada etapa del ciclo de vida. Por ejemplo, CodePipeline y CodeBuild pueden automatizar el despliegue de modelos y aplicar a la vez las políticas definidas en código, de modo que el modelo llega a producción con controles de acceso, cifrado y monitoreo.

La automatización también permite vigilar de forma continua las amenazas y el cumplimiento. **Amazon GuardDuty** y **AWS Security Hub**, integrados en los pipelines, generan alertas y hallazgos. Junto con **AWS Lambda**, **CloudWatch** y **EventBridge**, permiten remediar problemas automáticamente según políticas predefinidas. Así la seguridad queda integrada en los flujos de ML, como pide la seguridad por diseño.

### 1.6 Prepararse para eventos de seguridad

Ante un incidente se necesita un plan de respuesta sólido. **Amazon Detective** facilita las investigaciones: recopila automáticamente registros de los recursos (por ejemplo, de CloudTrail) y los hallazgos de GuardDuty. Luego usa ML, análisis estadístico y teoría de grafos para relacionarlos en un solo conjunto de datos donde se investigan las amenazas con rapidez. Detective puede integrarse con las políticas de gestión de incidentes y con las simulaciones de respuesta.

---

## 2. Asegurar los servicios de AWS: un enfoque por capas

Para proteger datos, aplicaciones e infraestructura en la nube conviene seguir un orden sistemático:

1. **Identidades.** Definen quién accede a qué recursos y qué puede hacer con ellos. Con IAM se crean usuarios, grupos y roles con permisos granulares. **MFA** y el mínimo privilegio reducen el riesgo de accesos no autorizados.
2. **Infraestructura.** **Amazon VPC** aísla la red, y las **NACL** y los **security groups** filtran el tráfico entrante y saliente para que solo llegue a los recursos el tráfico autorizado. El cifrado en reposo completa la protección.
3. **Datos.** Se protegen en reposo, en tránsito y en uso con cifrado de KMS, controles de acceso y monitoreo continuo. En ML también hay que cuidar los datos de entrenamiento y de inferencia. SageMaker AI incluye cifrado, controles de acceso detallados y monitoreo para detectar y mitigar amenazas.
4. **Flujos de ML.** La seguridad abarca todo el pipeline: ingesta, preprocesamiento, entrenamiento y despliegue, con controles en cada etapa.
5. **Cumplimiento.** Consiste en cumplir los requisitos de la organización y las normas aplicables:
   - **AWS Config** monitorea y registra de forma continua la configuración de los recursos para evaluar y auditar el cumplimiento.
   - **AWS Security Hub** da una vista centralizada de la postura de seguridad: reúne y prioriza los hallazgos de servicios de AWS y de productos de terceros.
   - **AWS Artifact** permite descargar cuando se necesiten los reportes de auditores, certificaciones, acreditaciones y otras atestaciones de terceros sobre AWS, como los de ISO, PCI y SOC.

---

## 3. Asegurar identidades con IAM

IAM controla quién está **autenticado** (ha iniciado sesión) y quién está **autorizado** (tiene permisos) para usar los recursos de tus cuentas AWS. Un **principal** es una persona o aplicación autenticada con un usuario IAM o un rol IAM. El acceso (la autorización) se gestiona creando **políticas** y adjuntándolas a identidades IAM (usuarios, grupos o roles) o a recursos de AWS.

Aunque el examen se centra en ML, como ingeniero de ML debes saber proteger todos los componentes de tus flujos. Empezamos por las identidades, que son la puerta de entrada.

### 3.1 Identidades

Una **identidad IAM** representa a un usuario humano o a una carga de trabajo programática que puede autenticarse y recibir autorización para actuar en cuentas AWS. Cada identidad puede tener una o más políticas que definen qué acciones puede realizar, sobre qué recursos y en qué condiciones. Las identidades IAM son **usuarios**, **grupos** y **roles**.

- **AWS IAM Identity Center** gestiona de forma centralizada las identidades de la fuerza laboral y su acceso a los recursos. Estas identidades son sus propios usuarios y grupos, distintos de los usuarios IAM. El acceso se asigna con *permission sets*, que crean automáticamente los roles IAM necesarios.
- **Federación**: las identidades no tienen que crearse en AWS. Pueden venir de proveedores externos (Microsoft Entra ID, Okta, Cisco Duo…) y asumir roles IAM para acceder a los recursos.

> No confundas **cuenta AWS** con **identidad**. La cuenta es el contenedor de todos tus recursos en la nube. La identidad (usuario, grupo o rol IAM) es una entidad dentro de la cuenta a la que se le asignan permisos sobre esos recursos. IAM es el servicio que gestiona identidades y accesos.

#### Usuarios IAM

Un usuario IAM es una entidad que creas en tu cuenta y que representa a la persona o carga de trabajo que lo usa para interactuar con AWS. Tiene un nombre y credenciales.

> **Buena práctica:** los usuarios humanos deben acceder a AWS mediante **federación** con un proveedor de identidad y **credenciales temporales**, no como usuarios IAM con credenciales de largo plazo.

Al crear un usuario IAM, IAM genera:

- **Nombre descriptivo** (*friendly name*): el nombre que le diste al crearlo. Aparece en la esquina superior derecha de la consola cuando inicias sesión con ese usuario.
- **ARN**: identifica al usuario de forma única en todo AWS. Sirve, por ejemplo, para indicarlo como `Principal` en una política que le permita escribir en un bucket S3. Ejemplo: `arn:aws:iam::123456789012:user/Dario`.
- **Identificador único**: solo se devuelve si creas el usuario con la API, Tools for Windows PowerShell o AWS CLI. No aparece en la consola.

Cada usuario IAM pertenece a una única cuenta AWS. No necesita un método de pago propio, porque toda su actividad se factura a la cuenta.

#### Roles IAM

Un rol IAM también tiene políticas de permisos que definen qué acciones puede realizar y sobre qué recursos. A diferencia de un usuario, no está ligado a una sola identidad: puede asumirlo cualquier principal que lo necesite, siempre que su **política de confianza** (*trust policy*) lo permita.

> Piensa en un rol como un personaje que un usuario IAM puede adoptar cuando lo necesita.

Un rol no tiene credenciales de largo plazo, como contraseñas o access keys. Cuando lo asumes, **AWS Security Token Service (STS)** te da **credenciales temporales** que duran lo que dura la sesión. Esto es una ventaja: los permisos del rol tienen fecha de caducidad, así que la carga de ML queda menos expuesta a ataques.

El ARN de un rol usa el servicio `iam`, igual que el de cualquier identidad IAM: `arn:aws:iam::123456789012:role/Accounting-Role`. Cuando alguien asume el rol, la **sesión** resultante tiene un ARN de STS formado por el rol asumido y el nombre de la sesión: `arn:aws:sts::123456789012:assumed-role/Accounting-Role/Mario`. La cadena `sts` indica que las credenciales temporales de la sesión las emite STS.

Pueden asumir un rol:

- Un usuario IAM de la misma cuenta o de otra.
- Otro rol IAM, de la misma cuenta o de otra (*role chaining*).
- Un **service principal**, es decir, un principal que representa a un servicio de AWS.
- Un usuario externo autenticado por un proveedor de identidad (IdP).

#### Amazon SageMaker Role Manager

> No admite nuevos clientes desde el 30-07-2026 (ver la nota de vigencia).

Role Manager sugiere permisos para varios *personas* de ML: **Data Scientist**, **MLOps** y **SageMaker AI Compute**. Cada persona trae preseleccionadas unas **actividades de ML**, cada una con sus permisos predefinidos (Tabla 8.1). Puedes añadir o quitar actividades.

| Actividad de ML | Permisos |
|---|---|
| Access Required AWS Services | Acceso a S3, ECR, CloudWatch y EC2. Necesaria para los roles de ejecución de jobs y endpoints |
| Run Studio Classic Applications | Trabajar en un entorno Studio Classic. Necesaria para los roles de ejecución del dominio y del perfil de usuario |
| Manage ML Jobs · Manage Models | Gestionar jobs y modelos de SageMaker AI a lo largo de su ciclo de vida\* |
| Manage Pipelines | Gestionar SageMaker Pipelines y sus ejecuciones |
| Search and visualize experiments | Auditar, consultar el linaje y visualizar experimentos de SageMaker AI |
| Manage Model Monitoring | Gestionar los programas de monitoreo de SageMaker AI Model Monitor |
| Amazon S3 Full Access | Realizar todas las operaciones de S3 |
| Amazon S3 Bucket Access | Operar sobre buckets S3 concretos |
| Query Athena Workgroups | Ejecutar y gestionar consultas de Athena |
| Manage AWS Glue Tables | Crear y gestionar tablas de Glue para SageMaker AI Feature Store y Data Wrangler |
| SageMaker Canvas Core Access | Experimentar en Canvas: preparación básica de datos, construcción y validación de modelos |
| SageMaker Canvas Data Preparation (powered by Data Wrangler) | Preparar datos de principio a fin en Canvas: agregar, transformar y analizar datos, y crear y programar jobs de preparación sobre grandes datasets |
| SageMaker Canvas AI Services | Usar modelos listos de Bedrock, Textract, Rekognition y Comprehend, y hacer fine-tuning de modelos fundacionales de Bedrock y JumpStart |
| SageMaker Canvas MLOps | Desplegar modelos desde Canvas directamente a un endpoint |
| SageMaker Canvas Kendra Access | Dar a Canvas acceso a Amazon Kendra (búsqueda de documentos empresariales), limitado a los índices seleccionados |
| Use MLflow | Gestionar experimentos, ejecuciones y modelos en MLflow |
| Manage MLflow Tracking Servers | Gestionar, iniciar y detener MLflow Tracking Servers |
| Access required to AWS Services for MLflow | Dar a los Tracking Servers acceso a S3, Secrets Manager y Model Registry |
| Run Studio EMR Serverless Applications | Crear y gestionar aplicaciones EMR Serverless desde SageMaker Studio |

\* El libro, que copia la documentación de AWS, describe *Manage ML Jobs* como «auditar, consultar el linaje y visualizar experimentos» (igual que *Search and visualize experiments*) y *Manage Models* como «gestionar jobs». Como esas descripciones no encajan con los nombres, aquí se agrupan y se describen según su nombre.

| Persona | Uso | Actividades preseleccionadas |
|---|---|---|
| Data Scientist | Desarrollo y experimentación general de ML en SageMaker AI | Run Studio Classic Applications, Manage ML Jobs, Manage Models, Manage AWS Glue Tables, SageMaker Canvas AI Services, SageMaker Canvas MLOps, SageMaker Canvas Kendra Access, Use MLflow, Access required to AWS Services for MLflow, Run Studio EMR Serverless Applications |
| MLOps | Tareas operativas | Run Studio Classic Applications, Manage Models, Manage Pipelines, Search and visualize experiments, Amazon S3 Full Access |
| SageMaker AI Compute | Roles de ejecución de jobs y endpoints | Access Required AWS Services |

#### Grupos IAM

Un grupo IAM sirve para gestionar los permisos de varios usuarios a la vez: todos sus miembros heredan las políticas del grupo. Por ejemplo, un grupo `ML-Trainers` puede tener una política que conceda `sagemaker:CreateTrainingJob`. Si llega un ingeniero de ML que necesita crear training jobs, basta con añadirlo al grupo. Si alguien cambia de puesto, basta con moverlo a otro grupo, sin editar sus permisos uno por uno.

Un grupo **no puede ser principal** en una política, porque tiene que ver con los permisos y no con la autenticación. Los principals son entidades autenticadas: usuarios o roles IAM.

¿Cómo sabe AWS si una identidad puede realizar una acción (por ejemplo, escribir) sobre un recurso (por ejemplo, un bucket S3)? Para eso están las políticas de acceso.

### 3.2 Políticas de acceso

Una política es un objeto que define los permisos de la identidad o del recurso al que está asociada. Es la pieza clave de la autorización en AWS, y la mayoría se guardan como documentos JSON. Sus elementos principales son:

| Elemento | Significado |
|---|---|
| `Version` | Versión del lenguaje de políticas (`2012-10-17`) |
| `Statement` | Una o más declaraciones de permisos |
| `Effect` | `Allow` o `Deny` |
| `Action` | Operaciones de API permitidas o denegadas |
| `Resource` | Recursos a los que se aplican las acciones |
| `Principal` | A quién se concede el permiso (en políticas basadas en recursos) |
| `Condition` | (Opcional) Condiciones en las que se aplica la política |

Cuando un principal autenticado hace una solicitud, AWS evalúa las políticas aplicables y decide si la permite o la deniega:

- Lo que ninguna política permite explícitamente queda **denegado por defecto** (denegación implícita).
- Un **`Deny` explícito** siempre prevalece sobre cualquier `Allow`.

A continuación, los tipos que hay que conocer para el examen. Más información: <https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html#access_policy-types>.

#### Políticas basadas en identidad

Son documentos JSON que se adjuntan a una identidad (usuario, grupo o rol) y controlan qué acciones puede realizar, sobre qué recursos y en qué condiciones. Esta política permite todas las acciones de DynamoDB (`dynamodb:*`) sobre la tabla `Vehicles` de la cuenta `123456789012` en `us-east-1`:

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "dynamodb:*",
    "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Vehicles"
  }
}
```

Un usuario suele tener varias políticas que, sumadas, definen sus permisos. Si esta fuera la única, el usuario podría realizar cualquier acción sobre la tabla `Vehicles`, pero no sobre otras tablas ni en EC2, S3 u otros servicios, porque todo lo no permitido se deniega por defecto.

#### Políticas basadas en recursos

Son documentos JSON que se adjuntan a un recurso en lugar de a una identidad. Conceden al principal indicado permiso para realizar ciertas acciones sobre ese recurso y definen en qué condiciones. Esta política de bucket permite al usuario `Dario` escribir en el bucket `your-unique-bucket-name`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/Dario"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::your-unique-bucket-name/*"
    }
  ]
}
```

- `Effect: Allow`: la acción está permitida.
- `Principal`: el usuario IAM `Dario`, identificado por su ARN.
- `Action: s3:PutObject`: permite escribir objetos en el bucket.
- `Resource`: el ARN de los objetos del bucket. El comodín `/*` cubre cualquier objeto. Sustituye `your-unique-bucket-name` por el nombre real del bucket.

No todos los recursos de AWS admiten políticas basadas en recursos. Lista de servicios: <https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html#all_svcs>.

#### Límites de permisos (permissions boundaries)

Un límite de permisos es una función avanzada de IAM que fija los **permisos máximos** que las políticas basadas en identidad pueden conceder a un **usuario o rol IAM** (no se aplica a grupos). Con un límite definido, la identidad solo puede realizar las acciones que permitan **a la vez** sus políticas de identidad y el límite. Este límite restringe a `Dario` a DynamoDB, S3 y CloudWatch:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:*",
        "s3:*",
        "cloudwatch:*"
      ],
      "Resource": "*"
    }
  ]
}
```

El límite **no concede permisos por sí solo**, solo los acota. Aquí, lo máximo que puede hacer Dario es cualquier operación en DynamoDB, S3 y CloudWatch. Nunca podrá operar en otro servicio, ni siquiera en IAM, aunque tenga una política de permisos que se lo permita.

#### Service Control Policies (SCP)

Si tu cuenta pertenece a una organización grande con varias cuentas para distintas funciones del negocio, las SCP permiten gestionar los permisos de forma consistente en toda la organización.

> **AWS Organizations** gestiona y gobierna de forma centralizada varias cuentas AWS: unifica la facturación, aplica SCP para controlar los permisos de las cuentas y automatiza su aprovisionamiento. Conviene usarlo cuando necesitas seguridad y cumplimiento uniformes en muchas cuentas. **AWS Control Tower** amplía Organizations con un entorno multicuenta ya configurado, seguro y conforme a las normas: creación automática de cuentas (Account Factory), seguridad centralizada y buenas prácticas de gobierno. Más información: <https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html>.

Las SCP son políticas JSON que fijan los **permisos máximos** de usuarios y roles en las cuentas de la organización o de sus unidades organizativas (OU). Funcionan como **barreras de protección** (*guardrails*): no conceden permisos, sino que limitan lo que se puede hacer, y así reducen el riesgo de acciones accidentales o maliciosas. Por ejemplo, si las cuentas de una OU solo deben crear instancias EC2 en una región, se crea una SCP que deniega explícitamente `ec2:RunInstances` en cualquier otra región y se aplica a esa OU.

Las SCP se aplican a todos los principals de las cuentas miembro, incluido el usuario raíz. Un `Deny` explícito en una SCP prevalece aunque otras políticas concedan el permiso. Son especialmente útiles para imponer cumplimiento y gobierno centralizado en organizaciones grandes y complejas. Más información: <https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html>.

---

## 4. Asegurar la infraestructura y los datos

### 4.1 Aislamiento de red con VPC

Una **VPC** es una sección aislada lógicamente de la nube de AWS en la que defines tu propia red virtual: rangos de IP, subredes, tablas de rutas y gateways. Es la pieza central del aislamiento de red. Dentro de ella puedes lanzar notebooks, training jobs y endpoints de inferencia de SageMaker, separados del resto de la red de AWS y de las redes externas.

- **NACL**: es un firewall **sin estado** (*stateless*) que actúa sobre la **subred**. No guarda información del tráfico anterior, así que hay que definir reglas tanto de entrada como de salida. Sus reglas **permiten o deniegan** tráfico según IP, protocolo y puerto, y suman una capa más de defensa en profundidad. Una NACL puede asociarse a varias subredes, pero cada subred solo puede tener una NACL a la vez.
- **Security group**: es un firewall **con estado** (*stateful*) que actúa sobre cada **instancia o recurso**. Recuerda el tráfico anterior: si permite tráfico entrante por el puerto 443 hacia una instancia EC2, las respuestas salen automáticamente, sean cuales sean las reglas de salida. Se adjunta a instancias EC2 y RDS, a load balancers, a notebooks de SageMaker, etc. Sus reglas (según IP, protocolo y puerto) solo permiten tráfico. Como origen pueden usar **otros security groups** o **prefix lists**, lo que permite dar acceso a grupos de recursos sin conocer sus IP, por ejemplo a un ELB con IP dinámicas. Una instancia puede tener varios security groups, y sus reglas se combinan para un control más fino.

| | NACL | Security group |
|---|---|---|
| Nivel | Subred | Instancia o recurso |
| Estado | Sin estado: hay que definir reglas de entrada y de salida | Con estado: el tráfico de respuesta se permite automáticamente |
| Reglas | Allow y Deny | Solo Allow |
| Asociación | Varias subredes por NACL; una NACL por subred | Varios security groups por instancia |

### 4.2 Conectividad privada

Los **VPC endpoints** conectan la VPC con servicios de AWS de forma privada, sin pasar por internet. El tráfico se queda dentro de la red de AWS, así que disminuye el riesgo de exponer los datos.

| Tipo | Cómo funciona | Servicios |
|---|---|---|
| Interface endpoint | Usa **AWS PrivateLink**: crea una interfaz de red elástica (ENI) dentro de la VPC | Muchos servicios, p. ej., SageMaker AI, S3 y EC2 |
| Gateway endpoint | Añade entradas a las tablas de rutas de la VPC | Solo Amazon S3 y Amazon DynamoDB |

PrivateLink no se limita a los servicios de AWS: también permite conectar de forma privada servicios propios entre distintas VPC y cuentas, sin exponer el tráfico a internet.

### 4.3 Protección de datos

Además de aislar la red, hay que cifrar los datos sensibles de ML en reposo, en uso y en tránsito. **AWS KMS** crea y gestiona las claves con las que se cifran los datos en servicios como S3, RDS y EBS. Para los datos en tránsito, AWS ofrece cifrado **SSL/TLS** entre la infraestructura de ML y otros servicios o sistemas externos. **ACM** simplifica obtener, renovar y desplegar los certificados SSL/TLS de sitios y aplicaciones. El cifrado mantiene la confidencialidad y la integridad de los datos frente a accesos no autorizados y manipulaciones.

### 4.4 Monitoreo y auditoría

Como se vio en el capítulo 7, **CloudTrail** registra las llamadas a la API y la actividad de los usuarios, y deja un historial de auditoría útil para el cumplimiento y el análisis de seguridad. **CloudWatch** recoge y monitorea métricas y logs, define alarmas y muestra el rendimiento y el estado de la infraestructura de ML. CloudTrail registra la actividad de la API, pero no el tráfico de red. Para registrar el tráfico IP de la VPC se usan los **VPC Flow Logs**, que pueden publicarse en CloudWatch Logs o en S3. Con estas herramientas puedes detectar incidentes y responder a ellos de forma proactiva.

### 4.5 Cumplimiento normativo

SageMaker AI entra en el alcance de muchos programas de cumplimiento, así que es adecuado para organizaciones con requisitos estrictos:

| Marco | Ámbito |
|---|---|
| HIPAA (EE. UU.) | Datos de salud. SageMaker AI es un servicio *HIPAA eligible*, apto para gestionar y analizar información de pacientes respetando las reglas de privacidad y seguridad de HIPAA |
| PCI DSS | Requisitos para proteger los datos de tarjetas de pago en flujos de ML que los usan |
| ISO 27001 | Estándar internacional de sistemas de gestión de la seguridad de la información: evaluación y gestión de riesgos y mejora continua |
| Otros | SOC 1, SOC 2, SOC 3, FedRAMP, GDPR, entre otros |

Que el servicio esté en el alcance de un programa no hace que tu carga de trabajo cumpla automáticamente. El cumplimiento es una **responsabilidad compartida**: te corresponde configurar y operar la solución según cada norma. Además, GDPR es un reglamento, no una certificación. **AWS Artifact** proporciona las atestaciones y los reportes de terceros que respaldan estos programas.

- Marcos de cumplimiento de SageMaker AI: <https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-compliance.html>
- Buenas prácticas de seguridad para SageMaker AI (conformance pack de AWS Config): <https://docs.aws.amazon.com/config/latest/developerguide/security-best-practices-for-SageMaker.html>

---

## 5. Resumen

Proteger modelos de ML con SageMaker AI exige un enfoque integral de **seguridad por diseño**. Este enfoque se basa en el mínimo privilegio, la separación de funciones y la defensa en profundidad, e integra los controles en la arquitectura desde el principio.

- **Identidades y acceso:** IAM define con políticas los permisos de usuarios, grupos y roles para que solo los usuarios autorizados accedan a los recursos de SageMaker AI. MFA añade una capa más, y los límites de permisos y las SCP imponen las políticas de seguridad de la organización.
- **Infraestructura:** las VPC aíslan la red, y las NACL (a nivel de subred) y los security groups (a nivel de instancia) filtran el tráfico. Los VPC endpoints conectan con los servicios de AWS sin exponer los datos a internet. CloudTrail y CloudWatch, junto con VPC Flow Logs para el tráfico de red, permiten auditar y detectar incidentes a tiempo.
- **Datos y cumplimiento:** KMS gestiona las claves de cifrado en reposo, TLS protege los datos en tránsito y ACM gestiona los certificados. SageMaker AI está en el alcance de HIPAA, PCI DSS, ISO 27001, SOC 1/2/3, FedRAMP y GDPR. Eso ayuda a cumplir las normas del sector y las de residencia de datos, siempre dentro del modelo de responsabilidad compartida.

---

## 6. Puntos clave para el examen

- **Diferencia entre usuarios, roles y grupos IAM.** Un **usuario IAM** representa a una persona o un servicio concreto y tiene credenciales de largo plazo (contraseña o access keys). Un **rol IAM** pueden asumirlo usuarios, otros roles, servicios de AWS o identidades federadas, y usa credenciales temporales de STS. Por eso es ideal para dar permisos temporales a aplicaciones o usuarios que necesitan distintos conjuntos de permisos. Un **grupo IAM** reúne usuarios que comparten permisos y no puede ser principal.
- **Elementos de una política de acceso.** Una política es un documento JSON asociado a una identidad o a un recurso. Contiene `Version`, `Statement`, `Effect`, `Action`, `Resource` y, de forma opcional, `Condition` (más `Principal` en las políticas basadas en recursos). Juntos definen qué acciones se permiten o deniegan, sobre qué recursos y en qué condiciones.
- **Tipos de políticas:**

  | Tipo | Se adjunta a | Qué hace |
  |---|---|---|
  | Basada en identidad | Usuario, grupo o rol | Define qué acciones puede realizar la identidad y sobre qué recursos |
  | Basada en recursos | Recurso (p. ej., bucket S3) | Define quién puede acceder al recurso y con qué acciones |
  | Límite de permisos | Usuario o rol | Fija el máximo que pueden conceder las políticas de identidad; no concede permisos |
  | SCP | Organización, OU o cuenta (AWS Organizations) | Fija el máximo para usuarios y roles de las cuentas miembro; no concede permisos |

- **SageMaker Role Manager.** Simplifica la creación de roles IAM para ML a partir de *personas* (Data Scientist, MLOps, SageMaker AI Compute) con permisos predefinidos por actividad. Así ayuda a aplicar el mínimo privilegio y a dar a cada usuario el acceso que corresponde a su función. No admite nuevos clientes desde el 30-07-2026.
- **NACL frente a security groups.** Las NACL son firewalls sin estado que actúan sobre la subred, permiten o deniegan tráfico según IP, protocolo y puerto, y necesitan reglas de entrada y de salida. Los security groups son firewalls con estado que actúan sobre la instancia, solo permiten tráfico y dejan pasar las respuestas automáticamente.
- **Conectividad privada en una VPC.** Se implementa con VPC endpoints. Los **interface endpoints** usan PrivateLink y una ENI dentro de la VPC (p. ej., para S3, SageMaker AI o EC2). Los **gateway endpoints** añaden rutas a las tablas de rutas y solo existen para S3 y DynamoDB. Con ambos, el tráfico se queda dentro de la red de AWS.

---

## 7. Preguntas de repaso

1. ¿Cuál es la principal diferencia entre roles IAM y usuarios IAM en cuanto a credenciales de acceso?
   - A. Los usuarios IAM tienen credenciales temporales y los roles IAM, credenciales de largo plazo.
   - B. Los roles IAM proporcionan credenciales temporales y los usuarios IAM tienen credenciales de largo plazo.
   - C. Ambos tienen credenciales de largo plazo.
   - D. Ambos tienen credenciales temporales.

2. ¿Qué servicio de AWS permite monitorear el cumplimiento de las políticas internas y los estándares regulatorios en Amazon SageMaker AI?
   - A. Amazon GuardDuty
   - B. AWS Config
   - C. AWS CloudTrail
   - D. Amazon Inspector

3. ¿Qué distinción clave separa los security groups de las NACL?
   - A. Los security groups actúan sobre la instancia y no tienen estado; las NACL actúan sobre la subred y tienen estado.
   - B. Los security groups actúan sobre la subred y no tienen estado; las NACL actúan sobre la instancia y tienen estado.
   - C. Los security groups actúan sobre la instancia y tienen estado; las NACL actúan sobre la subred y no tienen estado.
   - D. Los security groups actúan sobre la subred y tienen estado; las NACL actúan sobre la instancia y no tienen estado.

4. ¿Qué personas vienen predefinidas en Amazon SageMaker Role Manager?
   - A. Data Scientist y MLOps
   - B. System Administrator y Network Engineer
   - C. Financial Analyst y HR Manager
   - D. Marketing Specialist y Sales Representative

5. ¿Cuál es un beneficio clave de usar AWS PrivateLink para la comunicación segura entre VPC y servicios de AWS?
   - A. Habilita el acceso a internet a los servicios de AWS.
   - B. Crea endpoints públicos para los servicios de AWS.
   - C. Proporciona conectividad privada sin exponer los datos a internet.
   - D. Gestiona el tráfico de red dentro de una VPC.

6. ¿Cómo mejoran la seguridad las SCP dentro de una organización de AWS?
   - A. Proporcionando logs detallados de llamadas a la API.
   - B. Imponiendo permisos máximos a las cuentas miembro.
   - C. Cifrando los datos en reposo.
   - D. Monitoreando el tráfico dentro de la VPC.

7. ¿Qué servicio de AWS ayuda a imponer requisitos de residencia de los datos de entrenamiento en una aplicación de ML dentro de un entorno multicuenta?
   - A. AWS CloudTrail con políticas basadas en identidad
   - B. AWS Config con políticas basadas en recursos
   - C. AWS Control Tower con Service Control Policies
   - D. Amazon CloudWatch con listas de control de acceso

8. ¿Qué papel cumple AWS KMS en la protección de los datos de ML en Amazon SageMaker AI?
   - A. Generar logs de auditoría detallados de las llamadas a la API.
   - B. Monitorear el rendimiento de los modelos de ML.
   - C. Crear y gestionar claves criptográficas para cifrar datos.
   - D. Controlar el tráfico entre subredes e instancias.

9. ¿Qué servicio de AWS se usa principalmente para registrar las llamadas a la API y la actividad con fines de auditoría de seguridad y cumplimiento?
   - A. AWS Shield
   - B. AWS CloudTrail
   - C. AWS CloudFormation
   - D. Amazon CloudWatch

10. ¿Qué estrategia de seguridad propone incorporar la seguridad desde las primeras etapas de diseño y desarrollo?
    - A. Defensa en profundidad
    - B. Mínimo privilegio
    - C. Seguridad por diseño
    - D. Autenticación multifactor (MFA)

**Respuestas:** 1-B · 2-B · 3-C · 4-A · 5-C · 6-B · 7-C · 8-C · 9-B · 10-C
