# Deploy Open Liberty App from Gogs on OpenShift
Se va a hacer el despliegue de Gogs a partir de un template, que se puede consultar en el siguiente Gist.

[https://gist.githubusercontent.com/SaulVazquezRedHat/5007b8358011ad78e7d2ccc0114e9f7a/raw/gogs-template.yaml](https://gist.githubusercontent.com/SaulVazquezRedHat/5007b8358011ad78e7d2ccc0114e9f7a/raw/gogs-template.yaml)

```text-plain
https://gist.githubusercontent.com/SaulVazquezRedHat/5007b8358011ad78e7d2ccc0114e9f7a/raw/gogs-template.yamlkind: Template
apiVersion: template.openshift.io/v1
metadata:
  annotations:
    description: The Gogs git server (https://gogs.io/)
    tags: instant-app,gogs,go,golang
  name: gogs
objects:
- kind: ServiceAccount
  apiVersion: v1
  metadata:
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
- kind: Service
  apiVersion: v1
  metadata:
    annotations:
      description: Exposes the database server
    name: ${APPLICATION_NAME}-postgresql
  spec:
    ports:
    - name: postgresql
      port: 5432
      targetPort: 5432
    selector:
      name: ${APPLICATION_NAME}-postgresql
- kind: DeploymentConfig
  apiVersion: v1
  metadata:
    annotations:
      description: Defines how to deploy the database
    name: ${APPLICATION_NAME}-postgresql
    labels:
      app: ${APPLICATION_NAME}
  spec:
    replicas: 1
    selector:
      name: ${APPLICATION_NAME}-postgresql
    strategy:
      type: Recreate
    template:
      metadata:
        labels:
          name: ${APPLICATION_NAME}-postgresql
        name: ${APPLICATION_NAME}-postgresql
      spec:
        serviceAccountName: ${APPLICATION_NAME}
        containers:
        - env:
          - name: POSTGRESQL_USER
            value: ${DATABASE_USER}
          - name: POSTGRESQL_PASSWORD
            value: ${DATABASE_PASSWORD}
          - name: POSTGRESQL_DATABASE
            value: ${DATABASE_NAME}
          - name: POSTGRESQL_MAX_CONNECTIONS
            value: ${DATABASE_MAX_CONNECTIONS}
          - name: POSTGRESQL_SHARED_BUFFERS
            value: ${DATABASE_SHARED_BUFFERS}
          - name: POSTGRESQL_ADMIN_PASSWORD
            value: ${DATABASE_ADMIN_PASSWORD}
          image: ' '
          livenessProbe:
            initialDelaySeconds: 30
            tcpSocket:
              port: 5432
            timeoutSeconds: 1
          name: postgresql
          ports:
          - containerPort: 5432
          readinessProbe:
            exec:
              command:
              - /bin/sh
              - -i
              - -c
              - psql -h 127.0.0.1 -U ${POSTGRESQL_USER} -q -d ${POSTGRESQL_DATABASE} -c 'SELECT 1'
            initialDelaySeconds: 5
            timeoutSeconds: 1
          resources:
            limits:
              memory: 512Mi
          volumeMounts:
          - mountPath: /var/lib/pgsql/data
            name: gogs-postgres-data
        volumes:
        - name: gogs-postgres-data
          emptyDir: {}
    triggers:
    - imageChangeParams:
        automatic: true
        containerNames:
        - postgresql
        from:
          kind: ImageStreamTag
          name: postgresql:${DATABASE_VERSION}
          namespace: openshift
      type: ImageChange
    - type: ConfigChange
- kind: Service
  apiVersion: v1
  metadata:
    annotations:
      description: The Gogs server's http port
      service.alpha.openshift.io/dependencies: '[{"name":"${APPLICATION_NAME}-postgresql","namespace":"","kind":"Service"}]'
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
  spec:
    ports:
    - name: 3000-tcp
      port: 3000
      protocol: TCP
      targetPort: 3000
    - name: 10022-tcp
      port: 10022
      protocol: TCP
      targetPort: 10022
    selector:
      app: ${APPLICATION_NAME}
      deploymentconfig: ${APPLICATION_NAME}
    sessionAffinity: None
    type: ClusterIP
- kind: Route
  apiVersion: v1
  id: ${APPLICATION_NAME}-http
  metadata:
    annotations:
      description: Route for application's http service.
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
  spec:
    host: ${HOSTNAME}
    port:
      targetPort: 3000-tcp
    to:
      name: ${APPLICATION_NAME}
- kind: Route
  apiVersion: v1
  id: ${APPLICATION_NAME}-ssh
  metadata:
    annotations:
      description: Route for application's ssh service.
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}-ssh
  spec:
    host: secure${HOSTNAME}
    port:
      targetPort: 10022-tcp
    to:
      name: ${APPLICATION_NAME}
- kind: DeploymentConfig
  apiVersion: v1
  metadata:
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
  spec:
    replicas: 1
    selector:
      app: ${APPLICATION_NAME}
      deploymentconfig: ${APPLICATION_NAME}
    strategy:
      resources: {}
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: ${APPLICATION_NAME}
          deploymentconfig: ${APPLICATION_NAME}
      spec:
        serviceAccountName: ${APPLICATION_NAME}
        containers:
        - image: " "
          imagePullPolicy: Always
          name: ${APPLICATION_NAME}
          ports:
          - containerPort: 3000
            protocol: TCP
          - containerPort: 10022
            protocol: TCP
          resources: {}
          terminationMessagePath: /dev/termination-log
          volumeMounts:
          - name: gogs-data
            mountPath: /opt/gogs/data
          - name: gogs-config
            mountPath: /etc/gogs/conf
          readinessProbe:
              httpGet:
                path: /
                port: 3000
                scheme: HTTP
              initialDelaySeconds: 3
              timeoutSeconds: 1
              periodSeconds: 20
              successThreshold: 1
              failureThreshold: 3
          livenessProbe:
              httpGet:
                path: /
                port: 3000
                scheme: HTTP
              initialDelaySeconds: 3
              timeoutSeconds: 1
              periodSeconds: 10
              successThreshold: 1
              failureThreshold: 3
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        securityContext: {}
        terminationGracePeriodSeconds: 30
        volumes:
        - name: gogs-data
          emptyDir: {}
        - name: gogs-config
          configMap:
            name: gogs-config
            items:
              - key: app.ini
                path: app.ini
    test: false
    triggers:
    - type: ConfigChange
    - imageChangeParams:
        automatic: true
        containerNames:
        - ${APPLICATION_NAME}
        from:
          kind: ImageStreamTag
          name: ${APPLICATION_NAME}:${GOGS_VERSION}
      type: ImageChange
- kind: ImageStream
  apiVersion: v1
  metadata:
    labels:
      app: ${APPLICATION_NAME}
    name: ${APPLICATION_NAME}
  spec:
    tags:
    - name: "${GOGS_VERSION}"
      from:
        kind: DockerImage
        name: docker.io/openshiftdemos/gogs:${GOGS_VERSION}
      importPolicy: {}
      annotations:
        description: The Gogs git server docker image
        tags: gogs,go,golang
        version: "${GOGS_VERSION}"
- kind: ConfigMap
  apiVersion: v1
  metadata:
    name: gogs-config
    labels:
      app: ${APPLICATION_NAME}
  data:
    app.ini: |
      RUN_MODE = prod
      RUN_USER = gogs

      [database]
      DB_TYPE  = postgres
      HOST     = ${APPLICATION_NAME}-postgresql:5432
      NAME     = ${DATABASE_NAME}
      USER     = ${DATABASE_USER}
      PASSWD   = ${DATABASE_PASSWORD}

      [repository]
      ROOT = /opt/gogs/data/repositories

      [server]
      ROOT_URL=http://${HOSTNAME}
      SSH_DOMAIN=secure${HOSTNAME}
      START_SSH_SERVER=true
      SSH_LISTEN_PORT=10022

      [security]
      INSTALL_LOCK = ${INSTALL_LOCK}

      [service]
      ENABLE_CAPTCHA = false

      [webhook]
      SKIP_TLS_VERIFY = ${SKIP_TLS_VERIFY}
parameters:
- description: The name for the application.
  name: APPLICATION_NAME
  required: true
  value: gogs
- description: 'Custom hostname for http service route.  Leave blank for default hostname, e.g.: <application-name>-<project>.<default-domain-suffix>'
  name: HOSTNAME
  required: true
- displayName: Database Username
  from: gogs
  value: gogs
  name: DATABASE_USER
- displayName: Database Password
  from: '[a-zA-Z0-9]{8}'
  value: gogs
  name: DATABASE_PASSWORD
- displayName: Database Name
  name: DATABASE_NAME
  value: gogs
- displayName: Database Admin Password
  from: '[a-zA-Z0-9]{8}'
  generate: expression
  name: DATABASE_ADMIN_PASSWORD
- displayName: Maximum Database Connections
  name: DATABASE_MAX_CONNECTIONS
  value: "100"
- displayName: Shared Buffer Amount
  name: DATABASE_SHARED_BUFFERS
  value: 12MB
- displayName: Database version (PostgreSQL)
  name: DATABASE_VERSION
  value: "13-el9"
- name: GOGS_VERSION
  displayName: Gogs Version
  description: 'Version of the Gogs container image to be used (check the available version https://hub.docker.com/r/openshiftdemos/gogs/tags)'
  value: "0.11.34"
  required: true
- name: INSTALL_LOCK
  displayName: Installation lock
  description: 'If set to true, installation (/install) page will be disabled. Set to false if you want to run the installation wizard via web'
  value: "true"
- name: SKIP_TLS_VERIFY
  displayName: Skip TLS verification on webhooks
  description: Skip TLS verification on webhooks. Enable with caution!
  value: "false"
```

Para la construcción de este template de tomo como base el repositorio [https://github.com/OpenShiftDemos/gogs-openshift-docker](https://github.com/OpenShiftDemos/gogs-openshift-docker) en donde se puede encontrar un template desactualizado para un despliegue de Gogs con persistencia y sin persistencia.

El template que vamos a utilizar es una instancia de Gogs **sin** **persistencia**. 

Creando un proyecto
-------------------

Podemos crear un proyecto desde la perspectiva de **Administrador**, bajo **Home > Projects**, dando click sobre **Create Project**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/image.png)

Llenamos el formulario con:

Name:

**open-liberty-with-gogs**

Display name:

**Deploy Open Liberty from local Gogs**

Y click en **Create**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/1_image.png)

