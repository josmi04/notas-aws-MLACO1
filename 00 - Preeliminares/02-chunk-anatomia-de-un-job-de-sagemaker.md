---
tema: "Anatomía de un job de SageMaker: Processing y Training — repaso"
nota: 2 de 29 (versión chunk)
certificacion: MLA-C01
tipo: cheat-sheet de repaso
fuente: "[[02-anatomia-de-un-job-de-sagemaker]]"
tareas_guia: ["2.2 S", "1.2 K/S", "3.2 S"]
versiones: { aws_cli: "2.36.49", boto3: "1.43.98", sagemaker_sdk: "3.22.1" }
verificado: 2026-09-20
tags: [aws, sagemaker, mla-c01, processing, training, repaso]
---

# Anatomía de un job de SageMaker — repaso

---

## 1. Qué es un job

Una **llamada a la API** que dice: *enciende N máquinas de este tipo, corre esta imagen dentro, pon estos objetos de S3 a su alcance, guarda la salida en este otro sitio de S3 y apágalo todo al terminar*. Sustituye los 8 pasos manuales de EC2 (lanzar, esperar, conectar, instalar, copiar datos, ejecutar, copiar resultado, apagar).

Los cinco campos que componen cualquier petición: **rol · imagen · instancias · entrada S3 · salida S3** (+ límite de tiempo).

---

## 2. Plano de control vs. plano de ejecución

- **Plano de control**: el endpoint `sagemaker.<región>.amazonaws.com`. Valida la petición, devuelve un ARN, se queda con el encargo.
- **Plano de ejecución**: las instancias que arranca. **No aparecen en tu consola de EC2**, no tienen IP que puedas anotar, dejan de existir al terminar.
- Su única conexión con tus datos es el **rol de ejecución**, que SageMaker asume en tu nombre.

> **Asimetría clave (muy preguntada):** `create-*` es **asíncrono**. Devuelve el ARN en <1 s, antes de que exista ninguna máquina. El éxito de la llamada **no** es el éxito del job. El estado se consulta con `Describe*`.

---

## 3. Las dos familias y su diferencia real

| Familia | Llamada | Para qué | Qué deja |
|---|---|---|---|
| Processing | `CreateProcessingJob` | correr un programa sobre datos: limpiar, validar, convertir, evaluar | los archivos que tu programa escriba, copiados a S3 |
| Training | `CreateTrainingJob` | entrenar un modelo | `model.tar.gz` en una ruta convenida + métricas y tiempos |

La diferencia **no es de tamaño ni potencia: es de contrato**. Training promete artefacto en ruta fija e informa del tiempo facturable; Processing no promete nada salvo copiar lo que encuentre.

---

## 4. Desde dónde se lanza

La API es idéntica desde laptop (CLI), notebook de Studio, Lambda o CodeBuild. **Studio no es requisito.** Lo único que cambia es la identidad que firma. Un **dominio** de Studio agrupa perfiles de usuario y decide *desde dónde* corre el código y *con qué identidad*, no *qué* puedes lanzar. Comprobar: `aws sagemaker list-domains`.

---

## 5. URI de imagen: anatomía

```
683313688378.dkr.ecr.us-east-1.amazonaws.com/sagemaker-xgboost:1.7-1
└────┬─────┘        └───┬───┘                └──────┬──────┘ └─┬─┘
 cuenta AWS         región                    repositorio    tag
 propietaria     del registro                              (versión)
```

1. **La cuenta no es la tuya.** Las imágenes integradas viven en cuentas de AWS. No las copias ni las pagas: las referencias.
2. **La región va dentro del nombre.** Hay una copia del registro por región, en cuentas distintas. `us-east-1` ≠ `eu-west-1` (ej. `141502667606` en eu-west-1).
3. **El tag es la versión del framework/algoritmo**, no la de SageMaker.

El registro es **Amazon ECR**. La imagen **no viaja dentro de la petición**.

---

## 6. Obtener la URI sin inventarla

```python
from sagemaker.core.image_uris import retrieve
retrieve(framework="xgboost", region="us-east-1", version="1.7-1", image_scope="training")
```

