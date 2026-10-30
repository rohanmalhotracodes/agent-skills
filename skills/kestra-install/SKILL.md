---
name: kestra-install
description: Provision and configure a self-hosted Kestra server on a standalone host, Docker Compose, or Kubernetes. Use for new installations and backend selection, including safe Enterprise Edition license setup and EE-only backend checks; hand an existing server to kestra-ops.
---

# Kestra Install

Plan and provision a new self-hosted Kestra server. Keep generated configuration version-pinned,
edition-aware, and free of plaintext credentials.

## Required inputs

- Target: standalone host, Docker Compose, or Kubernetes with the official Helm chart
- Intended Kestra version and edition: OSS or Enterprise Edition (EE)
- Environment: evaluation or production
- Repository, queue, and internal-storage backends
- Existing secret-delivery mechanism and the operator who can approve deployment changes

Do not ask the user to paste license values into chat, generated documentation, logs, or a
checked-in file. Ask only whether all three license fields are available and where they will be
injected.

## Workflow

1. Confirm the target, pinned Kestra version, edition, environment, and backend choices. Do not
   silently substitute `latest` for an unspecified production version.
2. Classify the selected backends before generating configuration. Use the current Kestra
   documentation or plugin catalog when a backend is not listed below; do not infer edition from
   the backend name alone.
3. If EE is selected, or any backend is EE-only, read
   [references/enterprise-edition.md](references/enterprise-edition.md) and complete its license
   gate before pulling EE artifacts, provisioning EE plugins, or starting Kestra.
4. Generate the target-specific installation files. Show the proposed files and commands before
   applying changes to a host or cluster. Keep credentials in the target's secret mechanism, not
   in source-controlled configuration.
5. Start Kestra, verify that the server is reachable, and for EE confirm only the non-sensitive
   startup result. Never print the license or render secret-bearing configuration as evidence.
6. Hand an installed instance to `kestra-ops` for contexts, flow deployment, executions, and
   namespace operations when that skill is available.

## Backend edition gate

Apply this gate to the backend configuration generator's output, not only to a user's initial
request.

| Backend selection | Edition | Required action |
|---|---|---|
| PostgreSQL or MySQL JDBC queue + repository | OSS or EE | Continue; no EE backend requirement |
| Kafka queue | EE | Confirm license and EE artifact/plugin access before proceeding |
| Redis or AMQP queue | EE | Confirm license and EE artifact/plugin access before proceeding |
| Elasticsearch or OpenSearch repository | EE | Confirm license and EE artifact/plugin access before proceeding |
| Separate external log data store | EE | Confirm the license and the required integration before proceeding |

For the deployment-recipe names used by this project:

- `jdbc-postgres` is OSS-compatible.
- `jdbc-postgres-redis` requires EE because the Redis queue is EE-only.
- `elasticsearch-redis` requires EE for the repository and queue choices.
- `elasticsearch-kafka` requires EE for the repository and queue choices.

When the gate finds an EE-only choice:

1. State which choice requires EE.
2. Confirm that `id`, `fingerprint`, and `key` are available through an approved secret source.
3. Confirm the target will use a version-matched EE image or JAR and can access required EE
   plugins.
4. If the license cannot be confirmed, stop before deployment and offer an OSS-compatible JDBC
   backend instead. Do not generate a configuration that is known to fail at startup.

## Target routing

- **Docker Compose:** use the pinned `registry.kestra.io/docker/kestra-ee` image for EE and inject
  `KESTRA_EE_LICENSE_*` through a local ignored env file or an external secret store.
- **Standalone host:** use the version-matched EE executable and a protected systemd
  `EnvironmentFile=` containing `KESTRA_EE_LICENSE_*`.
- **Kubernetes:** use the official chart, an EE image plus `imagePullSecrets`, and preferably an
  existing Kubernetes Secret referenced by `configurations.secrets`.

The exact secure patterns and verification rules are in
[references/enterprise-edition.md](references/enterprise-edition.md). Read that reference only
for EE installs or when the backend gate detects an EE-only component.

## Scope boundary

This skill covers license delivery and the edition gate for install-time components. Do not add
or recommend `workerGroups` or other EE worker-topology configuration as part of this workflow;
that topology is intentionally out of scope until the Kestra 2.0 design stabilizes.