Y ya tenemos el proyecto creado.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/2_image.png)

Despliegue de template de Gogs
------------------------------

Desde la perspectiva de Developer, seleccionamos el proyecto **open-liberty-with-gogs**, damos click en el signo de más (**+**) que esta en la parte superior derecha.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/3_image.png)

Y pegamos el template, ya sea del script antes proporcionado o del [Gist](https://gist.githubusercontent.com/SaulVazquezRedHat/5007b8358011ad78e7d2ccc0114e9f7a).

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/4_image.png)

Y click en **Create**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/5_image.png)

Despliegue de Gogs

Desde la perspectiva de **Developer**, vamos a **+Add**, y bajo **Developer Catalog**, click en **All services**

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/6_image.png)

En el buscados escribimos _**gogs**_ para buscar el template que acabamos de crear. Y se da click sobre el template.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/7_image.png)

Click en **Instantiate Template**

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/8_image.png)

Escribimos el hostname. En mi caso lo obtengo de la barra de busqueda, y se agrega el subdominio.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/9_image.png)

Click en **Create**.

Y se van a desplegar dos aplicaciones. Una base de datos de Postgresql y el servidor de Gogs.

Confirmamos que de haya desplegado visitando el ruta de gogs, dando click sobre el icono de la aplicación gogs.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/10_image.png)

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/11_image.png)

