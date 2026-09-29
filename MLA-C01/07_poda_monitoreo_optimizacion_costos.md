# Capítulo 7 — Monitoreo de modelos y optimización de costos

**Objetivos del examen cubiertos**

- **Dominio 4: ML Solution Monitoring, Maintenance, and Security**
  - 4.1 Monitorear la inferencia del modelo
  - 4.2 Monitorear y optimizar la infraestructura y los costos

> **Nota de vigencia (septiembre de 2026).** Desde el 30 de julio de 2026, **SageMaker Model Monitor**, **SageMaker Clarify** y **Amazon Augmented AI (A2I)** ya no admiten clientes nuevos. Quienes ya los usan pueden seguir haciéndolo, pero no recibirán funciones nuevas. Como reemplazo de Model Monitor, AWS propone combinar sus soluciones open source de monitoreo para SageMaker AI con dashboards de QuickSight y con CloudWatch. Para Clarify propone una combinación parecida, que suma la librería SHAP y MLflow en SageMaker AI. El MLA-C01 y este capítulo los presentan como servicios vigentes, así que estúdialos de esa forma.

---

## 1. Monitoreo de la inferencia del modelo

Los modelos de ML se entrenan y evalúan con datos históricos, pero en producción reciben datos nuevos. Con el tiempo, la distribución de esos datos cambia por la evolución del comportamiento de los usuarios, la estacionalidad o factores externos. Entonces los patrones y relaciones que aprendió el modelo dejan de ser válidos. El resultado es una caída gradual del desempeño y de la calidad de las predicciones, llamada **model drift**.

La solución es **monitorear el modelo de forma continua y reentrenarlo cuando haga falta**. Esta es la última fase del ciclo de vida de ML, *Monitor Model* (Figura 7.1 del libro). Su objetivo es mantener la confiabilidad, la disponibilidad y el desempeño del modelo desplegado. El capítulo cubre dos frentes: las inferencias que produce el modelo y la infraestructura (y sus costos) que las genera.

Para el examen necesitas dominar dos cosas:

- **Cómo instrumentar la solución** con métricas, alertas y dashboards que sigan, por ejemplo, la exactitud (accuracy) del modelo, la latencia, el uso de recursos y el data drift.
- **Por qué importa cada métrica.** Por ejemplo, la exactitud revela cuándo se degrada el modelo, y el uso de recursos permite optimizar costos y evitar sobrecargas.

Estas prácticas están formalizadas en la **Machine Learning Well-Architected Lens**. La Lens aplica al ML los seis pilares del AWS Well-Architected Framework: excelencia operacional, seguridad, confiabilidad, eficiencia de desempeño, optimización de costos y sostenibilidad.

### 1.1 Drift en los modelos

En SageMaker AI, detectar y corregir el drift a tiempo es clave para que los modelos desplegados sigan siendo exactos. El servicio principal para esto es **Amazon SageMaker Model Monitor**. Con *monitoring schedules* (programaciones de monitoreo), captura datos de los endpoints, los analiza en busca de drift y genera alertas cuando detecta cambios significativos.

Estos son los tipos de drift que debes distinguir:

| Tipo | Qué cambia |
|---|---|
| **Data drift** | Las propiedades estadísticas de los datos de entrada. |
| **Model drift** (de calidad) | Empeoran las métricas de desempeño del modelo (accuracy, precision, recall…). |
| **Bias drift** | La equidad (fairness) de las predicciones entre distintos grupos. |
| **Feature attribution drift** | La importancia relativa de las features en las predicciones. |

El capítulo también menciona el **concept drift**. En él cambia la relación entre las entradas y la variable objetivo, aunque las entradas mantengan su distribución. Por eso solo se detecta al comparar las predicciones con las etiquetas reales, es decir, con el monitoreo de calidad del modelo.

### 1.2 Técnicas para monitorear la calidad de los datos y el desempeño del modelo

SageMaker Model Monitor supervisa la calidad de los modelos en producción de tres formas:

- Monitoreo continuo de un **endpoint en tiempo real**.
- Monitoreo continuo de un **batch transform** que se ejecuta periódicamente.
- Monitoreo **programado** (on-schedule) de trabajos de batch transform asíncronos.

Model Monitor se integra con **alarmas de Amazon CloudWatch** que avisan cuando hay drift en los datos o en la calidad del modelo. Detectarlo pronto permite actuar sin vigilancia manual ni herramientas extra: reentrenar, auditar los sistemas upstream o corregir problemas de calidad de datos. Puedes usar las capacidades de monitoreo preconstruidas, que no requieren código, o programar un análisis personalizado.

Model Monitor ofrece cuatro tipos de monitoreo:

1. **Data quality**: drift en la calidad de los datos de entrada.
2. **Model quality**: drift en las métricas de calidad del modelo, como accuracy.
3. **Bias drift** (modelos en producción): cambios en el sesgo de las predicciones.
4. **Feature attribution drift** (modelos en producción): cambios en la importancia de las features.

Los dos últimos se apoyan en SageMaker Clarify y pueden verse como parte del monitoreo de calidad del modelo. El bias drift comprueba que el modelo siga siendo equitativo entre grupos, algo clave para su calidad y para un despliegue ético. El feature attribution drift detecta cambios en la relevancia de las features. Esos cambios alteran cómo decide el modelo y, por tanto, su desempeño.

La Figura 7.2 del libro ubica estos tipos de monitoreo dentro del proceso general:

- **Monitoreo de calidad de datos.** Se crea un *baseline* (línea base) a partir de los datos de entrenamiento y se compara continuamente con los datos entrantes. Si los datos entrantes cambian, se detecta drift.
- **Monitoreo de calidad del modelo.** Se comparan las predicciones con los resultados reales (etiquetas o *ground truth*). Si pronosticas las ventas diarias, comparas el pronóstico con las ventas reales del día siguiente. En algunos casos, conseguir el resultado real exige un esfuerzo adicional. En un recomendador de productos, por ejemplo, quizá tengas que encuestar a un grupo de clientes para saber si las recomendaciones coincidieron con sus preferencias.
- **Atribución de features.** Como viste en el capítulo 3, **SageMaker Clarify** muestra cuánto pesa cada feature en las predicciones (*feature attribution*). Esto ayuda a entender las decisiones del modelo y puede revelar sesgos o patrones inesperados. En un modelo de aprobación de préstamos, Clarify indica qué features (puntaje crediticio, nivel de ingresos, historial laboral) influyeron más en una predicción concreta. Hay **feature attribution drift** cuando la importancia de las features calculada sobre los datos de producción se aparta de la del baseline calculado con los datos de entrenamiento.

Evaluar el drift consiste en monitorear los datos, detectar cambios y disparar acciones cuando haga falta. En CloudWatch puedes definir reglas y umbrales para recibir notificaciones de drift y automatizar las acciones correctivas.