- **No llama a AWS**: compone la cadena con una tabla local del paquete. Si el paquete es viejo, la URI es vieja. Tampoco comprueba que la imagen exista.
- `image_scope`: `training` e `inference` son **imágenes distintas** (repositorios distintos).
- **No existe `image_scope="processing"`** → `ValueError`. Para sklearn, la imagen de *training* es también la de Processing.
- `instance_type` solo importa en frameworks con variante por hardware (sklearn → sufijo `-cpu-py3`; XGBoost lo ignora).

---

## 7. Contrato `/opt/ml` — Processing

Dentro del contenedor **no hay credenciales de AWS ni SDK**: tu código nunca llama a S3. SageMaker copia S3→disco antes de arrancar y disco→S3 al terminar.

| Ruta | Quién la llena | Cuándo |
|---|---|---|
| `/opt/ml/processing/input/<nombre>` | SageMaker | antes de arrancar |
| `/opt/ml/processing/output/<nombre>` | tu programa | durante la ejecución |
| `/opt/ml/config/processingjobconfig.json` | SageMaker (petición completa) | antes de arrancar |
| `/opt/ml/config/resourceconfig.json` | SageMaker (nombres del clúster) | antes de arrancar |

`<nombre>` **lo eliges tú** en la petición (`LocalPath`). La convención `input/<nombre>` es costumbre de AWS, no obligación. Processing **no tiene equivalente de `/opt/ml/model`**: todas sus salidas son iguales.

---

## 8. Contrato `/opt/ml` — Training

| Ruta | Quién la llena | Significado |
|---|---|---|
| `/opt/ml/input/data/<canal>` | SageMaker | datos del canal |
| `/opt/ml/input/config/hyperparameters.json` | SageMaker | hiperparámetros, **como cadenas** |
| `/opt/ml/input/config/inputdataconfig.json` | SageMaker | descripción de canales |
| `/opt/ml/input/config/resourceconfig.json` | SageMaker | nombres del clúster |
| `/opt/ml/model` | tu programa | **se empaqueta y sube como el artefacto del modelo** |
| `/opt/ml/output/data` | tu programa | se empaqueta y sube aparte (salida auxiliar) |
| `/opt/ml/output/failure` | tu programa, solo si falla | sus **primeros 1024 caracteres** → `FailureReason` |

- Un **canal** es un grupo de objetos de S3 con nombre; el nombre decide el subdirectorio.
- `train` / `validation` los exige **el algoritmo** (XGBoost), no SageMaker.
- `/opt/ml/model` sube **comprimido en un solo objeto** (`model.tar.gz`), no archivo por archivo. `/opt/ml/output/data` → otro objeto (`output.tar.gz`).
- `/opt/ml/output/failure` es un **archivo**, no directorio, y solo se lee si el job falla.

---

## 9. El script también entra por una ruta de entrada

**No hay mecanismo especial para el código.** Tu `.py` se sube a S3 como cualquier objeto y se declara como una entrada más; lo que lo convierte en "el programa" es que `ContainerEntrypoint` le dice al contenedor que lo ejecute.

Cómo se relacionan script y petición: **por convenio externo, no por código**. Las rutas son literales fijos en el script. Si no coinciden con `LocalPath` → `FileNotFoundError` o, peor, **job `Completed` sin escribir nada**.

Dos detalles del script que son contrato, no estilo:
- `raise SystemExit(...)` → código de salida ≠ 0 → job en `Failed`. Es la **única** forma que tiene tu programa de avisar: SageMaker juzga por el código de salida del contenedor.
- `print(..., flush=True)` o `PYTHONUNBUFFERED=1` → los logs salen al momento en vez de perderse si el proceso muere.

---

## 10. `CreateProcessingJob` — campos