Creando usuario en Gogs
-----------------------

Creamos un usuario dando click en **Register**

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/12_image.png)

Username:

**gogsuser**

Email:

**gogsuser@redhat.com**

Password:

**gogs**

Re-Type:

**gogs**

Y click en **Create New Account**

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/13_image.png)

Se inicia sesión con las credenciales recién creadas.

Username:

**gogsuser**

Password:

**gogs**

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/14_image.png)

Subimos repositorio local a Gogs
--------------------------------

Creamos un nuevo repositorio dando click en el signo de de más (**+**) en **My Repositories**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/15_image.png)

Llenamos el formulario con:

Owner:

**gogsuser** 

Repository Name:

**deployment-getting-started-open-liberty**

Visibility:

**Check** This repository is Private

Y se deja el resto en blanco.

Click en **Create Repository**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/16_image.png)

En mi caso tengo una copia local del repositorio [https://github.com/SaulVazquezRedHat/deployment-getting-started-open-liberty](https://github.com/SaulVazquezRedHat/deployment-getting-started-open-liberty) de forma local sin un sistema git. 

> Para borrar el sistema de seguimiento anterior hay que borrar el directorio **.git** en el directorio raíz del proyecto. 

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/17_image.png)

Y voy a subir el código.

Se inicializa git.

