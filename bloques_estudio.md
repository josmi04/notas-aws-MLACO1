---
title: "Plan de notas para AWS MLA-C01: 30 bloques"
certificacion: AWS Certified Machine Learning Engineer – Associate
version_examen: MLA-C01
fecha: 2026-09-20
numero_bloques: 30
extension_objetivo_por_nota: "Aproximadamente 15 páginas, incluidas preguntas y soluciones"
perfil: "Conocimientos avanzados de estadística y Machine Learning; fundamentos teóricos de AWS"
tags: [aws, mla-c01, machine-learning, plan-de-estudio]
---

# Plan de notas para AWS MLA-C01

## Alcance y nivel

Este plan organiza el estudio en **30 bloques**, cada uno destinado a una nota independiente de **unas 15 páginas**. La extensión orientativa es de **11–12 páginas de desarrollo y ejemplos, más 3–4 páginas de preguntas y soluciones**. En Markdown la paginación depende de la exportación: estas cifras expresan profundidad aproximada, no una obligación de rellenar. El conjunto equivale a unas **450 páginas**.

Se dan por dominados los fundamentos avanzados de estadística y Machine Learning: algoritmos, optimización, hiperparámetros, regularización, métricas, validación, ingeniería de variables y fundamentos de aprendizaje profundo. Las notas deben concentrarse en **implementar, configurar y operar ML en AWS**, explicando las particularidades de sus servicios y las decisiones relevantes para MLA-C01.

El conocimiento de AWS de partida es teórico, equivalente al nivel Cloud Practitioner. Los mecanismos operativos de AWS deben explicarse cuando se introducen; los conocimientos avanzados de ML no implican experiencia avanzada en infraestructura.

La división es una propuesta pedagógica basada en la guía **AWS-Certified-Machine-Learning-Engineer-Associate_Exam-Guide.pdf**, versión MLA-C01, aportada al proyecto. Los bloques no representan subdivisiones oficiales ni sus cantidades reproducen las ponderaciones del examen. El alcance permanece centrado en MLA-C01.

## Criterios comunes para redactar cada nota

- Utilizar **@Notas de programación** y entregar un archivo Markdown.
- Declarar los conocimientos previos necesarios y el límite temático de la nota. Los bloques anteriores solo se consideran dominados si ya fueron estudiados.
- Dar por conocida la teoría avanzada de ML; explicar su implementación y comportamiento específico en AWS.
- Definir cada concepto nuevo de AWS antes de usarlo o en su primera aparición, respetando una progresión lógica.
- Incluir ejemplos realistas con **AWS CLI y boto3** cuando corresponda. Explicar contexto, recursos, permisos, parámetros relevantes, resultados esperados y comprobaciones.
- Utilizar el SDK de SageMaker cuando resulte pertinente, explicando su papel y su relación con boto3.
- Justificar las decisiones por sus consecuencias prácticas: rendimiento, coste, seguridad, disponibilidad y complejidad operativa.
- Concentrar el desarrollo práctico en uno o dos casos relacionados; tratar otras alternativas con la profundidad necesaria para elegir entre ellas.
- Verificar configuraciones e interfaces en documentación oficial al redactar cada nota, aclarando cambios de estado de servicios que afecten a los ejemplos.
- Cerrar con **5 preguntas de dificultad media y 5 de dificultad alta**, con escenarios de estilo AWS y un solucionario que justifique las respuestas y descarte las alternativas incorrectas.
- Evitar contenido de relleno y detalles especializados que no contribuyan al alcance acordado.

## I. Fundamentos operativos de AWS — 3 bloques

### 01. Operación programática con AWS CLI y boto3

Autenticación temporal, perfiles y regiones; sesiones y clientes; identificación de la cuenta y del principal que ejecuta las operaciones; solicitudes, respuestas y errores; paginación, espera de operaciones asíncronas y reintentos. Caso práctico reproducible que cree recursos, consulte su estado y recupere resultados.

### 02. IAM aplicado a trabajos y aplicaciones de ML

Políticas de identidad y de recursos; evaluación de permisos; roles y políticas de confianza; STS; acceso entre cuentas; diferencia entre asumir y pasar un rol; roles de ejecución de SageMaker. Construcción de permisos concretos y diagnóstico de denegaciones.

### 03. Conectividad y aislamiento de cargas de ML

VPC, subredes, rutas y grupos de seguridad; acceso público y privado; salida a internet mediante NAT; endpoints privados hacia servicios de AWS. Conectividad de entrenamiento e inferencia, acceso a datos y dependencias, y diagnóstico de fallos de red.

## II. Almacenamiento, ingesta y preparación — 11 bloques

### 04. Formatos, organización y consulta de datos para ML

CSV, JSON, Parquet, ORC, Avro y RecordIO según patrones de consumo; esquemas, compresión, particiones y tamaños de archivo. Lagos de datos y almacenes analíticos; Glue Data Catalog y Athena; papel de Redshift y Lake Formation. Caso de organización y consulta de datos en S3.

### 05. S3: operaciones y rendimiento de acceso a datos