| Campo | Obligatorio | Notas de examen |
|---|---|---|
| `ProcessingJobName` | sí | único en cuenta+región, 1–63 chars alfanuméricos y guiones. **Los nombres usados quedan ocupados para siempre** → marca de tiempo. Es el prefijo con el que buscas los logs. |
| `RoleArn` | sí | **Las instancias asumen este rol**; todos los accesos a S3 los hace él, no tú. Necesitas `iam:PassRole` sobre él. |
| `AppSpecification.ImageUri` | sí | la URI del chunk 5 |
| `AppSpecification.ContainerEntrypoint` | no | lista de argumentos ya partida (**no** cadena de shell). Sustituye al arranque por defecto de la imagen. `ContainerArguments` añade args sin tocarlo. Máx. 100 elementos cada una. |
| `ProcessingResources.ClusterConfig` | sí | `InstanceType` familia `ml.` (≠ EC2, **no comparte cuota**); `InstanceCount` 1–100; **`VolumeSizeInGB` es obligatorio**, 1–16384 (si los datos no caben, falla al descargar, no al procesar); `VolumeKmsKeyId` cifra el disco |
| `ProcessingInputs[]` | — | **hasta 10** |
| `ProcessingOutputConfig.Outputs[]` | — | **hasta 10**, cada una con `LocalPath` + destino S3. `KmsKeyId` cifra lo escrito en S3 |
| `StoppingCondition.MaxRuntimeInSeconds` | **no** | default = **un día**. Al alcanzarlo: `SIGTERM` + **120 s** de gracia |
| `Environment` | no | hasta 100 variables, todas cadenas. **No es sitio para secretos**: sale tal cual en `Describe*` |

Fuera de alcance aquí pero visibles en el `describe`: `NetworkConfig`, `ExperimentConfig`, `Tags`.

---

## 11. `ProcessingInputs` — subcampos

- `InputName`: libre, identifica la entrada en el `describe`.
- `S3Uri`: prefijo u objeto concreto.
- `S3DataType`: `S3Prefix` | `ManifestFile`. Con `S3Prefix` se descarga **todo** lo que cuelgue → un prefijo ancho es tiempo y disco desperdiciados.
- `LocalPath`: dónde aparecen esos objetos dentro del contenedor.
- `S3InputMode`: `File` (copia todo al disco antes de arrancar) | `Pipe`.
- `S3DataDistributionType`: ver chunk 12.

---

## 12. `S3DataDistributionType` — el campo que rompe jobs paralelos

| Valor | Qué hace |
|---|---|
| `FullyReplicated` | cada instancia recibe una **copia completa** → el trabajo se **triplica** con 3 instancias, no se reparte |
| `ShardedByS3Key` | reparte los objetos entre instancias |

Con `InstanceCount: 1` da igual. **Trampa clásica:** un script que agrega totales corriendo con `ShardedByS3Key` produce resultados parciales **sin error alguno** y el job termina en `Completed`. Y subir instancias con `FullyReplicated` multiplica el coste sin repartir el trabajo.

---

## 13. `S3UploadMode` (solo Processing)

| Valor | Cuándo sube | Cuándo lo quieres |
|---|---|---|
| `EndOfJob` | al terminar el contenedor, y **solo si terminó bien** | resultados que no sirven a medias |
| `Continuous` | mientras corre, según se escriben | salidas que quieres conservar aunque el job falle o lo corten después; jobs largos cuyo progreso se inspecciona |

**Consecuencia:** un job cortado por `MaxRuntimeInSeconds` con `EndOfJob` no sube **nada**; S3 conserva intactos los datos anteriores y el pipeline "no falla", simplemente no produce nada nuevo.

---

## 14. `CreateTrainingJob` — lo nuevo respecto a Processing

| Campo | Notas |
|---|---|
| `AlgorithmSpecification.TrainingInputMode` | **obligatorio, sin default.** Es el campo que más peticiones rechaza la primera vez. `File` / `Pipe` / `FastFile` |
| `InputDataConfig[]` | lista de **canales**. `ChannelName` decide `/opt/ml/input/data/<canal>`. `ContentType` se pasa al contenedor tal cual: **SageMaker no lo valida ni convierte nada** |
| `OutputDataConfig.S3OutputPath` | obligatorio. **No hay `LocalPath`**: las rutas de salida son fijas por contrato |
| `ResourceConfig` | equivalente de `ClusterConfig` |
| `HyperParameters` | mapa **cadena → cadena**. `"num_round": 100` sin comillas es **error de validación**, no conversión automática |

**Ruta final del artefacto:**
`s3://<S3OutputPath>/<TrainingJobName>/output/model.tar.gz`
SageMaker intercala el nombre del job → dos entrenamientos nunca se pisan el artefacto → el `S3OutputPath` se comparte entre jobs sin problema.

---

## 15. Equivalencias Processing ↔ Training (el examen las mezcla)

