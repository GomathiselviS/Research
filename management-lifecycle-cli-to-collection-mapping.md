# management-lifecycle — Raw CLI → Ansible Collection Mapping & Gap Analysis

**Date:** 2026-08-17
**Repo analyzed:** [`Ansible-SaaS/management-lifecycle`](https://github.com/Ansible-SaaS/management-lifecycle) (internal, `main`)
**Purpose:** AAP SaaS **management-plane** lifecycle (provision / configure / upgrade / deprovision) on AWS + ROSA/OpenShift.
**Collections compared (local dev checkouts):**
- `amazon.aws` (12.0.0-dev0) — 141 modules
- `community.aws` (12.0.0-dev0) — 138 modules
- `amazon.cloud` (0.4.0) — 63 modules

> Repo pins **`amazon.aws` 8.2.1** and **`community.aws` 8.0.0** in `requirements.yml` (older than the local dev checkouts). Module availability below is verified against the local checkouts; a few modules may not yet exist in the pinned 8.x lines — noted where relevant.

---

## 1. Executive Summary

- The repo is heavily **CLI-driven**: much AWS work runs through `ansible.builtin.command`/`shell` calling the `aws` CLI, plus `rosa`, `oc`, `terraform`, `kubectl`, `kustomize`.
- Where it *does* use modules, the surface is dominated by IAM, Secrets Manager, STS, S3, Route53, and ELB — **almost all already covered** by `amazon.aws`/`community.aws`.
- **The biggest coverage gaps** (no module in any of the three collections) are:
  1. **Route 53 Resolver DNS Firewall** (`aws route53resolver *`) — a whole subsystem, used for shared DNS firewall management.
  2. **AWS Account management** (`aws account put-alternate-contact`, `list-regions`) — no `account` module anywhere.
  3. **VPC Endpoint *Service* configuration** (`aws ec2 modify-vpc-endpoint-service-configuration`) — only endpoint (consumer) + info modules exist.
  4. **IAM OIDC identity provider** (`aws iam create/get-open-id-connect-provider`) — only SAML federation module exists.
  5. **IAM service-linked roles** (`aws iam create-service-linked-role`) — not supported by `iam_role`.
- The highest-volume CLI call — `aws secretsmanager replicate-secret-to-regions` (28×) — is **already supported** by `community.aws.secretsmanager_secret`'s `replica` parameter and should be migrated to the module.

---

## 2. Module Usage Today (FQCN counts)

**`amazon.aws`:** `sts_assume_role` (35), `iam_role` (27), `iam_policy` (27), `secretsmanager_secret` (23 — see §5 note), `aws_caller_info` (12), `route53` (6), `kms_key` (6), `s3_bucket` (5), `s3_object` (4), `s3_bucket_info` (4), `elb_application_lb_info` (3), `ec2_vpc_endpoint_service_info` (3), `route53_info` (2), `iam_role_info` (2), `iam_policy_info` (2), `iam_instance_profile` (2), `elb_application_lb` (2), `cloudtrail` (2), `kms_key_info` (1), `iam_user`/`iam_user_info` (1), `iam_group` (1), `ec2_vpc_net_info` (1), `ec2_tag` (1), `ec2_security_group`/`_info` (1), `aws_secret` (1, deprecated alias).

**`community.aws`:** `secretsmanager_secret` (40), `s3_lifecycle` (2), `acm_certificate_info` (2), `s3_bucket_notification` (1), `route53_wait` (1).

**`amazon.cloud`:** none.

> ⚠️ Same inconsistency as the SRE-ops repo: **both** `community.aws.secretsmanager_secret` (40×) **and** `amazon.aws.secretsmanager_secret` (23×) are used. The **module** lives only in `community.aws`; `amazon.aws` ships only a *lookup* by that name. Standardize on `community.aws.secretsmanager_secret`. (See §5.)

---

## 3. Raw `aws` CLI Usage → Module Mapping

### 3a. ✅ Already covered — migrate CLI → module

| Raw CLI (count) | Target module | Notes |
|---|---|---|
| `aws secretsmanager replicate-secret-to-regions` (28) | `community.aws.secretsmanager_secret` | Use the `replica` list param (region + optional kms_key_id). Already implemented. |
| `aws secretsmanager get-secret-value` (8) | `amazon.aws` lookup `secretsmanager_secret` **or** `community.aws.secretsmanager_secret` (register) | Prefer the lookup plugin for reads. |
| `aws secretsmanager put-secret-value` (4) | `community.aws.secretsmanager_secret` | `secret`/`json_secret` params. |
| `aws elbv2 describe-load-balancers` (5) | `amazon.aws.elb_application_lb_info` | Already used elsewhere in repo. |
| `aws s3 mb` (4) / `s3 rb` (1) | `amazon.aws.s3_bucket` | `state: present/absent`. |
| `aws s3api list-buckets` (2) / `head-bucket` (2) | `amazon.aws.s3_bucket_info` | |
| `aws s3 cp` (1) | `amazon.aws.s3_object` | Already used elsewhere. |
| `aws ec2 describe-vpcs` (3) | `amazon.aws.ec2_vpc_net_info` | |
| `aws ec2 describe-nat-gateways` (2) | `amazon.aws.ec2_vpc_nat_gateway_info` | |
| `aws ec2 describe-addresses` (2) | `amazon.aws.ec2_eip_info` | |
| `aws ec2 describe-vpc-endpoint-service-configurations` (2) | `amazon.aws.ec2_vpc_endpoint_service_info` | Already used (info-only). |
| `aws rds describe-db-instances` (2) | `amazon.aws.rds_instance_info` | |
| `aws rds describe-db-clusters` (2) | `amazon.aws.rds_cluster_info` | |
| `aws rds describe-db-subnet-groups` (2) | `amazon.aws.rds_subnet_group` (info via check) / CLI acceptable | Info module is thin; minor. |
| `aws iam list-roles` (2) | `amazon.aws.iam_role_info` | |
| `aws iam list-attached-role-policies` (2) | `amazon.aws.iam_role_info` | Returns attached policies. |
| `aws sts assume-role` (2, scripts) | `amazon.aws.sts_assume_role` | Already used heavily in playbooks. |
| `aws secretsmanager restore-secret` (1, script) | `community.aws.secretsmanager_secret` | `recovery_window`/`state: present` un-deletes. |
| `aws secretsmanager delete-secret` (1, script) | `community.aws.secretsmanager_secret` | `state: absent`. |

### 3b. ⚠️ Partially covered — module exists but missing the specific action

| Raw CLI (count) | Nearest module | Gap |
|---|---|---|
| `aws ec2 modify-vpc-endpoint-service-configuration` (3) | `amazon.aws.ec2_vpc_endpoint_service_info` (info only) | **No module manages the endpoint *service*** (create/modify/permissions). Consumer side (`ec2_vpc_endpoint`) exists; provider side does not. |
| `aws iam create-service-linked-role` (2) | `amazon.aws.iam_role` | `iam_role` does not create AWS **service-linked** roles (`AWSServiceRoleFor*`). |
| `aws account put-alternate-contact` (3) | *(none)* | See §3c. |

### 3c. ❌ No module in any collection — CLI is the only option today

| Raw CLI (count) | Service | Coverage |
|---|---|---|
| `aws route53resolver *` (13 across create/delete/list/associate firewall rule-groups, domain-lists, rules, associations, tags) | **Route 53 Resolver DNS Firewall** | **No module anywhere.** `community.aws.networkfirewall*` is a *different* service (AWS Network Firewall). Biggest single gap. |
| `aws account put-alternate-contact` (3) / `list-regions` (1) | **AWS Account Management API** | No `account` module in any collection. |
| `aws iam create-open-id-connect-provider` (2) / `get-open-id-connect-provider` (2) | **IAM OIDC identity provider** | Only `community.aws.iam_saml_federation` (SAML) exists — no OIDC equivalent. |

### 3d. Not AWS-CLI (kept as CLI — no collection substitute expected)

`rosa login/create/list/delete/link` (ROSA cluster mgmt), `oc login`, `kubectl --token`, `terraform state/init/show/refresh/destroy/untaint` (managed via `cloud.terraform.terraform` where practical), `kustomize build`, `jq`/`yq`. These are legitimately CLI/wrapper workflows; `rosa` and `oc` have no first-class module coverage and should remain CLI.

---

## 4. Gap Summary — What to Add / Modify in the Collections

Prioritized by impact for this repo.

### Priority 1 — New modules (no coverage at all)

1. **Route 53 Resolver DNS Firewall** — a small module family in `amazon.aws` (or `community.aws`):
   - `route53resolver_firewall_domain_list` (+ `_info`)
   - `route53resolver_firewall_rule_group` (+ `_info`)
   - `route53resolver_firewall_rule`
   - `route53resolver_firewall_rule_group_association`
   - Tagging support (`tag-resource` / `list-tags-for-resource`) via standard `tags`/`purge_tags`.
   - *Rationale:* the repo's `create_shared_dns_firewall.yml` / `manage_shared_dns_firewall_associations.yml` are 100% CLI today.

2. **AWS Account Management** — `amazon.aws.account_*`:
   - `account_alternate_contact` (put/delete/get; billing/operations/security contact types).
   - `account_region_info` / region enable-disable (`list-regions`, `enable-region`, `disable-region`).
   - *Rationale:* used in `configure_alternate_contacts.yml`, multi-account/region setup.

3. **IAM OIDC identity provider** — `amazon.aws.iam_openid_connect_provider` (+ `_info`):
   - create/delete/get, thumbprint + client-id list management.
   - *Rationale:* ROSA/OpenID IdP setup (`create_rosa_openid_idp.yml`, `create_controller_rh_sso_idp.yml`).

### Priority 2 — Extend existing modules

4. **`amazon.aws.ec2_vpc_endpoint_service`** (new, provider side) *or* extend endpoint modules:
   - Manage VPC endpoint **service configurations** and **permissions** (`modify-vpc-endpoint-service-configuration`, `modify-vpc-endpoint-service-permissions`). Currently only `_service_info` (read) exists.

5. **`amazon.aws.iam_role` — service-linked role support:**
   - Add ability to create/manage AWS **service-linked roles** (`create-service-linked-role` with `AWSServiceName`), or a dedicated `iam_service_linked_role` module.

6. **`amazon.aws.rds_subnet_group_info`** (thin gap):
   - A dedicated info module for DB subnet groups (currently only the managed `rds_subnet_group`).

### Priority 3 — Consistency / hygiene (no code change to collections)

7. **Promote/standardize `secretsmanager_secret`** — decide whether the module lives in `amazon.aws` or `community.aws` and align the redirect in `amazon.aws/meta/runtime.yml` with an actual module (today it points to a non-existent `amazon.aws.secretsmanager_secret` module; only the lookup exists). See §5.

---

## 5. `secretsmanager_secret` — FQCN Reality Check

- The **module** `secretsmanager_secret.py` exists **only in `community.aws`** (`plugins/modules/`).
- `amazon.aws` ships **only a lookup plugin** named `secretsmanager_secret` (`plugins/lookup/`) — **no module**.
- `amazon.aws/meta/runtime.yml` has `aws_secret` → `amazon.aws.secretsmanager_secret`, but that module target does **not** exist in the local checkout.
- The module already supports **multi-region replication** via the `replica` param (`region` + optional `kms_key_id`), plus add/remove-replication logic — directly replacing the 28× `aws secretsmanager replicate-secret-to-regions` CLI calls.

**Action for this repo:** standardize all secret task calls on `community.aws.secretsmanager_secret` (or pin a published `amazon.aws` that actually ships the module), and replace the `replicate-secret-to-regions` / `put-secret-value` / `get-secret-value` / `restore` / `delete` CLI shell-outs with the module + lookup.

---

## 6. Migration Recommendations (Prioritized)

1. **Fix `secretsmanager_secret` FQCN** and migrate the SecretsManager CLI cluster (28 + 8 + 4 + others) to the module/lookup — highest volume, already fully supported.
2. **Replace low-risk read CLIs** (`elbv2 describe`, `s3api list/head`, `ec2 describe-vpcs/nat-gateways/addresses`, `rds describe-*`, `iam list-*`) with the existing `*_info` modules — pure wins, better idempotency & check-mode.
3. **Migrate `s3 mb/rb`, `s3 cp`, `sts assume-role`** to `s3_bucket`, `s3_object`, `sts_assume_role`.
4. **Pin AWS collections consistently** — this repo pins 8.x while sibling repos are unpinned; align so `secretsmanager_secret` and `route53_wait` resolve identically everywhere.
5. **File upstream requests / contribute modules** for the Priority-1 gaps (route53resolver DNS firewall, account management, IAM OIDC provider). Until then, keep those as documented CLI shell-outs.
6. **Keep `rosa`/`oc`/`terraform`/`kustomize` as CLI** — out of scope for AWS collections.

---

## Appendix — Full Raw `aws` CLI Inventory

**In playbooks:**
```
28  aws secretsmanager replicate-secret-to-regions
 8  aws secretsmanager get-secret-value
 5  aws elbv2 describe-load-balancers
 4  aws secretsmanager put-secret-value
 4  aws s3 mb
 3  aws route53resolver list-firewall-rule-groups
 3  aws ec2 modify-vpc-endpoint-service-configuration
 3  aws ec2 describe-vpcs
 3  aws account put-alternate-contact
 2  aws s3api list-buckets / head-bucket
 2  aws rds describe-db-subnet-groups / db-instances / db-clusters
 2  aws iam list-roles / list-attached-role-policies / get-open-id-connect-provider
 2  aws iam create-service-linked-role / create-open-id-connect-provider
 2  aws ec2 describe-vpc-endpoint-service-configurations / describe-nat-gateways / describe-addresses
 1  aws s3 rb / s3 cp
 1  aws route53resolver {create,delete,list,associate,disassociate,import,tag}-firewall-* (11 distinct verbs)
 1  aws account list-regions
```

**In tools/scripts:**
```
2  aws sts assume-role
2  aws secretsmanager get-secret-value
1  aws secretsmanager restore-secret
1  aws secretsmanager delete-secret
```

**Other CLIs (kept):** `terraform state/init/show/refresh/destroy/untaint`, `rosa login/create/list/delete/link/nlb`, `oc login`, `kubectl --token`, `kustomize build`, `jq`, `yq`.

---

## Appendix — Method

- Repo cloned via `git clone` with `gh auth token` (internal/private; public fetch 404s).
- Module usage: `grep -rhoE` over `playbooks/` + root env playbooks for FQCN patterns.
- Raw CLI: `grep -rhoE 'aws <svc> <verb>'` over `playbooks/` and `tools/scripts/`.
- Collection coverage verified against `plugins/modules/*.py` in each local collection and each `meta/runtime.yml`.