Objetos, claves, prefijos, metadatos y etiquetas desde su uso programático; carga, descarga, listado y paginación; consistencia y sobrescrituras; cargas divididas en partes, concurrencia y reintentos; distribución de solicitudes y aceleración de transferencias. Diagnóstico de un flujo de datos lento.

### 06. S3 y KMS: autorización y cifrado de datos de ML

Políticas del bucket, acceso entre cuentas, bloqueo público y puntos de acceso; papel de las ACL heredadas. SSE-S3, SSE-KMS y DSSE-KMS; políticas y permisos sobre claves. Configuración y diagnóstico de acceso a conjuntos de datos cifrados.

### 07. S3: recuperación, replicación y conservación de datos

Versionado, marcadores de borrado y recuperación; replicación regional e interregional, roles y tratamiento de objetos existentes. Clases de almacenamiento, restauración, costes y reglas de ciclo de vida. Dos casos relacionados: recuperación de errores y conservación económica de versiones históricas.

### 08. EBS, EFS y FSx: almacenamiento conectado al cómputo

Bloques frente a archivos; persistencia y relación con instancias y zonas; rendimiento, snapshots y modificación de EBS; montaje y acceso compartido con EFS; variantes relevantes de FSx. Selección y configuración del almacenamiento para procesamiento y entrenamiento.

### 09. Extracción y movimiento de datos hacia una plataforma de ML

Lectura y exportación desde RDS y DynamoDB; impacto sobre el origen, capacidad y consistencia; extracción completa e incremental; transferencia desde infraestructura existente con DataSync y papel de Storage Gateway. Diseño de una ingesta por lotes verificable.

### 10. Kinesis Data Streams y Data Firehose: ingesta y entrega continua

Productores, registros, particiones, consumidores, retención y capacidad; distribución de claves y saturación. Entrega con Firehose, agrupación de registros, transformación, formatos, destinos y fallos. Implementación y comparación de ambos servicios sobre un mismo caso de eventos.

### 11. Procesamiento continuo con Flink e integración con Kafka

Tiempo del evento, ventanas, estado, eventos tardíos y recuperación en Flink. Fundamentos operativos necesarios de Kafka y MSK: temas, particiones, posiciones y grupos de consumidores. Caso central de procesamiento con Flink; comparación de fuentes Kafka y Kinesis, sin convertirlo en un manual de administración de Kafka.

### 12. AWS Glue y EMR: integración y transformación distribuida

Trabajos de Glue, lectura y combinación de fuentes, transformaciones con Spark, procesamiento incremental y escritura organizada en S3; seguimiento y diagnóstico de trabajos. Criterios de elección entre Glue y EMR, con profundidad operativa concentrada en Glue.

### 13. Preparación reproducible y calidad de datos en AWS

SageMaker Processing para ejecutar código propio; entradas, salidas, dependencias y registros. Data Wrangler y DataBrew como alternativas visuales; controles con Glue Data Quality; ejecución de transformaciones sin contaminación entre particiones. Implementación completa mediante código y comparación de alternativas.

### 14. Feature Store, etiquetado y preparación de datos confiables

Desarrollo principal de SageMaker Feature Store: grupos de variables, ingesta, consulta, almacenamiento inmediato e histórico y coherencia temporal. Integración de etiquetas mediante Ground Truth; control de calidad del etiquetado; aplicación de controles de sesgo y protección de información sensible en el conjunto resultante.

## III. Desarrollo y entrenamiento — 6 bloques

### 15. Elegir entre desarrollo propio, algoritmos de SageMaker y servicios de IA

Selección según formato de entrada, personalización, requisitos operativos, latencia y coste. Particularidades de algoritmos incorporados de SageMaker; servicios administrados para documentos, lenguaje, voz e imágenes. Casos de integración e invocación; se da por conocida la teoría de los algoritmos.

### 16. SageMaker JumpStart y Amazon Bedrock para modelos preentrenados

Selección, acceso, invocación y adaptación de modelos; preparación de datos para ajuste; despliegue o consumo administrado; evaluación, permisos y costes. Comparación entre ambos caminos, limitada a las competencias de MLA-C01.

### 17. SageMaker Training: ejecutar algoritmos y código propio

Anatomía del trabajo: cómputo, rol, imagen de ejecución, datos, hiperparámetros y artefactos. Modalidades de entrada; adaptación de scripts mediante script mode; dependencias, métricas y guardado del modelo. Recorrido completo con boto3 y uso explicado del SDK de SageMaker cuando corresponda.

### 18. Ajuste, evaluación y diagnóstico de modelos en SageMaker

Configuración del ajuste automático; definición y extracción de métricas; presupuesto y paralelismo; evaluación reproducible. SageMaker Debugger para diagnosticar entrenamiento y Clarify para explicabilidad y sesgo. Interpretación operativa de resultados, dando por dominada su base estadística.

### 19. Optimización del entrenamiento: cómputo, datos y recuperación

