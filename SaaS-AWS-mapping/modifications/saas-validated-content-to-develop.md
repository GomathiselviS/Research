# New Validated Content to Develop (from Ansible-SaaS use cases)

**Date:** 2026-08-18

Recurring, duplicated, or CLI-only patterns across the Ansible-SaaS repos that are generic enough to
become **validated content** — either new roles in `cloud.aws_ops` / `cloud.aws_troubleshooting`, or in
the internal `redhat.customer_lifecycle_aws` collection (already imported by `ansible-saas-sre`).

This is the companion to [`ansible-saas-repos-modifications.md`](ansible-saas-repos-modifications.md),
whose *"Adopt existing validated content now"* section covers content that **already exists**. This doc
covers content that **must be built**. Module-level gaps (upstream `amazon.aws`/`community.aws`
contributions) live in [`aws-collections-new-modules.md`](aws-collections-new-modules.md) and
[`aws-collections-module-modifications.md`](aws-collections-module-modifications.md); the roles here can
**wrap the raw CLI now** and swap to those modules once they land.

## Repos analysed

`ansible-saas-sre`, `ansible-saas-sre-operations`, `management-lifecycle`, `customer-lifecycle-aws`
(+ `ansible-saas-sops` runbooks). See the per-repo mapping docs and the README for details.

## Selection criteria

A pattern is a validated-content candidate when it is (1) **recurring across ≥2 repos** or very
high-volume in one, (2) **duplicated / CLI-only** today, and (3) **generic** enough to be useful outside
the SaaS context (not ROSA/AAP-specific orchestration).

---

## Proposed roles (prioritized)

### P1 — Chained assume-role credentials (extend `aws_setup_credentials`)

- **Driven by:** all four repos — ~170 `sts_assume_role` calls total (sre 59, sre-ops 35,
  management-lifecycle 35, customer-lifecycle 41), each wrapped in bespoke `set_management_cloud_account`
  / `get_rosa_credential` glue.
- **Gap vs. existing:** `cloud.aws_ops.aws_setup_credentials` builds a `module_defaults` dict for
  `group/aws` but does not support **role chaining** (management account → customer account) or
  session tagging.
- **Deliverable:** extend `aws_setup_credentials` (or a thin wrapper role) to accept an assumed-role ARN
  + external ID + session name and emit the same `aws_setup_credentials__output` dict, so every AWS task
  in every repo shares one credentials contract.
- **Leverage:** highest — touches every repo and the single most-called module.
- **Home:** `cloud.aws_ops` (upstream-friendly) with a SaaS-specific wrapper in
  `redhat.customer_lifecycle_aws` if chaining logic is Red-Hat-account-specific.

### P1 — Aurora global-cluster managed switchover / failover

- **Driven by:** `customer-lifecycle-aws` (`failover_deployment.yml`, `refresh_mrbc_aurora_writer.yml`),
  MRBC flow.
- **Gap vs. existing:** `cloud.aws_ops.create_rds_global_cluster` covers create/delete/detach only;
  `amazon.aws.rds_cluster` offers `promote`/detach but **not** `switchover-global-cluster` (zero data
  loss) or `failover-global-cluster`. No module anywhere. **The #1 true gap.**
- **Deliverable:** a role that performs planned switchover (zero-data-loss) and unplanned failover,
  returning the new writer region/endpoint as facts (feeds `refresh_mrbc_aurora_writer` / Dynatrace).
  Wraps `aws rds switchover-global-cluster` / `failover-global-cluster` until an upstream module exists.
- **Home:** `redhat.customer_lifecycle_aws` now (SaaS-driven), promote to `cloud.aws_ops` later.

### P1 — Aurora global-cluster create/delete parity for the SaaS use case (enhance `create_rds_global_cluster`)

- **Driven by:** `customer-lifecycle-aws`, `management-lifecycle` storage-infrastructure creation.
- **Gap vs. existing:** the role hardcodes one-primary + one-replica with derived names; the repos drive
  naming from customer/instance IDs + Terraform state and track multiple candidate writer regions.
