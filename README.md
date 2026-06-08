# Tenure

![Version: 1.0.0](https://img.shields.io/badge/Version-1.0.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 1.0.20](https://img.shields.io/badge/AppVersion-1.0.20-informational?style=flat-square)

Open-source AI memory layer for engineering teams. Automatically inject
organizational knowledge, coding standards, and project context into every
AI session across VS Code, Claude Code, Cursor, Cline, Continue, and other
clients.

**Homepage:** <https://tenureai.dev>

## Prerequisites

- Kubernetes 1.28+
- Helm 3.12+

## Installation

Add the Tenure Helm repository:

```bash
helm repo add tenure https://charts.tenureai.dev
helm repo update
```

Install the chart with the release name `tenure`:

```bash
helm upgrade --install tenure tenure/tenure \
  --create-namespace \
  --namespace tenure
```

This deploys Tenure in **bundled** mode with a self-contained MongoDB instance,
auto-generated secrets, and a ClusterIP service.

## Configuration

### Deployment mode

The chart supports two top-level modes:

| Mode                | Description                                                                                                |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| `bundled` (default) | Deploys a StatefulSet MongoDB alongside Tenure. Ideal for development, proofs of concept, and small teams. |
| `external`          | Connects to an existing MongoDB (e.g., Atlas, DocumentDB, self-managed). Recommended for production.       |

Set the mode in your `values.yaml` or via `--set`:

```yaml
mode: bundled
```

### Database

**Bundled mode** values:

```yaml
database:
  bundled:
    enabled: true
    password: tenure
    persistence:
      enabled: true
      size: 8Gi
      storageClass: ""
```

**External mode** values (requires `mode: external`):

```yaml
database:
  external:
    uri: "mongodb+srv://..."
    tls:
      enabled: false
      caCertSecret: ""
```

When external TLS is enabled, `caCertSecret` must reference a Secret in the same
namespace containing the CA certificate.

### Secrets

Secret behavior depends on the mode:

- **Bundled mode**: If `secrets.existingSecret` is empty, the chart auto-generates
  a Secret containing `api-token`, `master.key`, and `belief.key`.  
  **Important:** Auto-generated secrets are regenerated on every `helm template`
  render if values are not explicitly provided. For a stable installation, capture
  the generated Secret after the first install and pin the values on upgrades,
  or switch to `secrets.existingSecret`.
- **External mode**: You **must** provide `secrets.existingSecret`. The referenced
  Secret must contain the keys `api-token`, `master.key`, and `belief.key`.

```yaml
secrets:
  existingSecret: ""
  apiToken: ""
  masterKey: ""
  beliefKey: ""
```

### Identity

`identity.userId` controls how the instance presents itself. It defaults to `"team"`
for all Helm-based installations.

```yaml
identity:
  userId: ""
```

### Observability

Tenure exports OpenTelemetry traces and metrics when an OTLP endpoint is provided:

```yaml
observability:
  otlpEndpoint: "https://otel-collector.monitoring.svc:4317"
```

### Ingress

To expose Tenure externally:

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: tenure.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: tenure-tls
      hosts:
        - tenure.example.com
```

### Resources

Standard Kubernetes requests and limits:

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

## Important Operational Notes

- **Replica count**: `replicaCount` is currently limited to `1`. Tenure requires
  distributed job locking, a shared WebSocket bus, and search-index initialization
  hooks before it can safely run multiple API replicas.
- **Secret lifecycle**: In bundled mode, auto-generated secrets are regenerated
  on every `helm template` render if not explicitly provided. For a stable
  installation, capture the generated Secret after the first install and pass
  them explicitly via `--set` on upgrades, or set `secrets.existingSecret`.
- **TLS for external MongoDB**: If `database.external.tls.enabled` is true,
  `database.external.tls.caCertSecret` must reference a Secret in the same
  namespace containing the CA certificate.

## Values

| Key                                               | Type   | Default                         | Description                                                                                      |
| ------------------------------------------------- | ------ | ------------------------------- | ------------------------------------------------------------------------------------------------ |
| affinity                                          | object | `{}`                            | Pod affinity rules                                                                               |
| backup.exportsEnabled                             | bool   | `true`                          | Enable the backup export API                                                                     |
| backup.importsEnabled                             | bool   | `true`                          | Enable the backup import API                                                                     |
| containerSecurityContext.allowPrivilegeEscalation | bool   | `false`                         | Disallow privilege escalation                                                                    |
| containerSecurityContext.capabilities.drop[0]     | string | `"ALL"`                         | Drop all capabilities                                                                            |
| containerSecurityContext.readOnlyRootFilesystem   | bool   | `true`                          | Mount the root filesystem as read-only                                                           |
| database.bundled.enabled                          | bool   | `true`                          | Deploy the bundled MongoDB StatefulSet                                                           |
| database.bundled.image.repository                 | string | `"mongodb/mongodb-atlas-local"` | Bundled MongoDB image repository                                                                 |
| database.bundled.image.tag                        | string | `"8"`                           | Bundled MongoDB image tag                                                                        |
| database.bundled.password                         | string | `"tenure"`                      | Root password for the bundled MongoDB                                                            |
| database.bundled.persistence.accessMode           | string | `"ReadWriteOnce"`               | PVC access mode                                                                                  |
| database.bundled.persistence.enabled              | bool   | `true`                          | Enable persistence for bundled MongoDB                                                           |
| database.bundled.persistence.size                 | string | `"8Gi"`                         | PVC size for bundled MongoDB                                                                     |
| database.bundled.persistence.storageClass         | string | `""`                            | Storage class for the PVC (default cluster class if empty)                                       |
| database.bundled.resources                        | object | `{}`                            | Resource requests and limits for bundled MongoDB                                                 |
| database.external.tls.caCertSecret                | string | `""`                            | Secret name containing the CA certificate for external MongoDB TLS                               |
| database.external.tls.enabled                     | bool   | `false`                         | Enable TLS for external MongoDB connections                                                      |
| database.external.uri                             | string | `""`                            | Connection URI for external MongoDB (required when mode is external)                             |
| disableJobs                                       | bool   | `false`                         | Disable background jobs                                                                          |
| env                                               | object | `{}`                            | Extra environment variables to inject into the Tenure container                                  |
| identity.userId                                   | string | `""`                            | Instance identity / user ID (defaults to "team")                                                 |
| image.pullPolicy                                  | string | `"IfNotPresent"`                | Image pull policy                                                                                |
| image.repository                                  | string | `"tenureai/tenure"`             | Tenure image repository                                                                          |
| image.tag                                         | string | `""`                            | Tenure image tag (defaults to Chart appVersion if empty)                                         |
| ingress.annotations                               | object | `{}`                            | Additional ingress annotations                                                                   |
| ingress.className                                 | string | `""`                            | Ingress class name                                                                               |
| ingress.enabled                                   | bool   | `false`                         | Enable ingress                                                                                   |
| ingress.hosts                                     | list   | `[]`                            | Ingress hosts configuration                                                                      |
| ingress.tls                                       | list   | `[]`                            | Ingress TLS configuration                                                                        |
| mode                                              | string | `"bundled"`                     | Deployment mode: "bundled" (self-contained MongoDB) or "external" (existing MongoDB)             |
| nodeSelector                                      | object | `{}`                            | Node selector for the Tenure pod                                                                 |
| observability.otlpEndpoint                        | string | `""`                            | OTLP collector endpoint for OpenTelemetry traces and metrics                                     |
| podSecurityContext.fsGroup                        | int    | `1000`                          | Filesystem group for the pod                                                                     |
| podSecurityContext.runAsGroup                     | int    | `1000`                          | Group ID to run the container as                                                                 |
| podSecurityContext.runAsNonRoot                   | bool   | `true`                          | Require a non-root user                                                                          |
| podSecurityContext.runAsUser                      | int    | `1000`                          | User ID to run the container as                                                                  |
| replicaCount                                      | int    | `1`                             | Number of replicas (currently limited to 1)                                                      |
| resources                                         | object | `{}`                            | Resource requests and limits for the Tenure container                                            |
| secrets.apiToken                                  | string | `""`                            | API token (only used when existingSecret is empty and mode is bundled)                           |
| secrets.beliefKey                                 | string | `""`                            | Belief key for CSFLE (only used when existingSecret is empty and mode is bundled)                |
| secrets.existingSecret                            | string | `""`                            | Name of an existing Secret containing api-token, master.key, and belief.key                      |
| secrets.masterKey                                 | string | `""`                            | Master key for the credential vault (only used when existingSecret is empty and mode is bundled) |
| service.port                                      | int    | `5757`                          | Kubernetes service port                                                                          |
| service.type                                      | string | `"ClusterIP"`                   | Kubernetes service type                                                                          |
| tolerations                                       | list   | `[]`                            | Pod tolerations                                                                                  |

---

Autogenerated from chart metadata using [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2)

## Upgrading

```bash
helm repo update
helm upgrade tenure tenure/tenure -n tenure
```

## Uninstalling

```bash
helm uninstall tenure -n tenure
```

## Maintainers

| Name       | Email               | Url                           |
| ---------- | ------------------- | ----------------------------- |
| Jeff Flynt | <jeff@tenureai.dev> | <https://github.com/tenurehq> |

## Source Code

- <https://github.com/tenurehq/tenure>