| Concepto | Processing | Training |
|---|---|---|
| Imagen | `AppSpecification.ImageUri` | `AlgorithmSpecification.TrainingImage` |
| Máquinas | `ProcessingResources.ClusterConfig` | `ResourceConfig` |
| Entradas | `ProcessingInputs[]` (hasta 10) | `InputDataConfig[]` (canales) |
| Salidas | `ProcessingOutputConfig.Outputs[]` (hasta 10) | `OutputDataConfig` (**uno solo**) |
| Límite de tiempo | `StoppingCondition` | `StoppingCondition` |
| Parámetros del programa | `Environment`, `ContainerArguments` | `HyperParameters`, `Environment` |

---

## 16. Estados del job (5 valores)

```
[*]        → InProgress    (create aceptado)
InProgress → Completed     (exit 0 + salidas subidas)
InProgress → Failed        (exit ≠ 0, error al descargar/subir, imagen inaccesible)
InProgress → Stopping      (StopProcessingJob o MaxRuntimeInSeconds)
Stopping   → Stopped       (tras SIGTERM + 120 s de gracia)
```

Dos lecturas que el examen aprovecha:

- **`Stopped` no es `Failed`.** Un job cortado por tiempo y uno parado a mano dan el mismo estado. Si tu automatización trata *"no Failed"* como éxito, ese job **pasa desapercibido** → comprobar `== "Completed"` explícitamente.
- **`Failed` no dice dónde falló.** Permisos al descargar, `KeyError` en pandas y disco lleno al subir dan el mismo estado. Lo que los distingue es `FailureReason`.

---

## 17. `Describe*` — campos de diagnóstico

Comunes:
- **`FailureReason`**: lo primero que se mira si el estado es `Failed`. En Training viene de los primeros 1024 caracteres de `/opt/ml/output/failure`, o lo escribe SageMaker si el fallo fue fuera de tu código (descarga, imagen, cuota).
- **`ExitMessage`**: **solo Processing**. Trae el final de la salida de error del contenedor (ahí ves tu excepción de Python).
- **`ProcessingStartTime` no es el instante del `create`**: es cuando el contenedor empezó a correr. Entre uno y otro hay encendido + descarga de imagen + descarga de datos, **que también se facturan**. En jobs de 3 minutos ese preámbulo suele ser la mitad del gasto → argumento contra partir un pipeline en veinte jobs diminutos.

Solo Training:
- **`SecondaryStatus`** → chunk 18.
- **`ModelArtifacts.S3ModelArtifacts`**: la ruta exacta del artefacto. **Léela de aquí en vez de construirla a mano.**
- **`TrainingTimeInSeconds` vs `BillableTimeInSeconds`**: tiempo que las instancias estuvieron encendidas vs. lo que te cobran. Coinciden salvo con spot / warm pools.

---

## 18. `SecondaryStatus` — orden de fases (Training)

```
Starting → Downloading → Training → Uploading → Completed
```

| Fase | Qué ocurre |
|---|---|
| `Starting` | pidiendo y arrancando máquinas |
| `Downloading` | bajando imagen y datos |
| `Training` | tu contenedor corriendo |
| `Uploading` | subiendo el artefacto |

El histórico con la hora de cada transición: `SecondaryStatusTransitions`.

> **Un job atascado muchos minutos en `Starting` no tiene un problema de código**: es capacidad o cuota del tipo de instancia.

---

## 19. Waiters

`processing_job_completed_or_stopped` · `training_job_completed_or_stopped`

- **Defaults distintos:** Processing cada **60 s ×60** (1 h); Training cada **120 s ×180** (6 h). Un Training largo agota el waiter aunque el job siga vivo → `WaiterConfig={"Delay": …, "MaxAttempts": …}`.
- **`Failed` lanza `WaiterError`.** La excepción dice que dejó de esperar, **no por qué falló el job** → dentro del `except` hay que volver a llamar al `describe`.
- **`Stopped` se considera éxito de la espera** (*completed or **stopped***) → un job cortado por tiempo sale por el camino del `try`. Si eso importa, lee el estado después.

Parar a mano: `aws sagemaker stop-processing-job`. Vuelve enseguida y el job pasa a `Stopping`; el contenedor recibe la señal y dispone de la ventana de gracia para cerrar. Si el código la ignora, se pierden las salidas en modo `EndOfJob`.

---

## 20. CloudWatch Logs