- **Deliverable (enhancements to the existing role):**
  - Parameterize cluster/instance names (don't force `<global>-primary` / `<global>-replica`).
  - Allow **N replica regions**.
  - Add `storage_encrypted` / `kms_key_id` inputs (repos create a customer KMS key first).
  - Return writer endpoint/region as role facts.
- **Home:** `cloud.aws_ops` (upstream enhancement to `create_rds_global_cluster`).

### P2 — VPC PrivateLink provider-side setup/teardown role

- **Driven by:** sre, sre-operations, management-lifecycle, customer-lifecycle, sops — provider-side
  PrivateLink is CLI-only in **every** repo (`ec2 modify-vpc-endpoint-service-configuration` /
  `-permissions`, `describe`/`reject-vpc-endpoint-connections`).
- **Gap vs. existing:** no module (consumer-side `ec2_vpc_endpoint` + `ec2_vpc_endpoint_service_info`
  exist; provider side does not).
- **Deliverable:** a role that manages endpoint-service config, allowed-principal permissions, and
  connection accept/reject. **Wrap the CLI now**, swap to `ec2_vpc_endpoint_service` /
  `ec2_vpc_endpoint_connection` once upstreamed (tracked in `aws-collections-new-modules.md`).
- **Home:** `redhat.customer_lifecycle_aws` (shared across repos), or `cloud.aws_ops` once the modules land.

### P2 — Route 53 Resolver DNS Firewall association role

- **Driven by:** management-lifecycle (`create_shared_dns_firewall.yml`,
  `manage_shared_dns_firewall_associations.yml`), sre, customer-lifecycle (`shared_dns_firewal_association`).
- **Gap vs. existing:** whole `route53resolver` firewall subsystem is 100% CLI; no module in any
  collection.
- **Deliverable:** a role for firewall rule-group / domain-list / association management. Wrap CLI now,
  swap to the `route53resolver_firewall_*` module family once upstreamed.
- **Home:** `redhat.customer_lifecycle_aws` now, promote later.

### P3 — Secrets Manager multi-region replication role

- **Driven by:** management-lifecycle (**28×** `replicate-secret-to-regions` — the most-duplicated CLI in
  the codebase), customer-lifecycle, sre.
- **Gap vs. existing:** the module capability already exists (`community.aws.secretsmanager_secret`
  `replica:` param) — this is a **dedup/consistency** role, not a missing capability.
- **Deliverable:** a thin validated role over `community.aws.secretsmanager_secret` that takes a secret +
  region list and enforces replica state idempotently (plus the `amazon.aws` lookup for reads), so the 28
  copies collapse to one interface.
- **Home:** `redhat.customer_lifecycle_aws`.

---

## Summary

| Priority | Role | Driven by | Existing basis | Home |
|---|---|---|---|---|
| P1 | Chained assume-role credentials | all 4 (~170 STS calls) | extend `aws_setup_credentials` | `cloud.aws_ops` (+ wrapper) |
| P1 | Aurora switchover / failover | customer-lifecycle | none (true gap) | `redhat.customer_lifecycle_aws` |
| P1 | Aurora create/delete SaaS parity | customer-lifecycle, mgmt | enhance `create_rds_global_cluster` | `cloud.aws_ops` |
| P2 | PrivateLink provider-side setup/teardown | all 4 + sops | none (wrap CLI) | `redhat.customer_lifecycle_aws` |
| P2 | DNS Firewall association | mgmt, sre, customer | none (wrap CLI) | `redhat.customer_lifecycle_aws` |
| P3 | Secrets multi-region replication | mgmt (28×), customer, sre | dedup over existing module | `redhat.customer_lifecycle_aws` |

**Two tracks:** P1/P3 build on capabilities that exist today (extend a role, wrap an existing module);
the P2 roles are **blocked on upstream modules** but a validated role can wrap the CLI now and swap to
the module later — giving the SaaS teams one stable interface either way.