Elección de CPU y GPU; cuellos de botella de lectura y cómputo; distribución del entrenamiento al nivel necesario para configurar y elegir opciones; checkpoints; entrenamiento con Spot y recuperación. Comparación de configuraciones por tiempo, coste y uso de recursos.

### 20. Experimentos, trazabilidad y registro de modelos

Registro de configuraciones, métricas y artefactos; comparación y reproducción de ejecuciones; relación entre datos, código y modelo; Model Registry, versiones y aprobaciones. Promoción de un candidato evaluado hacia despliegue y recuperación de versiones anteriores.

## IV. Despliegue y automatización — 7 bloques

### 21. Inferencia en tiempo real con SageMaker

Modelo para alojamiento, configuración y endpoint; entorno de inferencia; solicitudes y respuestas; creación, seguimiento e invocación. Selección inicial de recursos, permisos, registros y diagnóstico de errores. Desplegar un modelo previamente entrenado.

### 22. Inferencia por lotes, asíncrona y sin servidores

Batch Transform; organización de entradas y resultados; inferencia asíncrona y solicitudes pendientes; capacidad y concurrencia de inferencia sin servidores. Implementaciones comparables y elección según latencia, duración, tamaño de solicitudes y patrón de tráfico.

### 23. Contenedores y alternativas de alojamiento de modelos

Construcción, prueba y publicación de imágenes en ECR; contratos de entrenamiento e inferencia de SageMaker; incorporación de dependencias propias. Comparación con alojamiento en EC2, ECS, EKS y Lambda; múltiples modelos o contenedores y papel de SageMaker Neo.

### 24. Capacidad, escalamiento y actualización segura de endpoints

Métricas y políticas de escalamiento; capacidad mínima y máxima, cuotas y saturación. Variantes, reparto de tráfico, pruebas A/B y en sombra; actualizaciones graduales, alarmas y reversión. Resolver crecimiento de demanda y sustitución de modelos en producción.

### 25. Infraestructura de ML reproducible con CloudFormation y CDK

Recursos, parámetros, dependencias y salidas; creación y actualización de infraestructura; comunicación entre conjuntos de recursos; fallos y reversión. Implementación de un entorno acotado de ML y comparación del uso de plantillas y CDK.

### 26. SageMaker Pipelines y coordinación de trabajos de ML

Pasos de preparación, entrenamiento, evaluación y registro; parámetros, dependencias, condiciones y reutilización de resultados. Comparación con Step Functions y Airflow administrado mediante MWAA. Implementación principal en SageMaker Pipelines y diagnóstico de ejecuciones.

### 27. Automatización por eventos e integración y entrega continuas

EventBridge, SQS y SNS dentro de un flujo concreto; duplicados y reintentos. Git, CodeBuild, CodePipeline y papel de CodeDeploy; pruebas, construcción de artefactos, aprobaciones y despliegue. Automatizar cambios de código y activaciones de reentrenamiento.

## V. Operación, costes y seguridad — 3 bloques

### 28. Monitoreo de datos, predicciones, sesgo y explicaciones

Captura de datos y referencias iniciales con Model Monitor; programación de controles; incorporación de resultados reales; monitoreo de sesgo y atribuciones con Clarify. Alertas y decisiones de investigación, reentrenamiento o reversión. Se da por conocida la teoría del deterioro predictivo.

### 29. Observabilidad y optimización de infraestructura y costes

CloudWatch: métricas, registros, consultas, paneles y alarmas; diagnóstico de latencia, errores y capacidad; papel de X-Ray. Dimensionamiento mediante herramientas de recomendación; Cost Explorer, Budgets, etiquetas y modalidades de compra. Caso de diagnóstico con consecuencias de rendimiento y coste.

### 30. Seguridad y auditoría de un sistema de ML completo

Integración de permisos, cifrado, conectividad privada, aislamiento y secretos en preparación, entrenamiento y despliegue. Protección del proceso de entrega; CloudTrail y AWS Config; investigación de accesos y cambios. Caso completo de revisión y corrección de configuraciones.

## Instrucción de profundidad para las consultas

> Da por dominados los fundamentos avanzados de estadística y Machine Learning. No desarrolles definiciones introductorias de algoritmos, hiperparámetros, métricas, regularización o validación. Concentra la explicación en su implementación y comportamiento en AWS: recursos, configuraciones, interfaces, permisos, limitaciones, decisiones y diagnóstico. Apunta a unas 15 páginas equivalentes, incluidas las diez preguntas y sus soluciones razonadas. Distribuye la extensión según la dificultad real y evita rellenar con teoría conocida.

## Plantilla breve para solicitar una nota

> Usa @Notas de programación para desarrollar el bloque **[número y título]** de este plan, respetando todo su alcance y los criterios comunes. Ya he estudiado **[bloques o notas concretas]**. Da por dominados los fundamentos avanzados de ML, pero explica los mecanismos nuevos de AWS. Incluye ejemplos realistas con AWS CLI y boto3 cuando corresponda, y las diez preguntas con soluciones razonadas. La extensión objetivo es de unas 15 páginas equivalentes. Entrega un archivo `.md`.