Todo lo que el contenedor escriba en stdout/stderr va a CloudWatch. **No hay que configurarlo.** Es el único sitio donde ves tus `print`.

| Tipo | Grupo de logs | *Stream* |
|---|---|---|
| Processing | `/aws/sagemaker/ProcessingJobs` | `[nombre-job]/[hostname]-[epoch]` |
| Training | `/aws/sagemaker/TrainingJobs` | `[nombre-job]/algo-[n]-[epoch]` |

- **Grupo** = contenedor administrativo (retención, permisos), **uno por tipo de job**, compartido por toda la cuenta y región.
- ***Stream*** = mensajes de **una** máquina de **un** job. `algo-1`, `algo-2`, `algo-3` con `InstanceCount: 3`.
- El sufijo epoch **no se puede predecir** → siempre se consulta **por prefijo** (`nombre-del-job/`). De ahí que convenga un nombre de job legible.

```
aws logs tail /aws/sagemaker/ProcessingJobs \
  --log-stream-name-prefix <nombre-job> --since 1h --follow --format short
```

- El grupo va **posicional**, sin bandera. `--since` omitido = solo **últimos 10 minutos** (hace parecer vacío un job de ayer). `--log-stream-name-prefix` es incompatible con `--log-stream-names`.

En Python: **`filter_log_events`** (busca en varios streams de un grupo a la vez) frente a `get_log_events` (exige el nombre exacto de un stream). Usa `pagina.get("events", [])`: **las páginas sin resultados no traen la clave**.

> **Señal diagnóstica de oro:** si el job está en `Failed` y **no existe ningún stream**, el contenedor nunca arrancó → el problema es de **datos, permisos o imagen**, nunca de tu código. La respuesta está en `FailureReason`.

---

## 21. SDK vs boto3 vs CLI

El **SageMaker Python SDK** construye las peticiones desde objetos de Python; por debajo llama a boto3 con tu misma sesión. No es un servicio ni una credencial distinta.

Lo que el SDK **decide por ti** (y por eso una petición del SDK no se parece campo por campo a una escrita a mano):
- Genera el **nombre único** desde `base_job_name` → relanzar nunca choca por nombre repetido.
- Rellena obligatorios de la API con defaults suyos (`TrainingInputMode="File"`, `volume_size_in_gb=30`).
- Si le das una **ruta local**, la sube al *bucket por defecto* `sagemaker-<región>-<cuenta>`; si omites `output_data_config`, **inventa** una ruta ahí → riesgo de cumplimiento cuando el destino de los datos está regulado.
- `train(wait=True, logs=True)` = waiter + `tail` juntos. `wait=False` vuelve de inmediato, como `create_training_job`.
- `train(dry_run=True)` valida toda la configuración **sin crear el job**.

| Situación | Herramienta |
|---|---|
| Exploración en notebook, iterar sobre un entrenamiento | **SDK** |
| Lambda que dispara un job al llegar datos | **boto3** (el SDK es dependencia grande; la petición ya está decidida) |
| La petición vive en el repo y se revisa en PR | **CLI `--cli-input-json`** (el JSON es el artefacto revisable) |
| Automatización en otro lenguaje o desde otro servicio | **boto3 / API** |
| Reproducir exactamente lo enviado, para depurar | **boto3** |
| Subir código local y empaquetarlo con el job | **SDK** |

> **Regla corta: el SDK decide cosas por ti; boto3 no decide nada.** En exploración eso es lo que quieres; en producción cada decisión implícita es una diferencia entre entornos.

**CLI y boto3 usan nombres de campo idénticos** (ambos son envoltorios finos sobre la misma API) → un JSON prototipado se pasa a Python con `create_processing_job(**peticion)` sin traducir nada.

---

## 22. Aviso de versión: SDK v2 vs v3

| v2 (material de estudio y docs pre-2025) | v3 (3.22.1, actual) |
|---|---|
| `sagemaker.estimator.Estimator` + `.fit()` | `sagemaker.train.model_trainer.ModelTrainer` + `.train()` |
| `sagemaker.processing.ScriptProcessor` / `SKLearnProcessor` + `.run()` | `sagemaker.core.processing` (`Processor`, `ScriptProcessor`, `FrameworkProcessor`) |

