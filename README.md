# Documentación del Proyecto de Observabilidad y CI/CD

## 1. Resumen del Proyecto

Este proyecto implementa una plataforma integrada de observabilidad y CI/CD utilizando Docker y Docker Compose. El objetivo principal es crear un entorno robusto para monitorizar aplicaciones y automatizar los flujos de trabajo de desarrollo.

La solución combina las siguientes tecnologías clave:

-   **Elastic Stack (ELK)**: Se utiliza para la ingesta, almacenamiento y visualización de logs y métricas.
-   **OpenTelemetry**: Actúa como un colector de telemetría agnóstico para recibir, procesar y exportar datos de diferentes fuentes.
-   **Jenkins**: Proporciona la capacidad de automatización para la Integración Continua y Entrega Continua (CI/CD).

El entorno está diseñado para ser modular y escalable, permitiendo la monitorización tanto de los servicios de infraestructura (como el propio Jenkins) como de aplicaciones externas.

## 2. Componentes

La arquitectura se compone de los siguientes contenedores Docker, orquestados con Docker Compose.

### 2.1. Pila de Observabilidad (`docker-compose.yml`)

-   **Elasticsearch (`elasticsearch`)**
    -   **Imagen:** `docker.elastic.co/elasticsearch/elasticsearch:8.11.0`
    -   **Rol:** Motor de búsqueda y analítica. Almacena todos los logs y datos de telemetría enviados por el colector de OpenTelemetry. La seguridad está habilitada, requiriendo autenticación.
    -   **Puertos:** `9200` (API REST) y `9300` (comunicación interna del clúster).

-   **Kibana (`kibana`)**
    -   **Imagen:** `docker.elastic.co/kibana/kibana:8.11.0`
    -   **Rol:** Plataforma de visualización para los datos de Elasticsearch. Permite a los usuarios explorar logs, crear dashboards y monitorizar el estado de las aplicaciones.
    -   **Puerto:** `5601` (interfaz web).

-   **OpenTelemetry Collector (`otel-collector`)**
    -   **Imagen:** `otel/opentelemetry-collector-contrib:latest`
    -   **Rol:** Es el componente central para la recolección de telemetría. Recibe datos de múltiples fuentes, los procesa y los exporta a diferentes destinos. Su comportamiento se define en `config.yaml`.
    -   **Puertos:** `4317` (OTLP gRPC) y `4318` (OTLP HTTP).

### 2.2. Servidor de CI/CD (`docker-compose.jenkins.yml`)

-   **Jenkins (`jenkins`)**
    -   **Imagen:** `jenkins/jenkins:lts`
    -   **Rol:** Servidor de automatización para CI/CD. Se ejecuta con permisos de `root` y en modo `privileged` para poder interactuar con el daemon de Docker del host, permitiendo la construcción y gestión de imágenes Docker desde los pipelines.
    -   **Puerto:** `8080` (interfaz web).
    -   **Volumen:** Utiliza un volumen `jenkins_data` para persistir su configuración, el cual es compartido en modo lectura con el `otel-collector` para la recolección de logs.

## 3. Arquitectura y Flujo de Datos

Todos los servicios se comunican a través de una red Docker bridge personalizada llamada `otel-network`, lo que garantiza el aislamiento y una comunicación fluida entre contenedores.

El `otel-collector` está configurado con varias pipelines para manejar diferentes tipos de datos:

### Flujo 1: Logs de Jenkins a Elasticsearch

1.  **Recolección**: El receptor `filelog` en el `otel-collector` monitoriza activamente los archivos `*.log` dentro del volumen `/var/jenkins_home` (el volumen de datos de Jenkins).
2.  **Procesamiento**:
    -   Los logs capturados son procesados por el pipeline `logs/jenkins`.
    -   Se enriquecen con atributos estáticos que identifican el origen del log como `service.name: jenkins`.
    -   Se agrupan en lotes (`batch` processor) para optimizar el envío.
3.  **Exportación**:
    -   Los lotes de logs se envían directamente a Elasticsearch utilizando el exportador `elasticsearch/jenkins`.
    -   Los logs se escriben en índices diarios, siguiendo el patrón `jenkins-logs-%{+yyyy.MM.dd}`.