```text-plain
git init
```

Se pasa el código al stage y se hace un commit

```text-plain
git add .
git commit -m "first commit"
```

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/18_image.png)

Agregamos la referencia al repositorio remoto. Podemos copiar directamente el comando de la guía del repositorio.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/19_image.png)

```text-plain
git remote add origin http://gogs-server.apps.cluster-knk6q.knk6q.sandbox572.opentlc.com/gogsuser/deployment-getting-started-open-liberty.git
```

Y se hace push a Gogs.

```text-plain
git push -u origin main
```

Se ingresan las credenciales del usuario. Que en este caso son:

Username:

**gogsuser**

Password:

**gogs**

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/20_image.png)

Y ya podríamos ver reflejado el repositorio en Gogs.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/21_image.png)

Creación de secret
------------------

Se va a crear un secret para almacenar las credenciales de gogs.

Podemos a crear el secret desde la consola de OpenShift, desde la perspectiva de **Developer** creamos un  **Source secret** (nos aseguramos se estar en el proyecto correcto, en este caso sería _**open-liberty-woth-gogs**_)

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/22_image.png)

Y llenamos el formulario con:

**Secret name:**

key-for-liberty-s2i-build

**Authentication type:**

Basic Authentication

**Username:**

gogsuser

**Password or token:**

gogs

La contraseña es la contraseña del usuario.

Click en **Create**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/23_image.png)

Creando template
----------------

Primero vamos a descargar el template almacenado en el gist:

[https://gist.githubusercontent.com/SaulVazquezRedHat/4fabb95f4b7b1e0b64d03fc3ad636166/raw/template.yaml](https://gist.githubusercontent.com/SaulVazquezRedHat/4fabb95f4b7b1e0b64d03fc3ad636166/raw/template.yaml)

```text-plain
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: "open-liberty-build-template"
  annotations:
    description: "Build template for the demo getting started application"
    tags: "build"
objects:
  # ImageStream de imagen de Open Liberty de IBM
  - apiVersion: image.openshift.io/v1
    kind: ImageStream
    metadata:
      name: open-liberty
    spec:
      lookupPolicy:
        local: false
      tags:
        - name: kernel-slim-java11-openj9-ubi
          annotations: null
          from:
            kind: DockerImage
            name: 'icr.io/appcafe/open-liberty:kernel-slim-java11-openj9-ubi'
          importPolicy:
            importMode: Legacy
          referencePolicy:
            type: Source
  # ImageStream de imagen base de construcción
  - apiVersion: image.openshift.io/v1
    kind: ImageStream
    metadata:
      name: "${{APP_NAME}}"
    labels:
        build: "${{APP_NAME}}"
    spec:
      lookupPolicy:
        local: false
  # ImageStream de imagen base runtime (para despliegue)
  - apiVersion: image.openshift.io/v1
    kind: ImageStream
    metadata:
      name: "${APP_NAME}-runtime"
    labels:
        build: "${APP_NAME}-runtime"
    spec:
      lookupPolicy:
        local: false
  # BuildConfig de imagen de construcción
  - apiVersion: build.openshift.io/v1
    kind: BuildConfig
    metadata:
      name: "${{APP_NAME}}"
      labels:
        build: "${{APP_NAME}}"
    spec:
      nodeSelector: null
      output:
        to:
          kind: ImageStreamTag
          name: "${APP_NAME}:latest"
      resources: {}
      successfulBuildsHistoryLimit: 5
      failedBuildsHistoryLimit: 5
      strategy:
        type: Source
        sourceStrategy:
          from:
            kind: ImageStreamTag
            namespace: openshift
            name: 'ubi8-openjdk-11:1.12'
          env:
            - name: S2I_DELETE_SOURCE
              value: 'false'
      postCommit: {}
      source:
        git:
          uri: "${{URI_GIT_REPOSITORY}}"
          ref: "${{BRANCH_NAME}}"
          contextDir: "${{CONTEXT_DIRECTORY}}"
        sourceSecret:
          name: "${{SECRET_NAME}}"
      triggers:
        - type: GitHub
          github:
            secret: Hc670SCn-AezmLGdJsbb
        - type: Generic
          generic:
            secret: PrMDh4aG_GclOgAVUa-c
      runPolicy: Serial
  # BuildConfig de imagen runtime
  - apiVersion: build.openshift.io/v1
    kind: BuildConfig
    metadata:
      name: "${APP_NAME}-runtime"
    spec:
      nodeSelector: null
      output:
        to:
          kind: ImageStreamTag
          name: "${APP_NAME}-runtime:latest"
      resources: {}
      successfulBuildsHistoryLimit: 5
      failedBuildsHistoryLimit: 5
      strategy:
        type: Docker
        dockerStrategy:
          from:
            kind: ImageStreamTag
            name: 'open-liberty:kernel-slim-java11-openj9-ubi'
      postCommit: {}
      source:
        type: Dockerfile
        dockerfile: |-
          FROM open-liberty:kernel-slim-java11-openj9-ubi
          COPY --chown=1001:0 src/src/main/liberty/config /config
          RUN features.sh
          COPY --chown=1001:0 src/target/*.war /config/apps
          RUN configure.sh
        images:
          - from:
              kind: ImageStreamTag
              name: "${APP_NAME}:latest"
            paths:
              - sourcePath: /tmp/src
                destinationDir: .
      triggers:
        - type: ImageChange
          imageChange: {}
        - type: ImageChange
          imageChange:
            from:
              kind: ImageStreamTag
              name: "${APP_NAME}:latest"
      runPolicy: Serial
parameters:
- description: The application name
  name: APP_NAME
  displayName: Application Name
  required: true
- description: The remote git repository URI
  name: URI_GIT_REPOSITORY
  displayName: Git Repository URI
  required: true
- description: The working branch of git repository
  name: BRANCH_NAME
  displayName: Git Branch
  required: false
  value: "main"
- description: Directorio de contexto
  name: CONTEXT_DIRECTORY
  displayName: Context directory
  required: false
  value: "/"
- description: The name of the secret with SSH key
  name: SECRET_NAME
  displayName: Secret
  required: false
message: "... The Open Liberty Application is being deployed ..."
```

Vamos a la consola y damos click sobre el signo de más (**+**) que se encuentra en la parte superior derecha (asegurandonos de estar en el proyecto correcto).

Y pegamos el contenido del [Gist](https://gist.githubusercontent.com/SaulVazquezRedHat/4fabb95f4b7b1e0b64d03fc3ad636166/raw/template.yaml) sobre este editor de texto.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/24_image.png)

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/25_image.png)

Y click en **Create**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/26_image.png)