En v3 los módulos v2 **se eliminaron** (`ModuleNotFoundError: … was removed in the SageMaker Python SDK v3`). **En el examen, una pregunta sobre `Estimator.fit()` se refiere a esta misma mecánica**: una petición `CreateTrainingJob` construida por el SDK. **La capa de boto3 y de la CLI no cambió.**

---

## 23. Errores: dos momentos distintos

| Momento | Qué pasa |
|---|---|
| **En la llamada** | la API rechaza la petición y tu proceso recibe una excepción. **No se enciende ninguna máquina, no hay logs, no hay job que describir** |
| **Durante el job** | la llamada fue bien, el ARN existe, y el fallo aparece minutos después en `FailureReason`, con o sin logs según lo lejos que llegara |

Confundirlos es perder media hora mirando el sitio equivocado.

---

## 24. Los cuatro errores frecuentes

**`ValidationException`** (en la llamada) — campo obligatorio ausente, valor fuera de rango, tipo equivocado. El mensaje **nombra el campo** en la ruta de la petición. Casos típicos: omitir `TrainingInputMode`, hiperparámetro numérico sin comillas. Se arregla en el JSON, no en AWS.
→ Variante: relanzar con un nombre ya usado devuelve **`ResourceInUse`**, no `ValidationException`.

**`ResourceLimitExceeded`** (en la llamada) — **no es permisos, es cuota de servicio**:
- El nombre de la cuota **lleva el uso dentro**: `ml.m5.4xlarge for processing job usage` ≠ `… for training job usage` ≠ `… for endpoint usage`. **Tener sitio para entrenar no da sitio para procesar.**
- `is 0 Instances` es **lo normal** en una cuenta nueva para los tipos grandes. No es un castigo: hay que pedirla.
- Se consulta y se pide en **Service Quotas** (`list-services` → `list-service-quotas`), y las cuotas son **por cuenta y por región**.

**`AccessDenied` — dos errores distintos con el mismo nombre:**

| | Cuándo llega | Qué dice | Dónde se arregla |
|---|---|---|---|
| **Tuyo** | excepción de la llamada | `User: …:user/miguel is not authorized to perform: iam:PassRole` | **tu** política de identidad (`iam:PassRole` + condición `iam:PassedToService`) |
| **Del rol de ejecución** | `FailureReason`, minutos después | `Data download failed: Unable to download object s3://… (AccessDenied)` | política del **rol de ejecución** o la *bucket policy* |

> Ojo: un prefijo literal en la política (`fraude/2026/*`) hace que un archivo en `fraude/2025/` dé este mismo error con permisos "correctos".

**Imagen de otra región** — SageMaker **no valida que la imagen exista al crear el job**. La petición se acepta; minutos después, `Failed` con `FailureReason` sobre el registro y **sin ningún log** (el contenedor nunca existió). Tres causas, todas visibles en la URI:
1. cuenta/región de la URI ≠ región del job;
2. tag inexistente para ese framework;
3. imagen de inferencia en un job de entrenamiento, o al revés.

Prevención: `retrieve` convierte los tres en una excepción local e inmediata en vez de un job fallido diez minutos después.

---

## 25. Tabla de diagnóstico

| Síntoma | Dónde aparece | Qué tocar |
|---|---|---|
| `ValidationException` con nombre de campo | excepción de la llamada | el JSON de la petición |
| `ResourceInUse` | excepción de la llamada | el nombre del job |
| `ResourceLimitExceeded` | excepción de la llamada | Service Quotas (tipo **y uso**) |
| `AccessDeniedException` con `iam:PassRole` | excepción de la llamada | tu política de identidad |
| `Failed` + `AccessDenied` sobre S3 | `describe-*` | política del rol de ejecución o bucket policy |
| `Failed` sobre la imagen, **sin logs** | `describe-*` | la URI de imagen |
| `Failed` con excepción de Python | `describe-*` + CloudWatch Logs | tu script |
| **`Completed` pero S3 vacío** | `describe-*` dice `Completed` | la ruta donde escribe tu script vs. `LocalPath` |
| `Stopped` inesperado | `describe-*` | `MaxRuntimeInSeconds` |
| Atascado en `Starting` muchos minutos | `SecondaryStatus` | capacidad/cuota del tipo de instancia; **no es tu código** |

---

## 26. Diccionario de nombres