**Arquitectura de referencia (Figura 7.3 del libro).** La figura representa el monitoreo de la inferencia en tiempo real. A la izquierda están las fuentes de datos de entrenamiento y de producción. En el centro, los componentes de monitoreo (calidad de datos y calidad del modelo). A la derecha, los hallazgos. Para la inferencia batch se arma una arquitectura análoga. Los datos de entrada se limpian y preparan con **SageMaker Data Wrangler**. Las features se guardan en **SageMaker Feature Store**, un repositorio totalmente administrado para almacenar, actualizar, recuperar y compartir features. Ambos servicios se vieron en el capítulo 3. Esta arquitectura es una vista de bajo nivel del proceso de la Figura 7.2.

### 1.3 Flujo de monitoreo

Las tareas de detección de drift pueden incluirse en el flujo de ML con **SageMaker Pipelines**. Los hallazgos (*findings*) quedan en Amazon S3, donde Model Monitor escribe sus reportes en JSON. Desde ahí se pueden analizar con **Amazon Athena** y **Amazon QuickSight**.

**Requisito previo: habilitar la captura de datos (data capture).**

- **Endpoint en tiempo real:** se capturan las solicitudes entrantes y las predicciones que devuelve el modelo.
- **Batch transform:** se capturan las entradas y salidas del trabajo.

Después, el flujo sigue los cinco pasos de la Figura 7.3:

| Paso | Tarea | Qué hace |
|---|---|---|
| 1 | Baseline | Calcula estadísticas y restricciones de referencia y las guarda en S3. |
| 2 | Monitoreo de data drift | Compara los datos capturados con el baseline de datos. |
| 3 | Monitoreo de model drift | Compara las predicciones con las etiquetas reales. |
| 4 | CloudWatch | Recibe los resultados de los pasos 2 y 3 y dispara alarmas. |
| 5 | Reentrenamiento | Acción automática ante una alarma: actualizar datos, recalcular el baseline y reentrenar. |

**Paso 1: baseline.** Model Monitor genera un perfil de referencia y lo guarda en S3 como estadísticas más un conjunto de restricciones (*constraints*). Hay dos baselines:

- El **baseline de calidad de datos** se calcula sobre el dataset de entrenamiento e incluye estadísticas por feature.
- El **baseline de calidad del modelo** se calcula sobre un dataset de validación que contiene las predicciones del modelo y sus etiquetas. Fija restricciones sobre las métricas: por ejemplo, que el recall no baje de 0.8 o que la precision no baje de 0.9.

Después, las predicciones (en tiempo real o batch) se comparan con estas restricciones, y lo que quede fuera de ellas se reporta como **violación**. El momento de cálculo es distinto en cada baseline. El de datos solo necesita los datos de entrenamiento, así que en un pipeline puede calcularse antes del paso de entrenamiento. El del modelo necesita las predicciones de un modelo ya entrenado, así que se calcula después. Ambos deben recalcularse cada vez que se reentrena el modelo.

**Paso 2: monitoreo de data drift.** Según la programación definida (por ejemplo, cada hora o cada día), analiza los datos de entrada capturados y los compara con el baseline. Revisa estadísticas como la media, la mediana y la desviación estándar de las features, o la tasa de valores faltantes. Las violaciones se reportan en S3 y las métricas se publican en CloudWatch (paso 4), donde una alarma avisa si se desvían del baseline. La tarea corre en su propio cómputo, con **Deequ** (una herramienta open source construida sobre Apache Spark). Así no ralentiza la inferencia y escala con el volumen de datos. La frecuencia de ejecución, que controla el costo, se ajusta según qué tan rápido esperas que cambien los datos.

**Paso 3: monitoreo de model drift.** Mide la calidad del modelo comparando sus predicciones con las etiquetas reales. Para ello, cada cierto tiempo etiquetas una parte de los datos capturados por el endpoint (o por el batch transform), subes esas etiquetas a S3 e indicas su ubicación como parámetro al crear el trabajo de monitoreo. Igual que en el paso 2, los resultados se registran en CloudWatch (paso 4) y las violaciones se reportan en un bucket de S3.

**Paso 4: Amazon CloudWatch.** Es el servicio de monitoreo y observabilidad que gestiona métricas, logs y alarmas. Sobre las métricas de drift o de desempeño defines alarmas que notifican por email o SMS mediante **Amazon SNS**.

**Paso 5: reentrenamiento.** El cambio de estado de una alarma puede disparar el reentrenamiento automáticamente. Una regla de **Amazon EventBridge** reacciona a la alarma e inicia el pipeline de SageMaker, o bien una función Lambda o un flujo de Step Functions.

> **Para el examen: logs, eventos y alarmas**
> - **Logs:** registros detallados de lo que ocurre en aplicaciones, sistemas o servicios. Sirven para el troubleshooting y el análisis.
> - **Eventos:** hechos o cambios de estado de tus recursos que quieres vigilar y sobre los que quieres actuar, como una falla o un cambio de configuración.
> - **Alarmas:** notificaciones que se disparan al cruzarse un umbral o una condición, como las restricciones del baseline. Además de avisar, pueden iniciar acciones automáticas (directamente o vía EventBridge): actualizar los datos de entrenamiento o los hiperparámetros, recalcular el baseline y reentrenar el modelo (paso 5).

---

## 2. Principios de diseño para el monitoreo

El monitoreo observa en vivo el desempeño del modelo y de su infraestructura en producción. Por eso el examen exige conocer los principios que guían el diseño de un marco de monitoreo robusto, eficiente y sostenible. La tabla siguiente (Figura 7.4 del libro) resume los principios de la ML Well-Architected Lens para esta fase:

| Pilar | Principio |
|---|---|
| Excelencia operacional | MLOE-15: Enable model observability and tracking |
| | MLOE-16: Synchronize architecture and configuration, and check for skew across environments |
| Seguridad | MLSEC-12: Restrict access to intended legitimate consumers |
| | MLSEC-13: Monitor human interactions with data for anomalous activity |
| Confiabilidad | MLREL-12: Allow automatic scaling of the model endpoint |
| | MLREL-13: Ensure a recoverable endpoint with a managed version control strategy |
| Eficiencia de desempeño | MLPER-13: Evaluate model explainability |
| | MLPER-14: Evaluate data drift |
| | MLPER-15: Monitor, detect, and handle model performance degradation |
| | MLPER-16: Establish an automated re-training framework |
| | MLPER-17: Review for updated data/features for retraining |
| | MLPER-18: Include human-in-the-loop monitoring |
| Optimización de costos | MLCOST-27: Monitor usage and cost by ML activity |
| | MLCOST-28: Monitor Return on Investment for ML models |
| | MLCOST-29: Monitor endpoint usage and right-size the instance fleet |
| Sostenibilidad | MLSUS-15: Measure material efficiency |
| | MLSUS-16: Retrain only when necessary |

> Los identificadores son los de la versión de la Lens que usa el libro. La versión actual los renumeró; por ejemplo, MLOE-15 pasó a ser MLOPS06-BP02. Para el examen importan el nombre del principio y su pilar, no el código. Más información: https://docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens/best-practices-by-ml-lifecycle-phase.html

A continuación, cada pilar con sus principios y cómo se implementan.

### 2.1 Excelencia operacional

