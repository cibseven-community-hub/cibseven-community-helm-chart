# Example: CIB seven with SCIM 2.0

A minimal umbrella chart that wraps the [`cibseven`](../../charts/cibseven)
chart and splits identity handling in two:

| Concern | Mechanism |
| --- | --- |
| **Authentication** — who is logging in | OpenID Connect (`cibseven.webclient.sso` for the webclient, Spring Security OAuth2 Resource Server for the classic webapps and Engine REST) |
| **Identity** — which users and groups exist | The **SCIM Identity Provider Plugin** built into CIB seven Run, reading a SCIM 2.0 directory (`camunda.bpm.run.scim`) |

Unlike the [Keycloak identity provider plugin](../cibseven-keycloak-setup), the
SCIM plugin is compiled into CIB seven Run's core module — it ships with engine
**2.2.0** and later, so there is no jar to fetch and no init container. It is
activated purely through `camunda.bpm.run.scim.*` properties.

Nothing here is published to a Helm repository. Copy the directory into your own
GitOps repository, adapt [`values.yaml`](values.yaml) and deploy it from there.

[`values.yaml`](values.yaml) contains **only** what OIDC and SCIM need.
Everything else — replica count, service, database, probes, ingress, resources —
stays at the [`cibseven`](../../charts/cibseven/values.yaml) chart's defaults.
See [Beyond the OIDC and SCIM wiring](#beyond-the-oidc-and-scim-wiring) for the
settings you will most likely want to add on top.

## What this was tested against: scimgateway

**The setup was verified end to end against
[scimgateway](https://github.com/jelhub/scimgateway).** The plugin speaks plain
SCIM 2.0 and its own README names Azure AD, Okta, OneLogin, Google Workspace and
Ping Identity as target providers, but scimgateway is the implementation this
example is known to work with, and it is what the bundled
[`charts/scim-server`](charts/scim-server) test fixture deploys.

That choice is not arbitrary. For group-membership resolution the plugin sends a
SCIM bracket *valuePath* filter:

```
GET /Groups?filter=members[value eq "<user-id>"]
```

Two other SCIM servers were tried first and both fail on exactly that query:

| Server | Behaviour on `members[value eq "…"]` |
| --- | --- |
| **scimgateway** | Implements it correctly — group membership resolves |
| Apache SCIMple | Silently ignores the filter and returns 0 results |
| WSO2 Identity Server | Rejects it outright: HTTP 400, *"Not a valid attribute name/uri"* |

**This is a known limitation and work on it is underway upstream**, so the table
above is a snapshot of one engine version rather than a permanent verdict —
broader provider coverage is expected to land in a later release. Check the
[plugin README](https://github.com/cibseven/cibseven/blob/main/engine-plugins/identity-scim/README.md)
and the [release notes](https://github.com/cibseven/cibseven/releases) for the
current state before concluding that a given provider cannot work, and try a
newer engine image first.

Until then, if you point this at a different SCIM 2.0 service and users
authenticate but end up with no groups, test that filter against your endpoint
directly — bracket filter support is the most likely incompatibility by far.

## Prerequisites

- Kubernetes and Helm 3
- An OpenID Connect provider and a client you can configure
- A SCIM 2.0 endpoint with credentials — this is the one thing the example
  cannot supply for you. For a local try-out without a real directory, the
  bundled test server can stand in, at the cost of building its image
  (see [step 3](#3-optional-local-try-out-with-the-bundled-test-server))
- A default StorageClass — the example inherits the chart's H2 database, which
  requests a 1 Gi `ReadWriteOnce` volume (or switch to PostgreSQL, see below)
- An engine image from `run-2.2.0` onwards; earlier tags have no SCIM plugin

## 1. Set up the OIDC client

In your identity provider, create a **confidential client** for CIB seven:

- **Client ID**: `cibseven` (or anything else — it goes into `oidc.clientId`)
- **Client authentication**: on, standard authorization-code flow enabled
- **Valid redirect URIs**: the URL CIB seven is served under, plus `/*`, e.g.
  `https://cibseven.example.com/*`
- **Web origins**: the same origin

Note the issuer URL. For Keycloak that is
`https://<host>/realms/<realm>`; anything else, take `issuer` from
`<issuerUrl>/.well-known/openid-configuration`.

> The endpoint URLs in `values.yaml` are built from `oidc.issuerUrl` using
> Keycloak's OIDC path layout (`/protocol/openid-connect/…`). For a different
> provider, replace those four lines with the URLs from its discovery document —
> the paths are provider-specific, the issuer alone is not enough to derive them.

The user account you intend to log in with has to exist in the **SCIM directory**
too, under the `userName` the OIDC token's `preferred_username` claim carries.
Authentication and identity are separate here: OIDC proves who you are, SCIM
decides whether that user exists and which groups they belong to.

## 2. Create the secrets

Neither secret is stored in `values.yaml`:

```bash
kubectl create secret generic cibseven-oidc-client --from-literal=client-secret='<oidc-client-secret>'
```

```bash
kubectl create secret generic cibseven-scim-credentials --from-literal=credential='<scim-password-or-token>'
```

The second one holds the bearer token when `scim.authenticationType: bearer`
(the default), or the Basic auth password when it is `basic`.

Names and keys are configurable via `oidc.clientSecret` and `scim.credential`.
If you manage secrets with External Secrets, Sealed Secrets or SOPS, create them
that way instead — the chart only references them by name.

## 3. Optional: local try-out with the bundled test server

**Skip this section entirely if you have a SCIM endpoint** — it is off by default
and the example does not depend on it.

[`charts/scim-server`](charts/scim-server) deploys a throwaway scimgateway with
two seeded users (`camunda-admin-user`, `camunda-test-user`) and two seeded
groups (`camunda-admin`, `camunda-users`), so the plugin has something to talk to
before you involve a real directory. It is vendored into `charts/` rather than
pulled from a repository, and wired up with a `scim-server.enabled` condition in
`Chart.yaml`.

It is **disabled by default** because scimgateway publishes no container image
(the npm package ships TypeScript entrypoints needing Bun or Node ≥ 22.6, so
there is no ready-made way to run it either). You have to build one:

```bash
git clone https://github.com/jelhub/scimgateway && cd scimgateway && cp config/docker/Dockerfile . && docker build -t scimgateway:local .
```

Then make it reachable from your cluster — `kind load docker-image
scimgateway:local`, `minikube image load scimgateway:local`, or push it to a
registry your nodes can pull from and set `scim-server.image.repository`
accordingly with `pullPolicy: IfNotPresent`.

With the image in place, switch the plugin over to it:

```yaml
cibseven:
  scim:
    serverUrl: http://scim-server:8880
    authenticationType: basic
    username: gwadmin
    adminGroup: camunda-admin
scim-server:
  enabled: true
```

and make `cibseven-scim-credentials` hold scimgateway's default password,
`password`. Log in as `camunda-admin-user` — that user is in the seeded
`camunda-admin` group, so it gets administrator rights.

The server is in-memory, unauthenticated over plain HTTP and seeded with fixed
fixtures. It exists to exercise the plugin, nothing more; delete the directory if
you have no use for it.

## 4. Deploy

```bash
helm dependency update .
```

```bash
helm upgrade --install cibseven . --namespace cibseven --create-namespace
```

Check what you are about to apply first with `helm template cibseven . | less`.

Without an ingress you can reach the webclient over a port-forward — register
`http://localhost:8080/*` as an additional redirect URI in your OIDC client for
this:

```bash
kubectl -n cibseven port-forward svc/cibseven-cibseven 8080:8080
```

The webclient is served at `/webapp/`, the classic webapps at `/camunda/`.

## How the pieces fit together

Three things have to line up, and each one fails silently on its own:

1. **`--oauth2` in `image.args`** — CIB seven Run only loads the Spring Security
   OAuth2 module when started with this flag. Without it every
   `spring.security.oauth2.*` and `camunda.bpm.oauth2.*` property is ignored and
   the webapps stay unsecured. Passing any component flag switches `run.sh` into
   explicit mode, so `--webapps` and `--rest` have to be listed too.
2. **`camunda.bpm.run.auth.enabled: false`** — CIB seven Run's own
   pseudo/basic/composite Engine REST filter must be off when `--oauth2` is in
   play; running both makes Engine REST reject the webclient's requests. The
   image's `default.yml` turns it on, so it has to be overridden explicitly.
3. **`--production`** — avoids the demo `admin-user` baked into the image's
   `default.yml`. That property cannot be overridden from `application.yaml`
   (Spring Boot drops null leaf values rather than treating them as an
   override), and a configured `admin-user` is fatal once a read-only external
   identity provider is active.

Administrator rights come from the separate `camunda.bpm.run.admin-auth` plugin,
not from the SCIM plugin — unlike the Keycloak identity provider plugin, the
SCIM one has no admin-group option of its own. `scim.adminGroup` must name a
group that actually exists in the directory.

## Beyond the OIDC and SCIM wiring

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
- **Versions** — `cibseven.image.tag` is pinned to `run-2.2.0`; bump it
  deliberately rather than tracking `-latest`, and pin the dependency version in
  `Chart.yaml` too.
- **Webclient branding, languages, LDAP** — further
  `distro.applicationYaml` keys, documented in
  [charts/cibseven/README.md](../../charts/cibseven/README.md). Anything you add
  there merges with the OIDC block in this example.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Login redirects work but the token is rejected (`invalid token`, issuer mismatch) | `oidc.issuerUrl` differs between browser and pod. Use one URL both can reach. |
| Login succeeds but the user does not exist | The `preferred_username` claim has no matching `userName` in the SCIM directory. |
| User exists but belongs to no groups | The SCIM endpoint may not support bracket *valuePath* filters — a known limitation being worked on upstream, so try a newer engine image; see [the scimgateway section](#what-this-was-tested-against-scimgateway). |
| Login succeeds but the user has no permissions anywhere | The user is not in `scim.adminGroup`, or that group is not in the directory. |
| Webclient loads, but every Engine REST call returns 401/403 | `camunda.bpm.run.auth.enabled` must stay `false` when using `--oauth2`; the image default turns it on. |
| Every SCIM call fails with 401 | The `cibseven-scim-credentials` Secret does not match the endpoint, or `authenticationType` is wrong (the plugin defaults to `bearer`, the test server needs `basic`). |
| No user or group data at all, no SCIM traffic | Image older than `run-2.2.0` — the plugin isn't in it. |
| Pod stuck in `Pending` | No default StorageClass for the H2 PVC. |
| `scim-server` pod stuck in `ErrImageNeverPull` | You enabled the bundled test server without building and side-loading its image — see step 3. |

Set `verbose: true` under `camunda.bpm.run.scim` (commented out in
`values.yaml`) to log every SCIM request and response while debugging.

## References

- [`cibseven` chart README](../../charts/cibseven/README.md)
- [Keycloak variant of this example](../cibseven-keycloak-setup)
- [SCIM plugin README](https://github.com/cibseven/cibseven/blob/main/engine-plugins/identity-scim/README.md)
- [CIB seven 2.2.0 release notes](https://github.com/cibseven/cibseven/releases/tag/v2.2.0)
- [scimgateway](https://github.com/jelhub/scimgateway)
- [CIB seven Run user guide](https://docs.cibseven.org/manual/latest/user-guide/cibseven-run/)
