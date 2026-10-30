# Enterprise Edition installation

Use this reference after the main workflow selects Enterprise Edition or detects an EE-only
backend.

## License gate

Kestra EE uses three settings:

| Kestra setting | Environment variable |
|---|---|
| `kestra.ee.license.id` | `KESTRA_EE_LICENSE_ID` |
| `kestra.ee.license.fingerprint` | `KESTRA_EE_LICENSE_FINGERPRINT` |
| `kestra.ee.license.key` | `KESTRA_EE_LICENSE_KEY` |

Treat all three values as credentials. The fingerprint is also used to access versioned/EE
plugins and, with the ID, to authenticate to the private Kestra registry.

Before changing a target, confirm without collecting the values:

- all three fields are present in an approved secret source;
- the license is valid for the intended environment and pinned Kestra version;
- every Kestra process or pod will receive the same license configuration; and
- operators can read the private EE image/JAR and any EE plugin artifacts.

If any answer is no or unknown, stop before pulling or starting EE components. Offer a JDBC
PostgreSQL/MySQL OSS configuration when it meets the requirement.

Never:

- put a real license in a committed `.env`, Compose file, systemd unit, Helm values file, or
  Kubernetes manifest;
- echo license values, pass them to a screenshot command, or include rendered configuration in
  evidence;
- use `docker compose config`, `helm get values --all`, `kubectl get secret -o yaml`, or an
  equivalent secret-rendering command as proof; or
- commit a generated Secret manifest, even when its data is base64-encoded.

## Docker Compose

Use a pinned EE image:

```yaml
services:
  kestra:
    image: registry.kestra.io/docker/kestra-ee:<PINNED_VERSION>
    environment:
      KESTRA_EE_LICENSE_ID: ${KESTRA_EE_LICENSE_ID:?license ID is required}
      KESTRA_EE_LICENSE_FINGERPRINT: ${KESTRA_EE_LICENSE_FINGERPRINT:?license fingerprint is required}
      KESTRA_EE_LICENSE_KEY: ${KESTRA_EE_LICENSE_KEY:?license key is required}
```

Store the variables in a protected file outside the repository, for example
`/etc/kestra/ee-license.env`, and run Compose with:

```bash
docker compose --env-file /etc/kestra/ee-license.env up -d
```

For a local-only evaluation, `.env` may be used only after the generated project adds it to
`.gitignore`; restrict it to the current user. The generated file may contain these names but
must not contain example values that look usable:

```dotenv
KESTRA_EE_LICENSE_ID=<FROM_SECRET_STORE>
KESTRA_EE_LICENSE_FINGERPRINT=<FROM_SECRET_STORE>
KESTRA_EE_LICENSE_KEY=<FROM_SECRET_STORE>
```

Authenticate to `registry.kestra.io` without putting the fingerprint directly in a command-line
argument. Do not capture the login command or its environment in evidence.

After startup, verify container health and a successful license-validation message without
dumping the container environment or resolved Compose model.

## Standalone host with systemd

Use the version-matched EE executable from `https://registry.kestra.io/exe/`. Keep ordinary,
non-secret Kestra configuration in the main application YAML and load the license through a
root-owned environment file:

```ini
# /etc/kestra/ee-license.env (mode 0640, owned by root and the Kestra service group)
KESTRA_EE_LICENSE_ID=<FROM_SECRET_STORE>
KESTRA_EE_LICENSE_FINGERPRINT=<FROM_SECRET_STORE>
KESTRA_EE_LICENSE_KEY=<FROM_SECRET_STORE>
```

Reference that file from the service unit rather than embedding values in the unit:

```ini
[Service]
User=kestra
Group=kestra
EnvironmentFile=/etc/kestra/ee-license.env
ExecStart=/usr/local/bin/kestra server standalone -c /etc/kestra/application.yaml
```

Use the deployment's secret manager to materialize or rotate the environment file. A rotation
requires restarting every Kestra process that consumes it. Verify the unit's active state and
non-sensitive license-validation log line; do not use `systemctl show` properties that reveal the
environment.

## Kubernetes with the official Helm chart

An EE deployment needs both:

- the pinned `registry.kestra.io/docker/kestra-ee` image plus a registry credential referenced by
  `imagePullSecrets`; and
- the runtime license configuration described below.

### Preferred: reference an existing Kubernetes Secret

Create `ee-license.yml` in a protected temporary location, not in the repository:

```yaml
kestra:
  ee:
    license:
      id: <FROM_SECRET_STORE>
      fingerprint: <FROM_SECRET_STORE>
      key: |
        <FROM_SECRET_STORE>
```

Create or reconcile a Secret named `kestra-ee-license` with a data key named
`ee-license.yml`. Prefer an External Secrets, Sealed Secrets, SOPS, or equivalent controller when
the cluster already uses one. With plain `kubectl`, source the file from outside the repository
and do not print the resulting manifest.

The checked-in Helm values contain only the reference:

```yaml
image:
  repository: registry.kestra.io/docker/kestra-ee
  tag: <PINNED_VERSION>

imagePullSecrets:
  - name: kestra-ee-pull-secret

configurations:
  secrets:
    - name: kestra-ee-license
      key: ee-license.yml
```

The chart mounts that Secret key as a Kestra configuration file. The license does not belong in
the checked-in values file.

### Inline `configurations.application`

The chart also accepts the license under:

```yaml
configurations:
  application:
    kestra:
      ee:
        license:
          id: <FROM_SECRET_STORE>
          fingerprint: <FROM_SECRET_STORE>
          key: <FROM_SECRET_STORE>
```

This is chart-native but renders into the chart-managed application configuration. Use it only
when a trusted deployment system injects an untracked, ephemeral values overlay and the security
policy accepts Helm storing the supplied values. Never put the plaintext block in a checked-in
`values.yaml`; use `configurations.secrets` for normal production and GitOps workflows.

After deployment, verify rollout status, pod health, and a non-sensitive license-validation log
line. Do not render the Secret, the final Helm values, or pod environment variables.

## EE-only backend and plugin check

The backend generator must annotate its result with `edition: OSS` or `edition: EE` and list the
reason. At minimum, flag Kafka, Redis, and AMQP queues and Elasticsearch/OpenSearch repositories
as EE-only choices. Before plugin provisioning or deployment:

1. Name the backend capability that requires EE.
2. Confirm the license gate above is complete.
3. Resolve required artifacts against the pinned Kestra version and the current plugin catalog;
   do not invent coordinates or use `LATEST` in production.
4. Confirm every node/pod that needs the backend has the required artifact.

For an unrecognized backend or version, consult the current official documentation and plugin
catalog. If edition or artifact availability remains uncertain, stop and ask rather than produce
an installation expected to fail.

## Scope

Do not add `workerGroups` or other EE worker-topology configuration here. License delivery and
install-time EE backend/plugin gating are the only EE topology concerns in this workflow.

## Authoritative references

- [Enterprise license and first tenant](https://kestra.io/docs/enterprise/overview/setup)
- [Enterprise and advanced configuration](https://kestra.io/docs/configuration/enterprise-and-advanced)
- [Standalone EE installation](https://kestra.io/docs/enterprise/overview/standalone-server-installation)
- [Kubernetes installation](https://kestra.io/docs/installation/kubernetes)
- [Official Helm chart values](https://github.com/kestra-io/kestra/blob/develop/charts/kestra/values.yaml)
