# Example: CIB seven with Keycloak

A minimal, self-contained umbrella chart that wraps the [`cibseven`](../../charts/cibseven)
chart and wires it up to Keycloak for both halves of the problem:

| Concern | Mechanism |
| --- | --- |
| **Authentication** — who is logging in | OpenID Connect against Keycloak (`cibseven.webclient.sso` for the webclient, Spring Security OAuth2 Resource Server for the classic webapps and Engine REST) |
| **Identity** — which users and groups exist | The [Keycloak Identity Provider Plugin](https://github.com/cibseven-community-hub/cibseven-keycloak) reading Keycloak's admin API, replacing the engine's own user database |

Nothing here is published to a Helm repository. Copy the directory into your own
GitOps repository, edit the values marked below and deploy it from there.

[`values.yaml`](values.yaml) contains **only** what the Keycloak integration
needs. Everything else — replica count, service, database, probes, ingress,
resources — stays at the [`cibseven`](../../charts/cibseven/values.yaml) chart's
defaults, so the Keycloak-specific parts are the whole file rather than being
buried in a copy of the chart defaults. See
[Beyond the Keycloak wiring](#beyond-the-keycloak-wiring) for the settings you
will most likely want to add on top.

The chart renders successfully with placeholder values, but it will not *work*
until you have created the Keycloak client described in step 1 — the placeholder
`https://keycloak.example.com` does not exist.

## Prerequisites

- Kubernetes and Helm 3
- A reachable Keycloak instance and a realm you can administer
- A default StorageClass — the example inherits the chart's H2 database, which
  requests a 1 Gi `ReadWriteOnce` volume (or switch to PostgreSQL, see below)
- Cluster egress to `artifacts.cibseven.org`, where the init container fetches
  the identity provider jar from (`values.yaml` documents the air-gapped
  alternatives)

## 1. Set up the Keycloak client

In your realm, create **one confidential client** that is used for two things:
the browser login flow, and the plugin's server-to-server calls to the admin
API.

- **Client ID**: `cibseven` (or anything else — it goes into
  `keycloak.clientId`)
- **Client authentication**: on (confidential client)
- **Standard flow**: enabled
- **Service accounts roles**: enabled — this is what the identity provider
  plugin uses
- **Valid redirect URIs**: the URL CIB seven is served under, plus `/*`, e.g.
  `https://cibseven.example.com/*`
- **Web origins**: the same origin, e.g. `https://cibseven.example.com`

Then grant the client's **service account** read access to the realm's users and
groups. Under *Service accounts roles → Assign role → Filter by clients*, assign
these `realm-management` roles:

| Role | Needed for |
| --- | --- |
| `view-users` | listing and resolving users (implies `query-users` and `query-groups`) |
| `view-clients` | resolving client roles, if you map them |

Finally, create the group that should hold your administrators — `cibseven-admin`
in this example — and add yourself to it. Because the plugin is a **read-only**
identity provider, this group is the only way to get an administrator into the
system; there is no local admin user to fall back on.

> Keycloak's admin console has changed layout across versions. The names above
> refer to the client detail tabs in Keycloak 20+; on older versions the same
> settings live under *Settings*, *Credentials* and *Service Account Roles*.

## 2. Create the client secret

The client secret is never stored in `values.yaml`. Copy it from the client's
*Credentials* tab and put it into a Secret in the target namespace:

```bash
kubectl create secret generic cibseven-keycloak-client --from-literal=client-secret='<paste-secret-here>'
```

The name and key are configurable via `keycloak.clientSecret` in `values.yaml`.
If you manage secrets with External Secrets, Sealed Secrets or SOPS, create the
Secret that way instead — the chart only references it by name.

## 3. Adapt the values

Every environment-specific setting is collected in one block at the top of
[`values.yaml`](values.yaml):

```yaml
cibseven:
  keycloak:
    url: https://keycloak.example.com   # public Keycloak base URL
    realm: cibseven                     # realm holding your users
    clientId: cibseven                  # client from step 1
    administratorGroup: cibseven-admin  # group path of your admins
    clientSecret:
      secretName: cibseven-keycloak-client
      secretKey: client-secret
```

These are helper values, not keys of the `cibseven` subchart. The subchart
renders `distro.applicationYaml` and `extraEnvs` through Helm's `tpl`, so the
Keycloak URL and realm are written once here and referenced from everywhere
else.

## 4. Deploy

```bash
helm dependency update .
```

```bash
helm upgrade --install cibseven . --namespace cibseven --create-namespace
```

Check what you are about to apply first with `helm template cibseven . | less`.

Without an ingress you can reach the webclient over a port-forward — register
`http://localhost:8080/*` as an additional redirect URI in Keycloak for this:

```bash
kubectl -n cibseven port-forward svc/cibseven-cibseven 8080:8080
```

The webclient is served at `/webapp/`, the classic webapps at `/camunda/`.

## How the pieces fit together

Three things have to line up, and each one fails silently on its own:

1. **`--oauth2` in `image.args`** — CIB seven Run only loads the Spring Security
   OAuth2 module when started with this flag. Without it every
   `spring.security.oauth2.*` and `camunda.bpm.oauth2.*` property is ignored and
   the webapps stay unsecured. Note that passing any component flag switches
   `run.sh` into explicit mode, so `--webapps` and `--rest` have to be listed
   too.
2. **The plugin jar in `configuration/userlib`** — fetched by the init
   container, which copies the image's own userlib content along so the bundled
   JDBC drivers survive the volume mount. Without the jar, every
   `plugin.identity.keycloak.*` property is ignored.
3. **`--production`** — avoids the demo `admin-user` baked into the image's
   `default.yml`. That property cannot be overridden from `application.yaml`
   (Spring Boot drops null leaf values rather than treating them as an
   override), and a configured `admin-user` is fatal once the read-only Keycloak
   identity provider is active.

The comments in `values.yaml` spell each of these out at the place it is
configured.

## Beyond the Keycloak wiring

These are plain `cibseven` chart settings, left at their defaults here on
purpose. Add the ones you need under the same `cibseven:` key — see
[charts/cibseven/values.yaml](../../charts/cibseven/values.yaml) for the full
list.

- **Ingress** — off by default. SSO needs CIB seven to be reachable under a
  stable URL that matches the redirect URI from step 1, so enable
  `cibseven.ingress` with your own host (or expose the service with whatever
  your cluster uses instead: Gateway API, OpenShift Route, a mesh gateway).
- **Database** — H2 by default, which is fine for a try-out but not clusterable
  (`replicaCount` stays at 1). Point `cibseven.database` at PostgreSQL for real
  use.
- **Resources** — unset by default; clusters with a `ResourceQuota` or
  `LimitRange` will require requests and limits.
- **Liveness probe** — off by default, and worth leaving off unless you have
  tuned it: the engine takes a while to boot, and an impatient liveness probe
  turns a slow start into a restart loop. The readiness probe *is* on by default
  with a 120 s initial delay.
- **Versions** — pin `cibseven.image.tag` and the dependency version in
  `Chart.yaml` rather than tracking `-latest`.
- **Webclient branding, languages, LDAP** — further
  `distro.applicationYaml` keys, documented in
  [charts/cibseven/README.md](../../charts/cibseven/README.md). Anything you add
  there merges with the SSO block in this example.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Login redirects work but the token is rejected (`invalid token`, issuer mismatch) | `keycloak.url` differs between browser and pod. Use one public URL both can reach. |
| Login succeeds but the user has no permissions anywhere | The user is not in `administratorGroup`, or the group path does not match (`useGroupPathAsCamundaGroupId` means the *path*, not the name). |
| Webclient loads, but every Engine REST call returns 401/403 | `camunda.bpm.run.auth.enabled` must stay `false` when using `--oauth2`; the image default turns it on. |
| Users and groups are empty in Admin | The plugin jar is missing (check the init container's logs) or the service account lacks `view-users`. |
| Pod stuck in `Pending` | No default StorageClass for the H2 PVC. |
| Pod crashes with a datasource error | The `userlib` mount hid the image's JDBC drivers — the init container must copy `/camunda/configuration/userlib/.` first. |

## References

- [`cibseven` chart README](../../charts/cibseven/README.md)
- [CIB seven Run user guide](https://docs.cibseven.org/manual/latest/user-guide/cibseven-run/)
- [Keycloak Identity Provider Plugin](https://github.com/cibseven-community-hub/cibseven-keycloak)
- [cibseven-webclient `application.yaml` reference](https://github.com/cibseven/cibseven-webclient/blob/main/helm/cibseven-webclient/values.yaml)
