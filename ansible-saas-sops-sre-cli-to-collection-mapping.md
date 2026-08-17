# ansible-saas-sops + ansible-saas-sre — AWS CLI → Collection Mapping & Gap Analysis

**Date:** 2026-08-17
**Repos analyzed:**
- [`Ansible-SaaS/ansible-saas-sops`](https://github.com/Ansible-SaaS/ansible-saas-sops) — documentation / SOPs & runbooks (docs-only)
- [`Ansible-SaaS/ansible-saas-sre`](https://github.com/Ansible-SaaS/ansible-saas-sre) — core SRE tooling: playbooks, plugins/modules, inventory plugin, Python CLI ("Ansible as a Service CLI")

**Collections compared (local dev checkouts):** `amazon.aws` (141 modules), `community.aws` (138), `amazon.cloud` (63).

> Companion docs in this repo already cover the other AAP-SaaS Ansible repos: `sre-ops-aws-collections-comparison.md` (`ansible-saas-sre-operations`), `management-lifecycle-cli-to-collection-mapping.md`, and `customer-lifecycle-aws-module-mapping.md`. Section 6 consolidates the **recurring cross-repo gaps** worth upstreaming.

---

## 1. Executive Summary

- **`ansible-saas-sops` is documentation-only.** It contains **no Ansible module calls** and only ~9 manual `aws` CLI commands inside runbooks (mostly PrivateLink troubleshooting + secrets/STS). Nothing to migrate; it's a knowledge base.
- **`ansible-saas-sre` is the AWS-heavy sibling** — ~37 distinct `amazon.aws`/`community.aws` modules and ~18 raw `aws` CLI verbs.
- **Most CLI usage is already covered** by existing modules and should be migrated (VPC endpoint CRUD, ELB/RDS describe, STS, Secrets Manager, WAF web-ACL).
- **The persistent, un-covered gaps** (no module in any of the three collections) are the *same three* seen across every SaaS repo:
  1. **VPC Endpoint *Service* (provider side)** — config, permissions, and connection accept/reject. **Strongest recurring gap.**
  2. **Route 53 Resolver DNS Firewall** (`route53resolver` rule-groups / associations).
  3. Plus repo-specific: `rds apply-pending-maintenance-action`, `sts decode-authorization-message`.
- **`amazon.cloud` is unused** but is the *only* collection with `wafv2_logging_configuration` — relevant to `ansible-saas-sre`'s WAF logging CLI calls.

---

## 2. `ansible-saas-sops` (docs) — AWS CLI in Runbooks

Docs-only. The genuine `aws` commands (rest were prose fragments / DNS hostnames like `*.aws.ansiblecloud`, not CLI):

| CLI in runbooks (count) | Where | Maps to module? |
|---|---|---|
| `aws ec2 modify-vpc-endpoint` (2) | PrivateLink connectivity runbooks | ✅ `amazon.aws.ec2_vpc_endpoint` |
| `aws ec2 describe-vpc-endpoint-service-configurations` (1) | disconnect_privatelink | ✅ `amazon.aws.ec2_vpc_endpoint_service_info` (info) |
| `aws ec2 describe-vpc-endpoint-connections` (1) | PrivateLink | ❌ no module (provider connections) |
| `aws sts get-caller-identity` (2) | onboarding / connectivity | ✅ `amazon.aws.aws_caller_info` |
| `aws sts decode-authorization-message` (1) | incident/troubleshooting | ❌ no module (niche) |
| `aws secretsmanager get-secret-value` (2) | connectivity | ✅ lookup `amazon.aws.secretsmanager_secret` |
| `aws secretsmanager describe-secret` (1) | connectivity | ⚠️ `community.aws.secretsmanager_secret` (register result) |
| `aws configure export-credentials` (1) | onboarding | N/A (credential helper, not mappable) |

**Files with the most `aws` usage:** `disconnect_privatelink.md`, `private_connectivity_from_control_plane.md`, `private_connectivity_from_aap.md`, `private_connectivity_from_customer_vpc.md`, `new_region_support.md`, `vanity_domain.md`, `onboarding_guide.md`. These are **human runbooks**, so leaving them as CLI is acceptable — but they should point at the corresponding modules where they exist.

---

## 3. `ansible-saas-sre` — Modules Used (FQCN counts)

**`amazon.aws`:** `sts_assume_role` (59), `secretsmanager_secret` (23 — see §5), `s3_object` (14), `route53` (8), `s3_object_info` (6), `rds_instance` (4), `kms_key` (4), `elb_application_lb_info` (4), `ec2_vpc_net_info` (4), `ec2_vpc_endpoint_service_info` (4), `aws_secret` (4, deprecated alias), `ec2_vpc_endpoint_info` (3), `ec2_vpc_endpoint` (3), `ec2_tag` (3), `ec2_security_group` (3), `aws_caller_info` (3), `s3_bucket_info` (2), `s3_bucket` (2), `route53_info` (2), `kms_key_info` (2), `elb_application_lb` (2), `ec2_vpc_subnet` (2), `ec2_vpc_route_table` (2), `ec2_tag_info` (2), plus 1× each: `rds_snapshot_info`, `ec2_vpc_subnet_info`, `ec2_vpc_net`, `ec2_vpc_nat_gateway`, `ec2_vpc_igw`, `ec2_security_group_info`, `ec2_key`, `ec2_instance`, `cloudwatchlogs_log_group_info`.

**`community.aws`:** `secretsmanager_secret` (12), `route53_wait` (2), `acm_certificate_info` (2), `acm_certificate` (2), `wafv2_web_acl_info` (1).

**`amazon.cloud`:** none.

**boto3 (Python CLI code):** only `boto3.client('sts')` + `get_caller_identity`; Secrets Manager `get_secret_value`/`create_secret`/`update_secret`. Everything else routes through Ansible modules — a healthy sign.

---

## 4. `ansible-saas-sre` — Raw `aws` CLI → Module Mapping

### 4a. ✅ Already covered — migrate CLI → module

| Raw CLI (count) | Target module | Notes |
|---|---|---|
| `aws ec2 create-vpc-endpoint` (1) | `amazon.aws.ec2_vpc_endpoint` | Consumer endpoint create. Already used elsewhere. |
| `aws ec2 modify-vpc-endpoint` (1) | `amazon.aws.ec2_vpc_endpoint` | Modify via `state: present`. |
| `aws ec2 describe-vpc-endpoints` (2) | `amazon.aws.ec2_vpc_endpoint_info` | |
| `aws ec2 describe-vpc-endpoint-service-configurations` (4) | `amazon.aws.ec2_vpc_endpoint_service_info` | Info only; already used. |
| `aws elbv2 describe-load-balancers` (1) | `amazon.aws.elb_application_lb_info` | Already used. |
| `aws wafv2 get-web-acl` (1) / `list-web-acls` (1) | `community.aws.wafv2_web_acl_info` | Already used (`wafv2_web_acl_info`). |
| `aws wafv2 update-web-acl` (1) | `community.aws.wafv2_web_acl` | Manage rules/visibility via module. |

### 4b. ⚠️ Partially covered — module exists elsewhere / limited action

| Raw CLI (count) | Nearest module | Gap |
|---|---|---|
| `aws wafv2 put-logging-configuration` (1) / `delete-logging-configuration` (1) | `amazon.cloud.wafv2_logging_configuration` | **Only in `amazon.cloud`** (unused here). No equivalent in `community.aws`/`amazon.aws`. |
| `aws rds apply-pending-maintenance-action` (1) | `amazon.aws.rds_instance` | `rds_instance` does not apply pending-maintenance actions. |

### 4c. ❌ No module in any collection — CLI is the only option today

| Raw CLI (count) | Service | Coverage |
|---|---|---|
| `aws ec2 modify-vpc-endpoint-service-configuration` (4) | VPC Endpoint **Service** (provider) | **No module.** Only `_service_info` (read) exists. |
| `aws ec2 modify-vpc-endpoint-service-permissions` (2) | VPC Endpoint Service permissions | **No module.** |
| `aws ec2 describe-vpc-endpoint-connections` (1) | VPC Endpoint Service connections | **No module.** |
| `aws ec2 reject-vpc-endpoint-connections` (1) | VPC Endpoint Service connections | **No module.** |
| `aws route53resolver list-firewall-rule-groups` (1) / `list-firewall-rule-group-associations` (2) / `associate-firewall-rule-group` (1) / `disassociate-firewall-rule-group` (1) | **Route 53 Resolver DNS Firewall** | **No module anywhere** (`community.aws.networkfirewall*` is a different service). |

---

## 5. `secretsmanager_secret` — Same FQCN Reality Check

- The **module** `secretsmanager_secret.py` exists **only in `community.aws`**; `amazon.aws` ships only a **lookup** by that name.
- `ansible-saas-sre` uses **both** `amazon.aws.secretsmanager_secret` (23×, module form — would not resolve) and `community.aws.secretsmanager_secret` (12×), plus the deprecated `amazon.aws.aws_secret` alias (4×).
- **Action:** standardize on `community.aws.secretsmanager_secret` for module tasks and the `amazon.aws` lookup for reads; drop the `aws_secret` alias. (Same fix recommended for every SaaS repo.)

---

## 6. Consolidated Cross-Repo Gaps — What to Add/Modify in the AWS Collections

These gaps recur across `ansible-saas-sre`, `ansible-saas-sre-operations`, `management-lifecycle`, and the sops runbooks — strong candidates for upstream contribution.

### Priority 1 — New modules (no coverage anywhere)

1. **VPC Endpoint *Service* (provider side)** — the single most recurring gap:
   - `ec2_vpc_endpoint_service` — create/modify endpoint **service configuration**.
   - `ec2_vpc_endpoint_service` permissions (allowed principals) — `modify-vpc-endpoint-service-permissions`.
   - `ec2_vpc_endpoint_connection` (+ `_info`) — accept/reject/describe endpoint **connections**.
   - *Rationale:* PrivateLink is core to AAP SaaS; today all provider-side ops are `aws ec2 …` CLI in every repo. `amazon.aws` already has consumer-side `ec2_vpc_endpoint` + `ec2_vpc_endpoint_service_info` — this closes the provider side.

2. **Route 53 Resolver DNS Firewall** family in `amazon.aws`:
   - `route53resolver_firewall_domain_list` (+ `_info`), `route53resolver_firewall_rule_group` (+ `_info`), `route53resolver_firewall_rule`, `route53resolver_firewall_rule_group_association`, with standard tagging.

3. **AWS Account Management** (`account_alternate_contact`, `account_region_info` / enable-disable) — from `management-lifecycle`.

4. **IAM OIDC identity provider** (`iam_openid_connect_provider` + `_info`) — ROSA/OpenID IdP setup.

### Priority 2 — Extend existing modules

5. **`amazon.aws.rds_instance`** — support `apply-pending-maintenance-action`.
6. **`amazon.aws.iam_role`** — support AWS **service-linked** roles (or new `iam_service_linked_role`).
7. **WAF logging parity** — add `wafv2_logging_configuration` to `community.aws` (currently only in `amazon.cloud`), *or* adopt `amazon.cloud` for WAF logging in the SaaS repos.
8. **`sts` niche** — optional `sts_decode_authorization_message` (low priority; troubleshooting only).

### Priority 3 — Consistency / hygiene (no collection code change)

9. **Standardize `secretsmanager_secret` FQCN** across all repos and fix the dangling `amazon.aws` redirect (points to a non-existent module; only the lookup exists).
10. **Pin AWS collection versions consistently** across the SaaS repos (they currently mix pinned 8.x and unpinned) so module resolution is identical everywhere.

---

## 7. Recommendations

1. **Migrate `ansible-saas-sre`'s "already covered" CLI** (§4a) to modules — VPC endpoint CRUD, ELB/WAF/RDS describe — for idempotency and check-mode.
2. **Adopt `amazon.cloud.wafv2_logging_configuration`** for the WAF logging CLI calls (the only module path today).
3. **Keep `ansible-saas-sops` as CLI runbooks**, but cross-link each command to its module equivalent where one exists.
4. **Upstream the Priority-1 modules** (esp. VPC endpoint service / connections) — highest leverage since the same gap forces CLI in all four SaaS repos.
5. **Fix the `secretsmanager_secret` FQCN** and drop the `aws_secret` alias everywhere.

---

## Appendix — Raw CLI Inventories

**`ansible-saas-sre`:**
```
4  aws ec2 modify-vpc-endpoint-service-configuration
4  aws ec2 describe-vpc-endpoint-service-configurations
2  aws route53resolver list-firewall-rule-group-associations
2  aws ec2 modify-vpc-endpoint-service-permissions
2  aws ec2 describe-vpc-endpoints
1  aws wafv2 update-web-acl / put-logging-configuration / list-web-acls / get-web-acl / delete-logging-configuration
1  aws route53resolver list-firewall-rule-groups / disassociate-firewall-rule-group / associate-firewall-rule-group
1  aws rds apply-pending-maintenance-action
1  aws elbv2 describe-load-balancers
1  aws ec2 reject-vpc-endpoint-connections / modify-vpc-endpoint / describe-vpc-endpoint-connections / create-vpc-endpoint
```
boto3: `boto3.client('sts')` (get_caller_identity); Secrets Manager get/create/update_secret.

**`ansible-saas-sops` (docs):**
```
2  aws sts get-caller-identity
2  aws secretsmanager get-secret-value
2  aws ec2 modify-vpc-endpoint
1  aws sts decode-authorization-message
1  aws secretsmanager describe-secret
1  aws ec2 describe-vpc-endpoint-service-configurations
1  aws ec2 describe-vpc-endpoint-connections
1  aws configure export-credentials
```

---

## Appendix — Scope & Method

- Both repos cloned via `git clone` with `gh auth token` (internal/private).
- Usage extracted with `grep -rhoE` for `aws <service> <verb>`, FQCN module patterns, and `boto3.client/resource`.
- Coverage verified against `plugins/modules/*.py` and `meta/runtime.yml` in each local collection.
- **Out of scope** (not CLI/Ansible-module mappable): the Terraform (HCL) repos (`model-aap-deployment`, `management-aap-deployment`, `model-customer-aws`, per-instance `cus-*`/`mgt-*`) and the Go microservices (`ansible-saas-management-service`, `*-service`) — these provision AWS via Terraform/SDK and map to `cloud.terraform`, not the AWS module collections.

---

# Summary Tables

## Table 1 — Repos Analysed (sops + siblings named in its README)

| Repo | Type / Language | Role in AAP SaaS | AWS surface | In scope for CLI→module mapping? |
|---|---|---|---|---|
| `ansible-saas-sops` | Docs (Markdown) | Central SOPs / runbooks / architecture / onboarding | ~9 manual `aws` CLI cmds in runbooks; **no Ansible modules** | ✅ (docs only) |
| `ansible-saas-sre` | Ansible + Python CLI | Core SRE tooling, playbooks, plugins, inventory, `ansible-saas` CLI | ~37 modules, ~18 `aws` CLI verbs, minimal boto3 | ✅ (primary) |
| `model-aap-deployment` | Terraform (HCL) | Customer deployment model / IaC | AWS via Terraform resources | ❌ maps to `cloud.terraform`, not AWS modules |
| `management-aap-deployment` | Terraform (HCL) | Management-env deployment / IaC | AWS via Terraform resources | ❌ maps to `cloud.terraform` |
| `ansible-saas-management-service` | Go | Management service implementation | AWS via Go SDK | ❌ not Ansible/CLI |

> Related SaaS Ansible repos (covered in separate docs): `ansible-saas-sre-operations`, `management-lifecycle`, `customer-lifecycle-aws`.

## Table 2 — AWS CLI → Ansible Module Mapping (sops + ansible-saas-sre)

Legend: ✅ covered (migrate to module) · ⚠️ partial / only in one collection · ❌ no module (gap)

| Repo | Raw AWS CLI | Count | Target module | Status |
|---|---|---|---|---|
| both | `aws ec2 create-vpc-endpoint` | 1 | `amazon.aws.ec2_vpc_endpoint` | ✅ |
| both | `aws ec2 modify-vpc-endpoint` | 3 | `amazon.aws.ec2_vpc_endpoint` (`state: present`) | ✅ |
| sre | `aws ec2 describe-vpc-endpoints` | 2 | `amazon.aws.ec2_vpc_endpoint_info` | ✅ |
| both | `aws ec2 describe-vpc-endpoint-service-configurations` | 5 | `amazon.aws.ec2_vpc_endpoint_service_info` | ✅ (info) |
| sre | `aws elbv2 describe-load-balancers` | 1 | `amazon.aws.elb_application_lb_info` | ✅ |
| sre | `aws wafv2 get-web-acl` / `list-web-acls` | 2 | `community.aws.wafv2_web_acl_info` | ✅ |
| sre | `aws wafv2 update-web-acl` | 1 | `community.aws.wafv2_web_acl` | ✅ |
| sops | `aws sts get-caller-identity` | 2 | `amazon.aws.aws_caller_info` | ✅ |
| both | `aws secretsmanager get-secret-value` | 4 | lookup `amazon.aws.secretsmanager_secret` | ✅ |
| sops | `aws secretsmanager describe-secret` | 1 | `community.aws.secretsmanager_secret` (register) | ⚠️ |
| sre | `aws wafv2 put-logging-configuration` / `delete-logging-configuration` | 2 | `amazon.cloud.wafv2_logging_configuration` | ⚠️ only in `amazon.cloud` |
| sre | `aws rds apply-pending-maintenance-action` | 1 | `amazon.aws.rds_instance` (no such action) | ⚠️ gap |
| sre | `aws ec2 modify-vpc-endpoint-service-configuration` | 4 | — | ❌ no module |
| sre | `aws ec2 modify-vpc-endpoint-service-permissions` | 2 | — | ❌ no module |
| both | `aws ec2 describe-vpc-endpoint-connections` | 2 | — | ❌ no module |
| sre | `aws ec2 reject-vpc-endpoint-connections` | 1 | — | ❌ no module |
| sre | `aws route53resolver *` (firewall rule-groups / associations) | 6 | — | ❌ no module (whole subsystem) |
| sops | `aws sts decode-authorization-message` | 1 | — | ❌ no module (niche) |
| sops | `aws configure export-credentials` | 1 | N/A (credential helper) | — |

## Table 3 — Modules Currently Used by `ansible-saas-sre`

| Collection | Modules (with usage counts) |
|---|---|
| `amazon.aws` | `sts_assume_role` (59), `secretsmanager_secret`* (23), `s3_object` (14), `route53` (8), `s3_object_info` (6), `rds_instance` (4), `kms_key` (4), `elb_application_lb_info` (4), `ec2_vpc_net_info` (4), `ec2_vpc_endpoint_service_info` (4), `aws_secret`† (4), `ec2_vpc_endpoint_info` (3), `ec2_vpc_endpoint` (3), `ec2_tag` (3), `ec2_security_group` (3), `aws_caller_info` (3), `s3_bucket`/`s3_bucket_info` (2), `route53_info` (2), `kms_key_info` (2), `elb_application_lb` (2), `ec2_vpc_subnet` (2), `ec2_vpc_route_table` (2), `ec2_tag_info` (2); 1× each: `rds_snapshot_info`, `ec2_vpc_subnet_info`, `ec2_vpc_net`, `ec2_vpc_nat_gateway`, `ec2_vpc_igw`, `ec2_security_group_info`, `ec2_key`, `ec2_instance`, `cloudwatchlogs_log_group_info` |
| `community.aws` | `secretsmanager_secret` (12), `route53_wait` (2), `acm_certificate` (2), `acm_certificate_info` (2), `wafv2_web_acl_info` (1) |
| `amazon.cloud` | *(none)* |

\* `amazon.aws.secretsmanager_secret` is a **lookup** only; the **module** lives in `community.aws` — mixed usage is a latent bug (see Table 5).
† `aws_secret` is a deprecated alias — should be dropped.
`ansible-saas-sops` uses **no** Ansible modules.

## Table 4 — Modifications / Additions to the AWS Collections

| Priority | Change | Target collection | Type | Driven by (CLI/gap) |
|---|---|---|---|---|
| P1 | `ec2_vpc_endpoint_service` — create/modify endpoint **service** configuration | `amazon.aws` | New module | `modify-vpc-endpoint-service-configuration` (×4) |
| P1 | `ec2_vpc_endpoint_service` **permissions** (allowed principals) | `amazon.aws` | New module / option | `modify-vpc-endpoint-service-permissions` (×2) |
| P1 | `ec2_vpc_endpoint_connection` (+ `_info`) — accept/reject/describe | `amazon.aws` | New module | `describe`/`reject-vpc-endpoint-connections` |
| P1 | Route 53 Resolver **DNS Firewall** family (`route53resolver_firewall_domain_list`, `_rule_group`, `_rule`, `_rule_group_association` + `_info`) | `amazon.aws` | New modules | `route53resolver *` (×6) |
| P1 | `account_alternate_contact`, `account_region_info` / enable-disable | `amazon.aws` | New modules | (from `management-lifecycle`) |
| P1 | `iam_openid_connect_provider` (+ `_info`) | `amazon.aws` | New module | (ROSA/OIDC IdP) |
| P2 | `rds_instance` — support `apply-pending-maintenance-action` | `amazon.aws` | Extend module | `rds apply-pending-maintenance-action` |
| P2 | `iam_role` — support AWS **service-linked** roles (or new `iam_service_linked_role`) | `amazon.aws` | Extend / new | `iam create-service-linked-role` |
| P2 | Add `wafv2_logging_configuration` for parity | `community.aws` | New module (port) | `wafv2 put/delete-logging-configuration` |
| P2 | Optional `sts_decode_authorization_message` | `amazon.aws` | New module (low pri) | `sts decode-authorization-message` |
| P3 | Consolidate `secretsmanager_secret` home & fix dangling redirect | `amazon.aws` / `community.aws` | Hygiene | mixed FQCN usage |

## Table 5 — Changes to Add to the SaaS Repos (sops + siblings)

| Repo | Change | Type | Rationale |
|---|---|---|---|
| `ansible-saas-sre` | Migrate "✅ covered" CLI (Table 2) to modules: VPC endpoint CRUD, `elb_application_lb_info`, WAF web-ACL, RDS/ELB describe | Refactor | Idempotency, check-mode, less brittle than shell |
| `ansible-saas-sre` | Standardize `secretsmanager_secret` on `community.aws` (module) + `amazon.aws` lookup (reads); drop `aws_secret` alias (×4) | Fix | `amazon.aws.secretsmanager_secret` **module** does not exist — latent failure |
| `ansible-saas-sre` | Adopt `amazon.cloud.wafv2_logging_configuration` for WAF logging CLI calls | Adopt collection | Only module path today; `amazon.cloud` currently unused |
| `ansible-saas-sre` | Pin AWS collection versions in `requirements.yml` consistent with sibling repos | Consistency | Repos currently mix pinned 8.x / unpinned → resolution drift |
| `ansible-saas-sre` | Keep genuinely-uncovered CLI (endpoint-service, route53resolver, pending-maintenance) but wrap + comment why | Doc/guard | No module exists yet; flag as tech debt pending Table 4 P1 |
| `ansible-saas-sops` | Cross-link each runbook `aws` command to its module equivalent where one exists | Docs | Runbooks stay CLI, but point readers to supported modules |
| `ansible-saas-sops` | Update PrivateLink runbooks once endpoint-service modules land (Table 4 P1) | Docs (follow-up) | Replace manual CLI steps with module tasks |
| `model-aap-deployment` / `management-aap-deployment` | Out of scope here (Terraform); track separately for `cloud.terraform` alignment | — | IaC, not Ansible-module mappable |
| `ansible-saas-management-service` | Out of scope (Go SDK) | — | Service code, not Ansible/CLI |

## Table 6 — Coverage Scorecard (in-scope repos)

| Metric | sops | ansible-saas-sre |
|---|---|---|
| Ansible AWS modules used | 0 | ~37 |
| Raw `aws` CLI verbs | ~8 (runbooks) | ~18 |
| CLI ✅ already covered by a module | 6 | 7 |
| CLI ⚠️ partial / only `amazon.cloud` | 1 | 2 |
| CLI ❌ no module (true gap) | 2 | 6 |
| `amazon.cloud` modules used | 0 | 0 |