Instanciando template
---------------------

Desde la perspectiva de **Developer**, click en **Add to Project**

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/27_image.png)

Escribimos _**open-liberty-build-template**_, y click sobre **Instantiate Template**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/28_image.png)

Y llenamos los valores:

**Namespace:**

open-liberty-with-gogs

**Application Name:**

open-liberty-app-name

**Git Repository URI:**

_**<HTTP URL>**_

**Git Branch:**

main

**Context directory:**

/

**Secret:**

key-for-liberty-s2i-build

El repositorio de git lo podemos recuperar de la página de Gogs.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/38_image.png)

Click en **Create**:

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/29_image.png)

Y vemos que se han creado dos **BuildConfig**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/30_image.png)

Iniciamos la construcción del BuildConfig **open-liberty-app-name**, dando click en los tres puntos la derecha y dando click en **Start build**.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/31_image.png)

Y se va a comenzar a construir la imagen de construcción.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/32_image.png)

Cuando se termine de generar la imagen de construcción, se va  a comenzar a generar la images de despliegue.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/33_image.png)

Y después de un rato, va a terminar la construcción de ambos.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/34_image.png)

Despliegue de la aplicación
---------------------------

Desde la perspectiva de **Developer**, vamos a desplegar desde una imagen, dando click en la opción **Container images** en **+Add**

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/35_image.png)

Se llena el formulario con la siguiente información.

Check:

**Image stream tag from internal registry**

Project:

**open-liberty-with-gogs**

Image Stream:

**open-liberty-app-name-runtime**

Tag:

**latest**

Runtime icon:

**openliberty**

Application Name:

**open-liberty-app-name-runtime-app**

Name:

**open-liberty-app-name-runtime**

Target port:

**9080**

Create a route:

**Check**

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/36_image.png)

Y click en **Create**.

Y se va a desplegar la aplicación de Open Liberty

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/37_image.png)

Y podemos abrir la página de bienvenida consultando la ruta de la aplicación. Dando click en la flecha en la parte superior derecha del icono de Open Liberty.

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/39_image.png)

![](Deploy%20Open%20Liberty%20App%20from%20Gogs%20on%20OpenShift/40_image.png)