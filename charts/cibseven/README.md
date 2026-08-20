# CIB seven Helm Chart

A Helm chart for [CIB seven](https://github.com/cibseven), the CIB fork of the Camunda 7 process
engine (Engine, Tasklist, Cockpit, Admin). This chart was originally forked from
[camunda-community-hub/camunda-helm](https://github.com/camunda-community-hub/camunda-helm) and
adapted to install the CIB seven distribution by default.

## Install

Add the Helm repository and install a released version:

```sh
$ helm repo add cibseven https://cibseven-community-hub.github.io/cibseven-community-helm-chart
$ helm repo update
$ helm install demo cibseven/cibseven
```

To install a specific version, pass `--version` (e.g. `--version 0.0.3`). Available versions are
listed in the repository [index.yaml](https://cibseven-community-hub.github.io/cibseven-community-helm-chart/index.yaml)
and on the chart's [GitHub Releases](https://github.com/cibseven-community-hub/cibseven-community-helm-chart/releases) page.

Alternatively, install directly from a checkout of this repository:

```sh
$ helm install demo charts/cibseven
```

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
tag `run4-latest`. Like the upstream Camunda image, it ships in application-server distributions
(`tomcat`, `wildfly`) and Spring Boot ones (`run` for Spring Boot 3, `run4` for Spring Boot 4).
Each distro publishes its own `*-latest` tag and versioned tags — see the
[cibseven/cibseven Docker Hub page](https://hub.docker.com/r/cibseven/cibseven/tags) for the full
list.

The `run-*` and `run4-*` distributions are Spring Boot bundles that include the webclient
(Cockpit/Tasklist/Admin) in-process. Switch to `tomcat-latest` or `wildfly-latest` if you prefer an
application-server distro. Note that `image.args` (below) only applies to the `run`/`run4`
distributions — the application-server distros have no start script.

`repository` and `tag` use [`tpl`](https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-tpl-function) function, it allows you to do templating:

```yaml
image:
  tag: "{{ .Chart.Version }}"
```

### Start script arguments (`image.args`)

For the `run-*` distribution, `image.args` is passed straight to CIB seven Run's start script.
The arguments and their default states are documented upstream in
[Start script arguments](https://docs.cibseven.org/manual/latest/user-guide/cibseven-run/#start-script-arguments):

| Argument | Description | Default state |
| --- | --- | --- |
| `--webapps` | Enables the CIB seven web apps | `enabled` |
| `--rest` | Enables the REST API | `enabled` |
| `--example` | Enables the example application | `enabled` |
| `--production` | Applies the `production.yaml` configuration file | `disabled` |
| `--oauth2` | Enables Spring Security OAuth2 integration | `disabled` |
| `--detached` | Starts CIB seven Run as a detached process | `enabled` |
| `--help` | Prints the available start script arguments | – |

`args: []` (the chart default) starts the script with no arguments, so the default states above
apply and you get the webapps, the REST API and the example app.

#### Passing one argument turns the defaults off

The "default state" column only holds while **no** arguments are passed. As soon as you pass any
argument, the script is in explicit mode and you get exactly what you list — the upstream docs
spell this out for `--detached` ("To disable it, explicitly pass a valid argument to the script"),
and the same applies to the component flags.

This is the usual trap: adding only `--oauth2` to secure the deployment silently drops the webapps
and the REST API. Nothing fails — the container starts normally, the endpoints are simply not
there. Everything you still want has to be listed alongside:

```yaml
image:
  args:
    - "./cibseven.sh"
    - "--webapps"
    - "--rest"
    - "--oauth2"
    - "--production"
    # --production disables the example app; list it again to keep it.
    - "--example"
```

#### `--production` and the demo admin user

`--production` selects `production.yml` instead of `default.yml`. Beyond hardening, it matters
whenever an external, read-only identity provider (LDAP, Keycloak, SCIM) supplies users and
groups: `default.yml` defines a demo admin user (id and password `demo`), and that property
**cannot be cancelled** from `distro.applicationYaml`. Spring Boot's YAML loader drops null leaf
values entirely rather than treating them as an override, so a blank `admin-user.id` is invisible
and `default.yml`'s value wins. `production.yml` never defines one, which sidesteps the problem
instead of fighting it.

Note that `--production` also disables the example application, per the upstream docs; list
`--example` after it if you want the demo process back.

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

### Distro configuration (Spring Boot `application.yaml` override)

Configuration for the whole CIB seven distribution, not just the webclient webapp. The
`cibseven/cibseven:run-*` image is a Spring Boot process that serves both the classic Camunda
webapps at `/camunda/` and the modern cibseven-webclient at `/webapp/` from the same JVM, and any
Spring Boot property — engine, datasource, actuator, logging, or webclient-specific keys (LDAP,
SSO, UI toggles, branding, feature flags, etc.) — can be overridden here. See the upstream
[cibseven-webclient `application.yaml`](https://github.com/cibseven/cibseven-webclient/blob/main/helm/cibseven-webclient/values.yaml)
for the webclient-specific keys.

Provide the overrides as raw YAML under `distro.applicationYaml`. When non-empty, the chart
renders it into a `ConfigMap`, mounts it at `/opt/cibseven/config/application.yaml`, and sets
`SPRING_CONFIG_ADDITIONAL_LOCATION` on the container so Spring Boot merges it on top of the
image's built-in defaults. Helm expressions inside the block are evaluated (via `tpl`) so you
can reference other values.

```yaml
distro:
  applicationYaml: |
    cibseven:
      webclient:
        theme: cib
        productNamePageTitle: "My CIB seven"
        ldap:
          url: ldap://ldap.example.com:389
          folder: dc=example,dc=com
          userNameAttribute: uid
        authentication:
          tokenValidMinutes: 60
    ui:
      supportedLanguages: [en, de]
      layout:
        showFeedbackButton: false
```

For one-off env-var overrides (Spring Boot's `SPRING_APPLICATION_JSON`, `CIBSEVEN_WEBCLIENT_*`
naming, or anything else), keep using `extraEnvs`. Env vars win over `applicationYaml`, which in
turn wins over the image's baked-in defaults.

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