| Nombre | Qué es en realidad |
|---|---|
| `algo-1` | la primera (y única, si `InstanceCount: 1`) máquina del clúster; da nombre al stream de logs |
| `/opt/ml` | raíz del contrato de rutas SageMaker↔contenedor; **nada que ver** con "machine learning" como carpeta de proyecto |
| `model.tar.gz` | el empaquetado de `/opt/ml/model`; **no es un formato de modelo**, es un tar comprimido |
| `image_scope` | para qué sirve la imagen (`training` / `inference`), no su tamaño ni su alcance de red |
| `S3Prefix` | "el `S3Uri` es un prefijo, **descárgalo entero**", no "usa este prefijo como filtro" |
| `EndOfJob` | sube las salidas al final **y solo si el job terminó bien** |
| `ml.m5.xlarge` | tipo de instancia de SageMaker; comparte nombre con el de EC2, **no comparte cuota** |

---

## 27. Números a memorizar

| Cosa | Valor |
|---|---|
| `ProcessingJobName` | 1–63 caracteres, único **para siempre** en cuenta+región |
| `ProcessingInputs` / `Outputs` | máx. **10** cada uno |
| `ContainerEntrypoint` / `ContainerArguments` | máx. **100** elementos |
| `Environment` | máx. **100** variables (solo cadenas) |
| `InstanceCount` | **1–100** |
| `VolumeSizeInGB` | **1–16384** GB (obligatorio en Processing) |
| `MaxRuntimeInSeconds` | **default = 1 día**; gracia tras `SIGTERM` = **120 s** |
| `/opt/ml/output/failure` | **1024** primeros caracteres → `FailureReason` |
| Waiter Processing | 60 s × 60 intentos (1 h) |
| Waiter Training | 120 s × 180 intentos (6 h) |
| `aws logs tail` sin `--since` | últimos **10 minutos** |

---

## 28. Distractores y trampas de examen

- **`create-*` devuelve ARN ≠ el job corrió.** Solo certifica que la petición pasó validación y quedó registrada; ni hay máquinas encendidas ni datos descargados todavía.
- **`Completed` + S3 vacío** → el script escribió fuera del `LocalPath`, o no escribió nada y salió con código 0. **No** es falta de permisos (eso sería `Failed` con `AccessDenied` al subir) ni `MaxRuntimeInSeconds` (eso sería `Stopped`).
- **`Stopped` sin `FailureReason` + log que se corta justo al cumplirse el límite** = firma de `MaxRuntimeInSeconds`. La parada manual es *una* causa de `Stopped`, no la única.
- Para que un corte por tiempo no borre el trabajo hecho **y** sea evidente: `S3UploadMode: Continuous` **+** comprobar `== "Completed"` en la automatización. Quitar `StoppingCondition` es lo contrario de una solución: elimina el único freno a un job colgado que se factura por hora de instancia.
- **`FullyReplicated` + más instancias = triplicar el coste, no repartir el trabajo.** `ShardedByS3Key` + cálculo global = resultado incorrecto **sin ningún error**.
- **SageMaker copia, no consolida**: no fusiona las salidas de varias instancias en un solo archivo.
- **Sin stream de logs → el contenedor nunca arrancó** → datos, permisos o imagen; nunca tu código.
- Para una petición **versionada y revisable en PR**, la respuesta es **CLI con `--cli-input-json`**: el SDK rellena obligatorios con defaults suyos, de modo que **lo revisado no es lo enviado**.
- En **Lambda**, boto3 gana por peso de dependencia y por los defaults implícitos del SDK, no porque el SDK sea incapaz de fijar KMS o `TrainingInputMode` (ambos pueden las dos cosas).
- **`ResourceLimitExceeded` suena a permisos y no lo es.**

---

## Punteros a otras notas

`Pipe` / `FastFile` y formatos → [[Almacenamiento y formatos]] · script propio en contenedor de framework → [[Script mode]] · `MaxWaitTimeInSeconds`, spot y warm pools → [[Entrenar más rápido y barato]] · canales e hiperparámetros de XGBoost → [[Algoritmos integrados]] · imágenes propias en ECR → [[Contenedores y hosting múltiple]] · KMS → [[Protección de datos]] · `NetworkConfig` y VPC → [[Red]] · encadenar jobs → [[Orquestación]] · métricas del job → [[Observabilidad]]