La excelencia operacional (OE) consiste en construir el software de la manera correcta y ofrecer siempre una buena experiencia al cliente. Incluye buenas prácticas para organizar los equipos, diseñar el trabajo, operarlo eficazmente a gran escala, adaptarlo a los cambios y mejorarlo continuamente.

#### MLOE-15 · Enable model observability and tracking

Una aplicación de ML bien arquitecturada incluye un marco para que el equipo observe de forma proactiva las métricas vitales del modelo. Entre ellas están la salud operativa de las instancias que alojan el endpoint y la salud de las respuestas en la inferencia en tiempo real.

**Implementación:**

- **SageMaker Model Monitor:** monitorea continuamente la calidad de los modelos en producción y la compara con los resultados del entrenamiento.
- **Amazon CloudWatch:** recopila y analiza estadísticas de uso de los modelos.
- **SageMaker Model Dashboard:** portal centralizado para ver, buscar y explorar tus modelos.
- **SageMaker Clarify:** detecta sesgos que surgen durante el entrenamiento o en producción.
- **SageMaker ML Lineage Tracking:** guarda el historial (linaje) de los experimentos y artefactos que dieron origen a cada modelo.
- **SageMaker Model Cards:** reúnen la información del modelo, como los requisitos de negocio, las decisiones clave y las observaciones del desarrollo y la evaluación.
- **Validación automatizada de SageMaker (shadow testing):** compara en tiempo real el desempeño de un modelo nuevo con el del modelo en producción, usando las mismas solicitudes de inferencia reales.

#### MLOE-16 · Synchronize architecture and configuration, and check for skew across environments

Busca que los sistemas y las configuraciones sean idénticos en las fases de desarrollo y de despliegue.

**Implementación:**

- **AWS CloudFormation:** aprovisiona los recursos de forma rápida y consistente con infraestructura como código (IaC).
- **SageMaker Model Monitor:** compara continuamente lo que ocurre en producción con los resultados del entrenamiento.

### 2.2 Seguridad

El pilar de seguridad construye las protecciones de todos los componentes de la aplicación de ML: las identidades (usuarios y roles), la infraestructura (cómputo y red), la aplicación que sirve el modelo y, sobre todo, los datos. Los datos no son solo los de entrenamiento, validación y prueba. También incluyen los artefactos del modelo (`model.tar.gz`), que guardan sus parámetros y configuraciones. El capítulo 8 trata este pilar a fondo.

#### MLSEC-12 · Restrict access to intended legitimate consumers

Aplica el principio de mínimo privilegio a todos los consumidores del endpoint del modelo.

**Implementación:** trata el endpoint de inferencia como cualquier otra API HTTPS. Aplica controles de red, como restringir el acceso a ciertos rangos de IP, y control de bots. Además, firma las solicitudes HTTPS para verificar la identidad del solicitante y la integridad de la solicitud; el cifrado TLS protege los datos en tránsito.

#### MLSEC-13 · Monitor human interactions with data for anomalous activity

Busca detectar accesos anómalos o no autorizados de personas a los datos.

**Implementación:**

- Habilitar el **registro de acceso a datos**, para rastrear los eventos asociados a las solicitudes de inferencia del endpoint.
- **Amazon Macie:** descubre y clasifica los datos sensibles de S3 (por ejemplo, PII) según su nivel de sensibilidad.
- **Amazon GuardDuty:** detecta actividad maliciosa o no autorizada, como accesos no autorizados a cuentas de AWS, a cargas de trabajo y a datos en S3.

### 2.3 Confiabilidad

El pilar de confiabilidad se centra en que la aplicación de ML cumpla su función **correctamente** y **de forma consistente** cuando se espera que lo haga. Correctamente significa que cumple todos los requisitos de negocio para los que se diseñó. De forma consistente significa que su desempeño siempre es satisfactorio para el usuario. Por eso las métricas de desempeño necesitan umbrales (límites inferior y superior) que marquen lo aceptable, como la disponibilidad (uptime) o la latencia. Más sobre site reliability engineering (SRE): https://aws.amazon.com/what-is/sre

#### MLREL-12 · Allow automatic scaling of the model endpoint

El endpoint escala automáticamente para procesar las predicciones de forma confiable y elástica cuando cambia la demanda de inferencia.

**Implementación:** como viste en el capítulo 6, se usa el **auto scaling de endpoints de SageMaker AI**, que funciona sobre Application Auto Scaling, con políticas de target tracking, step scaling o scheduled scaling.

#### MLREL-13 · Ensure a recoverable endpoint with a managed version control strategy

El endpoint de inferencia debe ser tolerante a fallos, y para eso deben serlo todos sus componentes «tras bambalinas»: una cadena es tan fuerte como su eslabón más débil. Según las Figuras 6.3 y 6.6 del capítulo 6, esos componentes incluyen las instancias del endpoint, la endpoint configuration, las imágenes de contenedor de inferencia y los artefactos del modelo, entre otros.

**Implementación:**

- **SageMaker Pipelines y SageMaker Projects:** automatizan el flujo de ML completo, incluida la recuperación ante fallos.
- **AWS CloudFormation** (u otra herramienta de IaC): reconstruye programáticamente la infraestructura sobre la que opera la aplicación.
- **Amazon ECR:** almacena de forma segura y versionada las imágenes de contenedor de entrenamiento e inferencia que usa el modelo.

### 2.4 Eficiencia de desempeño

Este pilar optimiza los recursos de cómputo en la nube para alcanzar el desempeño que necesita la aplicación de ML. Esa eficiencia debe adaptarse a la demanda cambiante y a la evolución de la tecnología. Dicho de otro modo: usar los recursos correctos en el momento correcto, sin gastar de más ni quedarse corto. Cumplir todas esas restricciones obliga al ingeniero de ML a hacer compromisos (trade-offs), y los siguientes principios ayudan a elegirlos.

#### MLPER-13 · Evaluate model explainability

Consiste en evaluar si las predicciones del modelo pueden entenderse y justificarse en términos humanos, según las necesidades de negocio y los requisitos de cumplimiento. En la práctica es un compromiso entre la explicabilidad y la complejidad del modelo.

**Implementación:** **SageMaker Clarify** (capítulo 3) explica los resultados del modelo y detecta posibles sesgos. Sus funciones de fairness y explicabilidad ayudan a construir modelos menos sesgados, más comprensibles y más equilibrados.

#### MLPER-14 · Evaluate data drift

Consiste en evaluar los cambios significativos en la distribución de los datos de entrada a lo largo del tiempo, que pueden reducir la exactitud del modelo sobre datos nuevos.

**Implementación:**

- **SageMaker Model Monitor:** detecta de forma continua, según su programación, tanto el data drift como el deterioro de la calidad del modelo (que es como se manifiesta el concept drift).
- **SageMaker Clarify:** ayuda a identificar sesgos potenciales en los modelos.

#### MLPER-15 · Monitor, detect, and handle model performance degradation

Consiste en monitorear continuamente los modelos desplegados para detectar y corregir la caída de su desempeño. Incluye detectar data drift, concept drift y sesgo, y actuar (reentrenar o actualizar el modelo) cuando aparece la degradación.

