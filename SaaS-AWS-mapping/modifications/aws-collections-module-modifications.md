# Modifications to Existing Modules — amazon.aws / community.aws / amazon.cloud (Consolidated)

Changes to **existing** modules (add a parameter, an action, or fix packaging) so the SaaS repos can
replace raw `aws` CLI calls with modules. For brand-new modules, see `aws-collections-new-modules.md`.

Consolidated across: `ansible-saas-sre`, `ansible-saas-sre-operations`, `management-lifecycle`,
`customer-lifecycle-aws`, `ansible-saas-sops`.

## Table — Proposed Modifications

| Priority | Module | Collection | Modification | Replaces AWS CLI | Used in repos |
|---|---|---|---|---|---|
| P1 | `rds_cluster` | `amazon.aws` | Add Aurora Global Cluster **switchover** (zero-data-loss) and **managed failover** actions (today only `promote` / `remove_from_global_db`) | `rds failover-global-cluster` / `switchover-global-cluster` | customer-lifecycle |
| P1 | `rds_global_cluster` | `amazon.cloud` | Declarative (Cloud Control) create/update only — document limitation and/or add imperative failover/switchover, or defer to `amazon.aws.rds_cluster` | `rds *-global-cluster` | customer-lifecycle |
| P1 | `secretsmanager_secret` | `amazon.aws` / `community.aws` | **Fix packaging:** module exists only in `community.aws`; `amazon.aws` ships a lookup + a **dangling `aws_secret` redirect** to a non-existent `amazon.aws.secretsmanager_secret` module. Decide the home and align `meta/runtime.yml` | — (FQCN hygiene) | all repos |
| P2 | `secretsmanager_secret` | `community.aws` | Add imperative replication ops: **promote replica** (`stop-replication-to-replica`) and explicit `remove-regions-from-replication` beyond the declarative `replica:` list | `secretsmanager stop-replication-to-replica` / `remove-regions-from-replication` | customer-lifecycle, management-lifecycle |
| P2 | `rds_instance` | `amazon.aws` | Support **apply pending-maintenance action** | `rds apply-pending-maintenance-action` | sre |
| P2 | `iam_role` | `amazon.aws` | Support AWS **service-linked** roles (`AWSServiceName`) — or split into new `iam_service_linked_role` | `iam create-service-linked-role` | management-lifecycle |
| P2 | `elb_application_lb` | `amazon.aws` | Confirm/extend tag management so `add-tags` is fully declarative (no standalone ELB-tag module today) | `elbv2 add-tags` | customer-lifecycle |
| P3 | `wafv2_web_acl_association` | `amazon.cloud` | Already supports disassociate via `state: absent` — **no change**; documented here so repos adopt it instead of `aws wafv2 disassociate-web-acl` | `wafv2 disassociate-web-acl` | customer-lifecycle |

## Notes by collection

### `amazon.aws`
- **`rds_cluster`** is the strongest modification target: Aurora **switchover/failover** for MRBC/DR is a
  genuine gap today and needed by `customer-lifecycle-aws`'s `failover_deployment`.
- **`rds_instance`**, **`iam_role`**, **`elb_application_lb`** need incremental options only.
- Resolve the **`secretsmanager_secret`** redirect so the FQCN the repos already use
  (`amazon.aws.secretsmanager_secret`) either works or is removed.

### `community.aws`
- Home of the **`secretsmanager_secret`** module — extend it for imperative replica promotion/removal.
- Candidate home for a ported **`wafv2_logging_configuration`** (see new-modules doc) for parity.

### `amazon.cloud`
- Currently **unused** by every SaaS repo but already provides `wafv2_logging_configuration`,
  `wafv2_web_acl_association`, `rds_global_cluster`, `s3_bucket`, `kms_*`, `iam_role`, `eks_*`.
- Its **`rds_global_cluster`** is declarative-only (no failover/switchover) — that is why the imperative
  Aurora ops are routed to `amazon.aws.rds_cluster` above.
- No modification strictly required; the recommendation is **adoption** where it uniquely covers a need
  (WAF logging), tracked in `ansible-saas-repos-modifications.md`.