![](https://github.com/HumbleG0d/containers/blob/main/assets/image2.png)

### Flujo 2: Telemetría de una API Externa (Express.js) a APM

Esta configuración está preparada para recibir datos de una aplicación externa (no incluida en este docker-compose), como una API construida con Express.js e instrumentada con OpenTelemetry.

1.  **Recolección**:
    -   El `otel-collector` escucha en los puertos `4317` (gRPC) y `4318` (HTTP) a través del receptor `otlp`.
    -   La aplicación externa enviaría sus trazas, métricas y logs a uno de estos endpoints.
2.  **Procesamiento**:
    -   Existen pipelines separados para cada tipo de señal (`traces/express`, `metrics/express`, `logs/express`).
    -   Un filtro (`filter/express_only`) asegura que solo los datos con el atributo `service.name: express-metrics-api` sean procesados por estos pipelines.
    -   Los datos se enriquecen con atributos como `log_source: express-api`.
3.  **Exportación**:
    -   A diferencia de los logs de Jenkins, toda la telemetría de la API de Express se exporta al endpoint `http://apm-server:8200` usando el exportador `otlp/apm`. Esto está diseñado para integrarse con la solución APM (Application Performance Monitoring) de Elastic.

#### ERROR: POR CORREGIR
 No se logra obtener el servicio ` express-metrics-api` 

 ![](https://github.com/HumbleG0d/containers/blob/main/assets/image2.png)

## 4. Cómo Ejecutar

Para levantar el entorno, se deben seguir los siguientes pasos en orden. Es crucial iniciar primero la pila de observabilidad, ya que esta crea la red `otel-network` que el contenedor de Jenkins necesita.

**Paso 1: Iniciar la Pila de Observabilidad**

Antes de inciar la pila , tenemos que generar un token. Para ello hacemos uso del siguiente comando.

```bash
docker exec elasticsearch bin/elasticsearch-service-tokens create elastic/kibana kibana-token
```
este token lo reemplazamos en 

```bash
ELASTICSEARCH_SERVICEACCOUNTTOKEN=<TOKEN>
```

Abra una terminal en la raíz del proyecto y ejecute el siguiente comando:


```bash
docker-compose -f docker-compose.yml up -d
```


Este comando descargará las imágenes de Elasticsearch, Kibana y el OTel-Collector y las iniciará en segundo plano (`-d`).

**Paso 2: Iniciar Jenkins**

Una vez que la pila principal esté en funcionamiento, abra otra terminal y ejecute:

```bash
docker-compose -f docker-compose.jenkins.yml up -d
```

Este comando iniciará el contenedor de Jenkins y lo conectará a la red `otel-network` existente.

**Paso 3: Detener los servicios**

Para detener todos los servicios, ejecute los siguientes comandos:

```bash
docker-compose -f docker-compose.jenkins.yml down
docker-compose -f docker-compose.yml down
```

## 5. Acceso a los Servicios

Una vez que los contenedores estén en funcionamiento, se puede acceder a las interfaces web a través de los siguientes URLs:

-   **Kibana (Visualización y Exploración de Logs):**
    -   **URL:** `http://localhost:5601`
    -   **Usuario:** `elastic`
    -   **Contraseña:** `somethingsecret` (definida en `docker-compose.yml`)

-   **Jenkins (Automatización CI/CD):**
    -   **URL:** `http://localhost:8080`

## 6. Configuración

El comportamiento del sistema se controla a través de tres archivos principales:

-   **`docker-compose.yml`**: Define y configura la pila de observabilidad principal (Elasticsearch, Kibana, OTEL Collector). Aquí se especifican las imágenes, puertos, volúmenes y variables de entorno para estos servicios.

-   **`docker-compose.jenkins.yml`**: Define el servicio de Jenkins. Se mantiene en un archivo separado para permitir que la pila de observabilidad y Jenkins se inicien y gestionen de forma independiente.

-   **`config.yaml`**: Es el archivo de configuración para el OpenTelemetry Collector. Define los pipelines de datos (receptores, procesadores y exportadores), determinando qué datos se recolectan, cómo se transforman y a dónde se envían.