**Implementación:**

- **SageMaker Model Monitor:** monitorea continuamente la calidad de los modelos en producción.
- **SageMaker Auto Scaling:** ajusta a la demanda los recursos de cómputo del endpoint. Resuelve la degradación operativa (latencia, throughput), no la de la calidad de las predicciones.

#### MLPER-16 · Establish an automated re-training framework

Consiste en montar un sistema automatizado que reentrene los modelos cuando las métricas monitoreadas (data drift, degradación del desempeño) lo indiquen. Así el modelo se mantiene al día con los cambios en los datos.

**Implementación:**

- **SageMaker Model Monitor:** detecta el evento que justifica reentrenar (por ejemplo, un drift que supera un umbral). Su alarma de CloudWatch dispara el reentrenamiento vía EventBridge.
- **SageMaker Pipelines o AWS Step Functions:** orquestan los pasos del reentrenamiento.

#### MLPER-17 · Review for updated data/features for retraining

Consiste en revisar periódicamente los datos y las features que se usan para reentrenar, e incorporar los que hayan surgido desde el entrenamiento inicial, para que el modelo siga siendo exacto y relevante. Implica repetir la exploración de datos y el feature engineering a intervalos fijados según la volatilidad y la disponibilidad de los datos.

**Implementación:** **SageMaker Data Wrangler** (capítulo 3) para preparar los datos.

#### MLPER-18 · Include human-in-the-loop monitoring

Consiste en incorporar supervisión humana al monitoreo. Revisores humanos evalúan las inferencias del modelo, sobre todo las de baja confianza o muestras aleatorias. Al comparar sus etiquetas con las predicciones se detecta la degradación del desempeño.

**Implementación:** **Amazon Augmented AI (A2I)** para obtener revisión humana de predicciones de baja confianza o de muestras aleatorias (ver la nota de vigencia al inicio).

### 2.5 Optimización de costos

Este pilar busca diseñar, construir y operar aplicaciones de ML que ofrezcan la funcionalidad de negocio necesaria al menor costo posible: las inferencias más exactas por el menor gasto.

#### MLCOST-27 · Monitor usage and cost by ML activity

Se basa en el **etiquetado (tagging)**: adjuntar a cada recurso metadatos en forma de pares clave-valor. Con etiquetas puedes agrupar los recursos por propósito, responsable, entorno o cualquier otro criterio, lo que facilita buscarlos, filtrarlos y analizar sus costos. Es especialmente útil para asignar costos a actividades concretas, como el reentrenamiento y el hosting, mediante etiquetas dedicadas. Para que aparezcan en los reportes de facturación, las etiquetas deben activarse como *cost allocation tags*.

> El etiquetado de recursos es una capacidad clave de **FinOps**, el conjunto de prácticas para optimizar los costos en la nube. Más información: https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/tags-for-cost-allocation-and-financial-management.html

**Implementación:**

- **Etiquetas de AWS:** agrupan los recursos de ML por propósito (por ejemplo, la fase de ML), responsable, entorno y otros metadatos.
- **AWS Budgets:** hace seguimiento de los costos.

#### MLCOST-28 · Monitor Return on Investment for ML models

Consiste en crear reportes que comparen el valor que aporta el modelo en producción con su costo de ejecución, medido con KPIs de negocio. Esos KPIs deben definirse en la primera fase del ciclo de vida de ML, la definición del problema (Figura 7.5 del libro), para que los objetivos y los criterios de éxito queden claros desde el principio. Por ejemplo, si el modelo apoya la captación de clientes, mides cuántos clientes nuevos se captan en un periodo y cuánto gastan cuando se usa la predicción, frente a un baseline sin ella.

Los reportes permiten decidir según el retorno de la inversión (ROI). Si es positivo, puedes extender el modelo a problemas similares. Si es negativo, buscas acciones correctivas que reduzcan el costo de ejecución. Una opción es la **serverless inference** para tráfico intermitente: ahorra costo, aunque no mejora la latencia por los *cold starts*.

**Implementación:** una herramienta de reportes como **Amazon QuickSight**, para crear reportes de negocio que muestren el valor del modelo en términos de KPIs.

#### MLCOST-29 · Monitor endpoint usage and right-size the instance fleet

Consiste en usar el cómputo de producción de forma eficiente: monitorear el uso del endpoint y escalar las instancias elásticamente (scale-in y scale-out) según la demanda.

**Implementación:**

- **Amazon CloudWatch:** monitorea los endpoints de SageMaker.
- **SageMaker Auto Scaling:** ajusta a la demanda los recursos de cómputo del endpoint.
- Diseñar una **Frugal Architecture** para toda la solución de ML.

> **Frugal Architecture:** filosofía de diseño que trata el costo y la sostenibilidad como requisitos no funcionales críticos, al mismo nivel que la seguridad, el cumplimiento y el desempeño. La presentó Werner Vogels (CTO de Amazon) en re:Invent 2023. Ejemplo: si necesitas acceso rápido a grandes volúmenes de datos de entrenamiento con **Amazon FSx for Lustre** (capítulo 2), despliégalo en la misma zona de disponibilidad (AZ) que SageMaker AI para evitar los costos de transferencia de datos entre AZ. Más información: https://aws.amazon.com/blogs/architecture/achieving-frugal-architecture-using-the-aws-well-architected-framework-guidance

### 2.6 Sostenibilidad

Este pilar busca reducir el impacto ambiental, sobre todo el consumo de energía, para usar los recursos de forma eficiente y responsable y disminuir la huella de carbono. En ML es crítico por la enorme capacidad de cómputo que exige entrenar o ajustar modelos fundacionales como GPT-3, que tiene 175 000 millones de parámetros.

#### MLSUS-15 · Measure material efficiency

Consiste en medir cuántos recursos aprovisionados consume la carga de ML por unidad de trabajo, es decir, su «eficiencia material». Así puedes compararte con un punto de referencia e identificar mejoras que reduzcan el consumo sin perder desempeño.

**Implementación:**

- Definir los **recursos aprovisionados por unidad de trabajo** como uno de tus KPIs, para normalizar los KPIs de sostenibilidad y comparar su evolución.
- Establecer un **baseline** de eficiencia material como punto de referencia para medir las mejoras a medida que aplicas optimizaciones.
- **Estimar las mejoras** para cuantificar el ROI de esas optimizaciones.

#### MLSUS-16 · Retrain only when necessary

Como se vio en el capítulo 6, entrenar es costoso. Por eso este principio indica reentrenar solo cuando haga falta, no con un calendario fijo ni con demasiada frecuencia. Para lograrlo, monitoreas el desempeño en producción y reentrenas solo cuando detectas un drift significativo.

**Implementación:**

- Definir con los stakeholders de negocio **KPIs** del problema, como una exactitud mínima aceptable y un error máximo aceptable.
- **SageMaker Model Monitor:** dispara el reentrenamiento solo cuando el drift supera un umbral definido.
- **SageMaker Pipelines o AWS Step Functions:** automatizan los pipelines de reentrenamiento.

---

