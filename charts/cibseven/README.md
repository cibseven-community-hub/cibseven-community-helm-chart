# CIB seven Helm Chart

A Helm chart for [CIB seven](https://github.com/cibseven), the CIB fork of the Camunda 7 process
engine (Engine, Tasklist, Cockpit, Admin). This chart was originally forked from
[camunda-community-hub/camunda-helm](https://github.com/camunda-community-hub/camunda-helm) and
adapted to install the CIB seven distribution by default.

## Install

```sh
$ helm install demo charts/cibseven
```

Once a released chart is published to the Helm registry, install it from there instead.

## Links

* CIB seven org: <https://github.com/cibseven>
* CIB seven engine (Docker): <https://hub.docker.com/r/cibseven/cibseven>
* CIB seven engine (Docker sources): <https://github.com/cibseven/cibseven-docker>
* CIB seven webclient: <https://github.com/cibseven/cibseven-webclient>

## Example

Using this custom values file the chart will:

* Use a custom name for deployment.
* Use PostgreSQL as an external database (it assumes that the database `process-engine` is already created
  and the secret `cibseven-postgresql-credentials` has the mandatory data `DB_USERNAME` and `DB_PASSWORD`).
* Set custom config for `readinessProbe` and checking an endpoint that queries the database
  so no traffic will be sent to the REST API if the engine pod is not able to access the database.
* Expose Prometheus metrics of the engine over the metrics service with port `9404`.

```yaml
# Custom values.yaml

general:
  fullnameOverride: cibseven-rest
  replicaCount: 3

image:
  repository: cibseven/cibseven
  tag: run-latest
  command: ['./camunda.sh']
  args: ['--rest']

extraEnvs:
- name: DB_VALIDATE_ON_BORROW
  value: "false"

database:
  driver: org.postgresql.Driver
  url: jdbc:postgresql://cibseven-postgresql:5432/process-engine
  credentialsSecretName: cibseven-postgresql-credentials
  credentialsSecretEnabled: true

service:
  type: ClusterIP
  port: 8080
  portName: http

readinessProbe:
  enabled: true
  config:
    httpGet:
      path: /engine-rest/incident/count
      port: http
    initialDelaySeconds: 120
    periodSeconds: 60

metrics:
  enabled: true
  service:
    type: ClusterIP
    port: 9404
    portName: metrics
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/path: "/"
      prometheus.io/port: "9404"
```

## Configuration

### General

#### Replicas

Set the number of replicas:

```yaml
general:
  replicaCount: 1
```

**Please note**, cluster mode is not supported with the default H2 database — an external database
must be used if you want to run more than one replica.

#### Extra environment variables

The deployment could be customized by providing extra environment variables according to the
[cibseven-docker](https://github.com/cibseven/cibseven-docker) image. The extra environment variables
will be templated using the ['tpl' function](https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-tpl-function).
This is useful to pass a template string as a value to a chart or render external configuration files.

```yaml
extraEnvs:
- name: DB_VALIDATE_ON_BORROW
  value: "false"
- name: SERVICE_PORT
  value: {{ .Values.service.port }}
```

#### Debugging

Enable debugging in the CIB seven container by setting:

```yaml
general:
  debug: true
```

### Init Containers

For a reason or another, you could need to do some pre-startup actions before the engine starts.
e.g. you could wait for a specific service to be ready or to post to an external service.

If that's needed, it could be done as the following:

```yaml
initContainers:
- name: pre-startup-checks
  image: busybox:1.28
  command: ['sh', '-c', 'echo "The initContainers work as expected"']
```

### Extra Containers

For a reason or another, you could need to add sidecars.
e.g. you could have a fluentd that check your logs or a vault that inject your db secrets.

If that's needed, it could be done as the following:

```yaml
extraContainers:
- name: fluentd
  image: "fluentd"
  volumeMounts:
    - mountPath: /my_mounts/cribl-config
      name: config-storage
```

### Image

By default the chart uses the CIB seven fork of the Camunda 7 engine — image `cibseven/cibseven`,
tag `run-latest`. Like the upstream Camunda image, it ships in three distributions: `tomcat`,
`wildfly`, and `run`. Each distro publishes its own `*-latest` tag and versioned tags — see the
[cibseven/cibseven Docker Hub page](https://hub.docker.com/r/cibseven/cibseven/tags) for the full
list.

The `run-*` distribution is a Spring Boot bundle that includes the webclient (Cockpit/Tasklist/Admin)
in-process. Switch to `tomcat-latest` or `wildfly-latest` if you prefer an application-server distro.

`repository` and `tag` use [`tpl`](https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-tpl-function) function, it allows you to do templating:

```yaml
image:
  tag: "{{ .Chart.Version }}"
```

### Database

One of the [supported databases](https://docs.camunda.org/manual/latest/introduction/supported-environments/#databases)
could be used with CIB seven.

The H2 database is used by default which works fine if you just want to test CIB seven.
But since the database is embedded, only 1 deployment replica could be used.

For real-world workloads, an external database like PostgreSQL should be used.
The following is an example of using PostgreSQL as an external database.

First, assuming that you have a PostgreSQL system up and running with service and port
`cibseven-postgresql:5432`, also the database `process-engine` is created and you have its credentials,
create a secret has database credentials which will be used later by CIB seven deployment:

```sh
$ kubectl create secret generic         \
    cibseven-postgresql-credentials     \
    --from-literal=DB_USERNAME=foo      \
    --from-literal=DB_PASSWORD=bar
```

Now, set the values to use the external database:

```yaml
database:
  driver: org.postgresql.Driver
  url: jdbc:postgresql://cibseven-postgresql:5432/process-engine
  credentialsSecretName: cibseven-postgresql-credentials
  credentialsSecretEnabled: true
  # The username and password keys could be customized to whatever used in the credentials secret.
  credentialsSecretKeys:
    username: DB_USERNAME
    password: DB_PASSWORD
```

**Please note**, this Helm chart doesn't manage any external database, it just uses what's configured.

### Metrics

Enable Prometheus metrics for the engine by setting the following in the values file:

```yaml
metrics:
  enabled: true
  service:
    type: ClusterIP
    port: 9404
    portName: metrics
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/path: "/"
      prometheus.io/port: "9404"
```

If there is a Prometheus configured in the cluster, it will scrap the metrics service automatically.
Check the notes after the deployment for more details about the metrics endpoint.