## 3. Monitoreo de infraestructura y costos

Hasta aquí viste cómo monitorear la capacidad del modelo de producir inferencias exactas y confiables. Pero el desempeño del modelo es solo una parte: la infraestructura que lo soporta y sus costos también deben monitorearse.

- **Infraestructura.** Hay que seguir métricas como el uso de CPU y memoria, la E/S de disco, el throughput de red y la salud de las instancias, para detectar y resolver problemas rápidamente. Además de CloudWatch (alarmas y notificaciones), puedes usar AWS X-Ray (trazado y depuración de aplicaciones distribuidas), Amazon GuardDuty (monitoreo continuo de seguridad) y Amazon Inspector (evaluaciones de seguridad automatizadas), entre otros.
- **Costos.** AWS Cost Explorer, AWS Trusted Advisor y AWS Budgets muestran los patrones de gasto y pronostican los costos futuros. Las alertas de costo y uso evitan gastos inesperados. Los **SageMaker Savings Plans** reducen el costo respecto de On-Demand a cambio de comprometer un uso constante durante 1 o 3 años.

### 3.1 Servicios de monitoreo y observabilidad

Estos servicios son la base de las prácticas de **Site Reliability Engineering (SRE)**, mencionadas en el pilar de confiabilidad. SRE mantiene la escalabilidad y la estabilidad de las aplicaciones mediante operaciones automatizadas y monitoreo proactivo. En ML, garantiza que el modelo y su infraestructura produzcan inferencias exactas de forma consistente. Estos servicios también apoyan otros pilares:

- **Excelencia operacional:** agilizan el monitoreo y el troubleshooting.
- **Optimización de costos:** muestran cómo asignar mejor los recursos.
- **Sostenibilidad:** favorecen el uso eficiente de los recursos.

Instrumentar tu carga de ML con estos servicios equivale, en la práctica, a aplicarle la Well-Architected Lens.

Para el examen debes saber qué hace cada servicio y cuándo usarlo según el caso:

| Servicio | Para qué sirve |
|---|---|
| CloudWatch Logs Insights | Consultar y analizar logs. |
| EventBridge | Enrutar eventos y reaccionar ante ellos. |
| CloudTrail | Auditar quién hizo qué llamada a la API y cuándo. |
| X-Ray | Trazar solicitudes a través de servicios distribuidos. |
| GuardDuty | Detectar amenazas y actividad maliciosa. |
| Inspector | Detectar vulnerabilidades de software y exposición de red. |
| Security Hub | Centralizar los hallazgos de seguridad y evaluar el cumplimiento de buenas prácticas. |

#### Amazon CloudWatch Logs Insights

Permite buscar y analizar de forma interactiva grandes volúmenes de logs almacenados en **CloudWatch Logs**. Con sus lenguajes de consulta puedes hacer análisis de causa raíz y agilizar la gestión de incidentes y problemas.

Admite tres lenguajes de consulta:

- **Logs Insights QL:** lenguaje propio, con pocos comandos simples pero potentes.
- **OpenSearch PPL** (Piped Processing Language): comandos encadenados con pipes (`|`).
- **OpenSearch SQL:** sintaxis declarativa al estilo de SQL.

Otras funciones que facilitan el análisis:

- Descubrimiento automático de los campos de los logs de servicios como Route 53, Lambda, CloudTrail y VPC.
- Índices de campos (*field indexes*), que reducen el costo y aceleran las consultas sobre muchos grupos o eventos de log.
- Cifrado de los resultados de las consultas con AWS KMS, para proteger los datos sensibles de los logs.
- Detección y análisis de patrones.
- Consultas guardadas, que además pueden agregarse a dashboards.

**Casos de uso en ML:**

- **Trabajos de entrenamiento:** analizar los logs de los training jobs de SageMaker para detectar problemas o ineficiencias, como cuellos de botella de recursos o inconsistencias en los datos.
- **Endpoints:** seguir los logs de las solicitudes a los endpoints en tiempo real y diagnosticar problemas de inferencia.
- **Batch transform:** consultar los logs de estos trabajos para depurar errores y comprobar que el procesamiento funcione bien.

#### Amazon EventBridge

Es un bus de eventos serverless que conecta aplicaciones con datos de distintas fuentes. Enruta los eventos de las fuentes compatibles hacia servicios de AWS y destinos personalizados.

> Una **arquitectura orientada a eventos** (event-driven) crea sistemas débilmente acoplados que interactúan generando eventos y reaccionando a ellos. Este estilo aporta agilidad y permite construir aplicaciones confiables y escalables. Más información: https://aws.amazon.com/event-driven-architecture

En ML, EventBridge dispara acciones en respuesta a eventos y comunica los componentes del pipeline. **Complementa** a los orquestadores (SageMaker Pipelines, Step Functions), pero no los reemplaza. Se integra con Lambda, Step Functions y SageMaker AI. Por ejemplo, cuando termina un training job o se detecta una anomalía en el desempeño del modelo, puede invocar una función Lambda o iniciar un pipeline que preprocese datos, despliegue el modelo o lo reentrene.

También se combina con otros servicios de observabilidad:

- Puede enviar eventos a **CloudWatch** para crear métricas personalizadas, dashboards y alarmas.
- **X-Ray** puede trazar los eventos que pasan por EventBridge para mostrar dependencias, latencias y cuellos de botella.
- **Security Hub** y **GuardDuty** publican sus hallazgos como eventos, lo que permite responder a incidentes de forma coordinada.

**Casos de uso:**

- Automatizar el reentrenamiento ante disparadores predefinidos, como la llegada de nuevos datos de entrenamiento o la detección de model drift.
- Encadenar las etapas de un pipeline (preprocesamiento, feature engineering, entrenamiento, despliegue), reaccionando al fin de cada una. La orquestación con estado sigue a cargo de Pipelines o Step Functions.
- Notificar a los stakeholders el estado de los trabajos de ML, las métricas de desempeño y los posibles problemas.

#### AWS CloudTrail

Permite la auditoría, el cumplimiento y la gestión operacional y de riesgos de tus cuentas de AWS. Registra las acciones (llamadas a la API) de usuarios, roles y servicios, es decir, **quién hizo qué y cuándo**. En ML registra, por ejemplo, la creación, la actualización y la eliminación de recursos de SageMaker AI como training jobs, endpoints e instancias. Esa trazabilidad es esencial para depurar, auditar y cumplir normativas, y para reconstruir la secuencia de acciones en la infraestructura.

**Casos de uso:**

- Monitorear los patrones de acceso y uso de los recursos de SageMaker AI para detectar actividad inusual o no autorizada y responder a amenazas.
- Demostrar que se cumplen los estándares de gobernanza de datos y de seguridad.
- Integrarlo con CloudWatch para crear alarmas ante actividades o anomalías específicas.

#### AWS X-Ray

Es un servicio de trazado (tracing) que recopila datos detallados de las solicitudes que recibe tu aplicación. Permite verlos, filtrarlos y analizarlos para encontrar problemas y oportunidades de optimización. Muestra el ciclo de vida completo de cada solicitud, incluidas las llamadas a recursos downstream: servicios de AWS, microservicios, bases de datos y APIs web. Su **mapa de servicios** visual muestra cómo interactúan los componentes, lo que facilita localizar dependencias, cuellos de botella y errores.

SageMaker AI no está entre los servicios con integración nativa de X-Ray. Lo que se traza es la aplicación que invoca el endpoint (por ejemplo, API Gateway y Lambda). Allí la llamada al endpoint aparece como un segmento downstream con su latencia. Para trazar lo que ocurre dentro del contenedor hay que instrumentar el código.

**Casos de uso en ML:**

- **Trazado de solicitudes de inferencia:** seguir el flujo de una solicitud a lo largo de sus componentes (preprocesamiento, inferencia, postprocesamiento) para diagnosticar problemas de servicio del modelo y tiempos de respuesta, y optimizar la etapa que más latencia agrega.
- **Trazado de pipelines de extremo a extremo:** en pipelines con varios pasos y servicios, visualizar el flujo completo para ver cómo interactúan los servicios y detectar dependencias o problemas de desempeño.

#### Amazon GuardDuty

Es un servicio de **detección de amenazas** que monitorea y analiza continuamente fuentes de datos y logs de tu entorno, como los eventos de CloudTrail, los VPC Flow Logs y los logs DNS. Combina inteligencia de amenazas (listas de IPs, dominios y hashes de archivos maliciosos) con modelos de ML para identificar actividad sospechosa en tus cuentas, tus datos y tus recursos de AWS.

GuardDuty genera **hallazgos (findings)** detallados, pero **no remedia** las amenazas por sí mismo. La respuesta se automatiza con otros servicios:

- **Amazon EventBridge** (antes CloudWatch Events), que enruta los hallazgos.
- **AWS Lambda**, que ejecuta la remediación.
- **AWS Security Hub**, que centraliza los hallazgos.

Así, las amenazas se detectan y se atienden rápidamente, lo que reduce el riesgo de filtraciones, exfiltración de datos o accesos no autorizados al entorno de SageMaker AI.

**Casos de uso en ML:**

- **Protección de datos:** detectar patrones de acceso inusuales o intentos de exfiltrar datos desde notebooks o almacenamiento, para proteger los datos de entrenamiento y los artefactos del modelo.
- **Llamadas a la API sospechosas:** detectar actividad anómala sobre los recursos de SageMaker AI, como accesos desde IPs maliciosas o con credenciales comprometidas.
- **Integridad de los recursos:** detectar instancias comprometidas o comportamientos anómalos en la infraestructura que soporta las cargas de ML.

Evaluar el cumplimiento de buenas prácticas y detectar vulnerabilidades no es tarea de GuardDuty, sino de Security Hub e Inspector.

#### Amazon Inspector

Es un servicio automatizado de **gestión de vulnerabilidades**. Escanea de forma continua las instancias EC2 y las imágenes de contenedor de Amazon ECR, como las imágenes personalizadas de entrenamiento o inferencia. Busca vulnerabilidades de software conocidas (CVE), como software sin parches o dependencias desactualizadas, y exposición de red no deseada. Así garantiza que el cómputo sobre el que corren los entornos de SageMaker AI sea seguro y esté actualizado.

| | GuardDuty | Inspector |
|---|---|---|
| Enfoque | Detección continua de amenazas (actividad sospechosa) | Gestión de vulnerabilidades y exposición |
| Pregunta que responde | ¿Alguien está atacando o usando indebidamente mis recursos? | ¿Mis recursos tienen debilidades explotables? |
| Ejemplo | Acceso desde una IP maliciosa | Paquete con una CVE en una imagen de ECR |

**Casos de uso en ML:** identificar y remediar las debilidades de los entornos de cómputo de entrenamiento e inferencia, para proteger la confidencialidad y la integridad de los datos de entrenamiento, los artefactos del modelo y los resultados de inferencia. Si lo integras con AWS Lambda y **AWS Systems Manager**, puedes automatizar la remediación de las vulnerabilidades. Junto con GuardDuty, forma una estrategia completa: detección de amenazas en tiempo real más gestión continua de vulnerabilidades.

#### AWS Security Hub

Es un servicio de **gestión de la postura de seguridad en la nube (CSPM)**. Agrega, organiza y prioriza en un solo lugar los hallazgos de seguridad de varios servicios de AWS. Además, evalúa continuamente la configuración de tus recursos frente a buenas prácticas de seguridad. No detecta amenazas por sí mismo: consume los hallazgos de otros servicios.

Integraciones clave:

- **GuardDuty:** aporta la detección continua de amenazas, como intentos de acceso no autorizado o patrones inusuales de acceso a datos. Security Hub centraliza y prioriza esos hallazgos para responder rápido a incidentes que afecten a SageMaker AI.
- **Inspector:** aporta las vulnerabilidades y las desviaciones de buenas prácticas en el cómputo y los contenedores. Security Hub las agrega con recomendaciones de remediación.
- **AWS Config:** registra continuamente la configuración de los recursos y sus cambios, para comprobar que se cumplen las políticas de seguridad.
- **IAM Access Analyzer:** identifica permisos excesivos o no deseados.

**Casos de uso en ML:** ofrece una vista central de la postura de seguridad del entorno de ML. Por ejemplo, reúne las alertas de accesos no autorizados a los datos de entrenamiento (vía GuardDuty), las desviaciones de configuración de los recursos de SageMaker AI y la protección de los endpoints de inferencia mediante los hallazgos de GuardDuty e Inspector. También permite automatizar las respuestas.

### 3.2 Servicios de seguimiento y optimización de costos

AWS ofrece varios servicios de costos especialmente útiles para ML:

- **AWS Cost Explorer:** muestra los patrones de gasto y ayuda a asignar los costos.
- **AWS Cost and Usage Reports (CUR):** da visibilidad completa y detallada del uso y los costos para decidir la asignación de recursos y el presupuesto.
- **AWS Budgets:** permite fijar presupuestos personalizados y recibir alertas al superar umbrales.

En SageMaker AI, las principales palancas de ahorro son dos. Las **instancias Spot** sirven para tareas no críticas y tolerantes a interrupciones; en SageMaker AI se usan sobre todo en el entrenamiento, mediante Managed Spot Training. Los **Savings Plans** sirven para las cargas predecibles. Junto con una buena estrategia de etiquetado y la aplicación automatizada de políticas, permiten seguir, gestionar y optimizar los costos en todo el ciclo de vida de ML.

#### AWS Cost Explorer

Permite visualizar, entender y gestionar los costos y el uso a lo largo del tiempo. Desglosa el gasto por dimensiones como servicio, tipo de uso, periodo o etiqueta, para identificar tendencias y anomalías en SageMaker AI y otros servicios de ML. Sus **pronósticos**, basados en el historial, ayudan a presupuestar nuevas iniciativas de ML.

Con etiquetas, puedes seguir por separado los costos de preprocesamiento, entrenamiento e inferencia. Así identificas dónde optimizar, detectas sobreaprovisionamiento y encuentras oportunidades de ahorro, como Spot, Savings Plans o Reserved Instances. Se integra con **AWS Budgets** para fijar presupuestos y recibir alertas.

#### AWS Cost and Usage Reports (CUR)

Ofrece el **nivel más granular** de datos de facturación. Entrega **datos crudos** en archivos de reporte con numerosas columnas sobre el uso de recursos, la asignación de costos y los precios. Los archivos se depositan en un bucket de S3 y se actualizan al menos una vez al día. Hoy se configuran como CUR 2.0 desde AWS Data Exports.

| | Cost Explorer | CUR |
|---|---|---|
| Formato | Interfaz visual con gráficos y pronósticos | Archivos de datos crudos en S3 |
| Nivel de detalle | Agregado | Máximo (por recurso y hasta por hora) |
| Uso típico | Gestión de costos y presupuesto de alto nivel | Análisis y reportes a medida con Athena, QuickSight o herramientas de terceros |

**Casos de uso en ML:** analizar el costo de cada etapa del flujo de ML (preprocesamiento, entrenamiento, inferencia), seguir el uso y el costo de las instancias de SageMaker AI, identificar recursos subutilizados y atribuir los costos a proyectos o equipos concretos.

#### AWS Trusted Advisor

Revisa tu cuenta y da recomendaciones para aprovisionar los recursos según las buenas prácticas de AWS. Sus recomendaciones se agrupan en seis categorías:

- Optimización de costos
- Desempeño
- Seguridad
- Tolerancia a fallos
- Límites de servicio
- Excelencia operacional

Por ejemplo, identifica recursos inactivos o subutilizados que conviene reducir o eliminar. También detecta riesgos de seguridad, como buckets de S3 con permisos abiertos o una cuenta raíz sin MFA. Con el plan de soporte Basic solo ves los checks de límites de servicio y unos pocos de seguridad; el conjunto completo requiere un plan de soporte superior. Como revisa continuamente los recursos, ayuda a mantener la infraestructura de ML alineada con las buenas prácticas y a sostener un entorno seguro, eficiente y rentable.

#### AWS Budgets

Permite definir **presupuestos personalizados de costo y uso** y recibir alertas, por email o mediante SNS, cuando el gasto **real o pronosticado** supera un umbral. Así puedes intervenir a tiempo y evitar excesos. En ML, puedes crear presupuestos por proyecto, equipo o etapa del ciclo de vida filtrando por *cost allocation tags*. También puedes ver su evolución junto con Cost Explorer para tomar decisiones basadas en datos.

### 3.3 Modelos de precios

AWS ofrece varios modelos de precios para el cómputo, con distintos niveles de compromiso, ahorro y flexibilidad. En ML, elegir bien el modelo de precios es clave para controlar los costos sin sacrificar el desempeño. La tabla resume cada modelo, su tenencia (infraestructura compartida con otros clientes o dedicada a tu cuenta) y sus casos de uso (Tabla 7.1 del libro):

| Modelo | Cómo funciona | Tenencia | Casos de uso en ML |
|---|---|---|---|
| **On-Demand** | Pagas el tiempo de cómputo que usas (por hora o por segundo, según la instancia), sin compromiso | Compartida | Cargas impredecibles y análisis exploratorio; entrenamiento ad hoc y pruebas de algoritmos nuevos |
| **Reserved Instances** | Compromiso con una capacidad durante 1 o 3 años, con descuento | Compartida o dedicada | Cargas estables y predecibles; proyectos de largo plazo y entornos de producción |
| **Savings Plans** | Compromiso con un gasto por hora ($/hora) durante 1 o 3 años, a precios menores | Compartida o dedicada | Uso predecible; entrenamiento e inferencia continuos |
| **Spot Instances** | Capacidad EC2 no utilizada con grandes descuentos (hasta 90 %); AWS puede interrumpirla con 2 minutos de aviso | Compartida | Tareas no críticas y tolerantes a fallos, como el procesamiento batch y el preprocesamiento de datos |
| **Dedicated Hosts** | Pagas un servidor físico dedicado a tu uso | Dedicada | Requisitos regulatorios de aislamiento físico; cumplimiento y seguridad de cargas sensibles |

Detalles que conviene recordar:

- **Savings Plans** son una alternativa más flexible a las Reserved Instances. Cuando se escribió el libro existían tres tipos: **Compute** (EC2, Lambda y Fargate), **EC2 Instance** y **SageMaker**.
- **Spot:** ya no se «puja» por la capacidad; pagas el precio Spot vigente y puedes fijar opcionalmente un precio máximo.
- **En SageMaker AI:** las Reserved Instances, los Dedicated Hosts y los Savings Plans de tipo Compute o EC2 Instance no cubren las instancias de ML de SageMaker. Para ahorrar en ellas se usan los **SageMaker Savings Plans**, y Spot se usa sobre todo en el entrenamiento (Managed Spot Training).

### 3.4 Amazon SageMaker Savings Plans

Se aplican **solo al uso de instancias de SageMaker AI**. Comprometes un gasto constante ($/hora) durante 1 o 3 años y ahorras **hasta un 64 %** frente a On-Demand. El descuento se aplica automáticamente a todo el uso elegible:

- SageMaker Studio Notebook
- On-Demand Notebook
- Processing
- Data Wrangler
- Training
- Real-Time Inference
- Batch Transform

Su gran ventaja es la flexibilidad: conservas la tarifa aunque cambies de familia o tamaño de instancia, o de región. Por ejemplo, en cualquier momento puedes pasar de una instancia de CPU `ml.c5.xlarge` en US East (Ohio) a una `ml.inf1.2xlarge` en US West (Oregon) para inferencia y seguir pagando la tarifa del Savings Plan. Así optimizas la infraestructura de ML sin temer picos de costo inesperados.

---

## 4. Resumen

**Monitoreo de la inferencia.** Monitorear la inferencia es crucial para que los modelos sigan siendo exactos y confiables. El drift degrada el desempeño del modelo cuando cambian los patrones de los datos. Al comparar las predicciones con los resultados reales se detectan las discrepancias y se toman acciones correctivas, como reentrenar con datos actualizados.

**Calidad de datos y desempeño del modelo.** Los datos de entrada deben ser exactos, completos y consistentes para que las predicciones sean confiables. Métricas como accuracy, precision, recall y F1 permiten evaluar el desempeño y detectar desviaciones a tiempo. Los principios de diseño del monitoreo incluyen alertas automáticas ante la degradación, procesos de monitoreo transparentes e interpretables e integración de las herramientas de monitoreo con los flujos existentes.

**Monitoreo de la infraestructura.** La infraestructura que soporta las cargas de ML también debe monitorearse, para asegurar el desempeño y la eficiencia de costos. Servicios como CloudWatch y X-Ray dan visibilidad en tiempo real de su salud y desempeño: siguen el uso de recursos, detectan anomalías y permiten resolver problemas rápidamente.

**Seguimiento y optimización de costos.** Cost Explorer, CUR y Budgets muestran los patrones de gasto e identifican dónde optimizar. Las instancias Spot y los Savings Plans reducen el gasto de forma significativa sin perder desempeño. Por último, las arquitecturas frugales convierten el costo en un requisito no funcional: la eficiencia se incorpora al diseño y a la operación del sistema. Eso reduce tanto los costos como el impacto ambiental de los proyectos de ML.

---

## 5. Conceptos esenciales para el examen

- **Tipos de monitoreo de SageMaker Model Monitor.** Son cuatro:
  - **Calidad de datos:** detecta anomalías en los datos de entrada.
  - **Calidad del modelo:** sigue métricas de desempeño como accuracy y precision.
  - **Bias drift:** vigila que el modelo no se vuelva sesgado.
  - **Feature attribution drift:** observa los cambios en la importancia de las features.
- **Cuándo se hace el baseline.** El baseline de monitoreo se calcula antes de activar el monitoreo en producción y se recalcula en cada reentrenamiento. El de calidad de datos sale del dataset de entrenamiento; el de calidad del modelo, de un dataset de validación con las predicciones y las etiquetas del modelo ya entrenado. No lo confundas con un *modelo baseline*: un modelo simple que, al inicio del desarrollo, sirve de referencia para comparar modelos más complejos.
- **Servicios para la calidad de datos.** SageMaker Model Monitor usa las restricciones del baseline para detectar data drift, por ejemplo en la desviación estándar de las features o en la tasa de valores faltantes. Publica métricas en CloudWatch, cuyas alarmas avisan cuando se desvían del baseline. SageMaker Data Wrangler simplifica la preparación y el perfilado de los datos, y SageMaker Feature Store mantiene las features disponibles y consistentes.
- **Servicios para la calidad del modelo.** SageMaker Model Monitor sigue las métricas clave del modelo y avisa ante desviaciones. SageMaker Clarify detecta sesgos y explica las predicciones. CloudWatch monitorea métricas personalizadas y dispara alarmas que pueden iniciar acciones correctivas: actualizar los datos de entrenamiento o los parámetros del modelo, y reentrenarlo.
- **Pilares de la ML Well-Architected Lens en la fase de monitoreo.** Están los seis: excelencia operacional, seguridad, confiabilidad, eficiencia de desempeño, optimización de costos y sostenibilidad.
- **CloudWatch vs. EventBridge.** CloudWatch monitorea los recursos de AWS: recopila métricas y logs y ofrece alarmas accionables. EventBridge es un bus que enruta eventos de distintas fuentes hacia destinos, y es la base de las arquitecturas orientadas a eventos. En resumen, CloudWatch sirve para monitorear y EventBridge para enrutar eventos y reaccionar ante ellos.
- **Servicios para seguir y optimizar costos.**
  - **Cost Explorer:** analiza y pronostica el gasto con visualizaciones y reportes.
  - **Budgets:** fija presupuestos y envía alertas cuando el gasto real o pronosticado los supera.
  - **Trusted Advisor:** recomienda buenas prácticas para optimizar recursos, mejorar el desempeño y reducir costos.
  - **CUR:** entrega los datos de facturación más detallados.
  - **Savings Plans:** dan precios con descuento a cambio de comprometer un uso constante durante 1 o 3 años.

---

## 6. Preguntas de repaso

**1.** ¿Cuál es un impacto crítico de no atender el model drift en un entorno de producción?

- A. Mayor eficiencia computacional
- B. Predicciones exactas a largo plazo
- C. Posibles problemas de cumplimiento regulatorio y degradación de la exactitud en la toma de decisiones
- D. Mejor interpretabilidad del modelo

**2.** ¿Qué servicio de AWS ofrece monitoreo continuo para detectar anomalías en datos en tiempo real y compararlos con baselines?

- A. AWS Glue
- B. AWS Step Functions
- C. Amazon SageMaker Model Monitor
- D. AWS CodePipeline

**3.** Para que los modelos de ML mantengan un alto desempeño, ¿qué conjunto de métricas avanzadas debe seguirse y analizarse regularmente?

- A. Error absoluto medio (MAE), raíz del error cuadrático medio (RMSE) y curva ROC
- B. Latencia de red promedio, uso de CPU y tasas de E/S de disco
- C. Número total de solicitudes a la API, picos de latencia y throughput
- D. Volumen de datos de entrada (ingress) y salida (egress), y capacidad de almacenamiento

**4.** ¿Cuál es una consecuencia clave de no monitorear la calidad de los datos de entrada de los modelos de ML?

- A. Mayor escalabilidad del modelo
- B. Menor necesidad de feature engineering
- C. Introducción de sesgos e imprecisiones en las predicciones del modelo
- D. Mayor velocidad de entrenamiento

**5.** ¿Qué aspectos de la infraestructura deben priorizarse al monitorear despliegues de ML a gran escala?

- A. Balanceo de carga de aplicaciones, procesos de autenticación de usuarios y endpoints de API Gateway
- B. Latencia de red, métricas de uso de recursos y detección de anomalías del sistema
- C. Consistencia del diseño de interfaz, gestión del estado de sesión e invalidación de caché
- D. Ciclos de retroalimentación de la experiencia de usuario, tiempos de carga del front-end y renderizado gráfico

**6.** ¿Cómo contribuye Amazon CloudWatch a mantener el desempeño y la confiabilidad de la infraestructura de ML?

- A. Proporciona pipelines automatizados de despliegue de código
- B. Ofrece visibilidad en tiempo real de la salud del sistema, lo que permite detectar anomalías rápidamente y seguir el uso de recursos
- C. Gestiona el escalado y la asignación de memoria de las funciones serverless
- D. Facilita el aprovisionamiento con infraestructura como código (IaC) y el control de versiones

**7.** Para optimizar los costos de las cargas de ML, ¿qué estrategia combina análisis predictivo y mecanismos de alerta para evitar gastos inesperados?

- A. Implementar replicación entre regiones
- B. Configurar dashboards de costo y uso con AWS Cost Explorer
- C. Usar monitoreo mejorado del desempeño de las instancias
- D. Habilitar soluciones automatizadas de respaldo y recuperación

**8.** ¿Cuál es una ventaja significativa de usar AWS Savings Plans en cargas de ML con patrones predecibles y sostenidos?

- A. Capacidad ilimitada de llamadas a la API
- B. Flexibilidad para elegir tipos de almacenamiento de datos
- C. Reducciones sustanciales de costo por un uso constante a largo plazo en varios servicios de AWS
- D. Acceso a infraestructura dedicada en la nube

**9.** ¿Por qué es crítico el monitoreo en tiempo real de la inferencia en entornos de producción dinámicos?

- A. Para mantener predicciones exactas pese a las variaciones de los datos de entrada a lo largo del tiempo
- B. Para minimizar la latencia de los componentes de la interfaz de usuario
- C. Para automatizar la generación del dataset de entrenamiento
- D. Para asegurar la consistencia de los sistemas de control de versiones

**10.** ¿Qué servicio de AWS es fundamental para orquestar flujos complejos y automatizar procesos de varios pasos en un pipeline de ML?

- A. Amazon GuardDuty
- B. AWS Step Functions
- C. Amazon CloudWatch
- D. Amazon EventBridge
