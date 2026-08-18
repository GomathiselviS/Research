# SRE Operations Repo vs. AWS Collections — Comparison & Findings

**Date:** 2026-08-17
**SRE repo analyzed:** [`Ansible-SaaS/ansible-saas-sre-operations`](https://github.com/Ansible-SaaS/ansible-saas-sre-operations) (internal, `main`)
**Local collections compared:**
- `/home/gosriniv/github/collections/ansible_collections/amazon/aws`
- `/home/gosriniv/github/collections/ansible_collections/community/aws`
- `/home/gosriniv/github/collections/ansible_collections/amazon/cloud`

---

## 1. Executive Summary

- The SRE repo is a Red Hat **internal Ansible collection** (`redhat.ansible_saas_sre_operations`, v1.0.0, GPL-2.0-or-later) of SRE playbooks operating the **Ansible Automation Platform (AAP) SaaS** on ROSA / Kubernetes / AWS.
- Its actual AWS module footprint is **small** — mostly `ansible.builtin` + `kubernetes.core` glue, with **15 distinct AWS module references**.
- **~13 of the 15 AWS module calls resolve cleanly** against the local `amazon.aws` + `community.aws`.
- **One real mismatch:** `amazon.aws.secretsmanager_secret` (used 23×) is **not a module** in these checkouts — the module lives only in `community.aws`. Those task calls would fail to resolve.
- **`amazon.cloud` is present but completely unused** — the main opportunity, chiefly to replace `aws wafv2` and PrivateLink CLI shell-outs with real modules.

---

## 2. Collections Inventory

| Collection | Local version | # modules | Used by SRE repo? |
|---|---|---|---|
| `amazon.aws` | 12.0.0-dev0 (`main`) | 141 | ✅ heavily |
| `community.aws` | 12.0.0-dev0 | 138 | ✅ lightly |
| `amazon.cloud` | 0.4.0 | 63 | ❌ not at all |

> Note: local `amazon.aws` / `community.aws` are **development checkouts** (`12.0.0-dev0`). Published Galaxy releases the SRE repo installs at runtime may differ. The SRE repo's `requirements.yml` lists `amazon.aws`, `community.aws` with **no version pin** (only `ansible.platform` is pinned).

---

## 3. AWS Modules the SRE Repo Actually Calls

### From `amazon.aws` — all present locally ✅ **except one**

| Module | Uses | Present locally? |
|---|---|---|
| `amazon.aws.sts_assume_role` | 35 | ✅ |
| `amazon.aws.secretsmanager_secret` | 23 | ❌ **not a module here** (see §4) |
| `amazon.aws.s3_object` | 10 | ✅ |
| `amazon.aws.s3_object_info` | 2 | ✅ |
| `amazon.aws.kms_key_info` | 2 | ✅ |
| `amazon.aws.s3_bucket` | 1 | ✅ |
| `amazon.aws.s3_bucket_info` | 1 | ✅ |
| `amazon.aws.kms_key` | 1 | ✅ |
| `amazon.aws.ec2_vpc_net_info` | 1 | ✅ |
| `amazon.aws.ec2_vpc_subnet_info` | 1 | ✅ |
| `amazon.aws.ec2_tag` | 1 | ✅ |
| `amazon.aws.ec2_tag_info` | 1 | ✅ |
| `amazon.aws.ec2_security_group` | 1 | ✅ |

### From `community.aws` — both present ✅

| Module | Uses | Present locally? |
|---|---|---|
| `community.aws.secretsmanager_secret` | 14 | ✅ |
| `community.aws.acm_certificate` | 2 | ✅ |

### From `amazon.cloud`

| Module | Uses |
|---|---|
| *(none)* | 0 |

---

## 4. ⚠️ Key Finding — `secretsmanager_secret` FQCN Mismatch

The SRE repo calls the secret module **two different ways for the same thing**:

- `amazon.aws.secretsmanager_secret` — **23×**
- `community.aws.secretsmanager_secret` — **14×**

**In these local checkouts:**
- The **module** `secretsmanager_secret.py` exists **only in `community.aws`** (`community/aws/plugins/modules/secretsmanager_secret.py`).
- `amazon.aws` ships only a **lookup plugin** named `secretsmanager_secret` (`amazon/aws/plugins/lookup/secretsmanager_secret.py`) — **there is no `amazon.aws.secretsmanager_secret` module.**
- `amazon.aws/meta/runtime.yml` has an `aws_secret` → `amazon.aws.secretsmanager_secret` redirect (i.e. amazon.aws *intends* to host it), but the module file is **not present** in this `12.0.0-dev` checkout.

**Impact:** the 23 `amazon.aws.secretsmanager_secret` task calls would **fail to resolve as a module** against these collections. The repo is internally inconsistent.

**Recommendation:**
- Standardize on `community.aws.secretsmanager_secret` (the FQCN that resolves today), **or**
- Pin a published `amazon.aws` version that actually ships the `secretsmanager_secret` module, and migrate all calls to it consistently.

---

## 5. Raw AWS CLI Shell-Outs — Module Replacement Opportunities

The SRE repo shells out to the `aws` CLI in several places where a module could be used. Some already-used or already-available modules could replace them:

| Raw CLI in SRE repo | Uses | Existing module available? |
|---|---|---|
| `aws secretsmanager describe-secret` | 5 | Partly — `community.aws.secretsmanager_secret` (+ `*_info` patterns) |
| `aws s3 cp` | 4 | ✅ `amazon.aws.s3_object` (already used elsewhere in repo) |
| `aws ec2 create-vpc-endpoint` | 2 | ✅ `community.aws.ec2_vpc_endpoint` (present locally) |
| `aws secretsmanager replicate-secret-to-regions` | 1 | Partly — `community.aws.secretsmanager_secret` (`replica` param) |
| `aws ec2 modify-vpc-endpoint-service-permissions` | 1 | ❌ no dedicated module (endpoint **service** mgmt) — CLI justified |
| `aws ec2 modify-vpc-endpoint-service-configuration` | 1 | ❌ no dedicated module — CLI justified |
| `aws ec2 describe-vpc-endpoint-service-configurations` | 1 | ❌ no dedicated module — CLI justified |
| `aws ec2 describe-vpc-endpoints` | 1 | ✅ `community.aws.ec2_vpc_endpoint_info` |
| `aws wafv2 get-web-acl` | 1 | ⚠️ not in amazon.aws/community.aws — **see `amazon.cloud` (§6)** |
| `aws wafv2 list-web-acls` | 1 | ⚠️ not in amazon.aws/community.aws — **see `amazon.cloud` (§6)** |
| `aws wafv2 update-web-acl` | 1 | ⚠️ not in amazon.aws/community.aws — **see `amazon.cloud` (§6)** |

---

## 6. `amazon.cloud` — Present but Unused (Main Opportunity)

The SRE repo uses **zero** `amazon.cloud` modules. It is a CloudFormation-registry-backed collection (63 modules) that overlaps several SRE needs and could replace CLI shell-outs.

**Most relevant candidates:**
- **WAF:** `amazon.cloud.wafv2_web_acl_association`, `wafv2_ip_set`, `wafv2_regex_pattern_set`, `wafv2_logging_configuration` — the SRE repo currently does **all WAF work via `aws wafv2` CLI** (`delete_aap_waf_dynamic_rules.yml`, IP allowlist tasks). This is the clearest win.
- **Storage / keys:** `s3_bucket`, `kms_alias`, `kms_replica_key`.
- **IAM / compute:** `iam_role`, `iam_instance_profile`, `eks_cluster`, `eks_addon`, `eks_fargate_profile`.
- **Data:** `rds_db_instance`, `rds_db_cluster_parameter_group`, `redshift_*`, `memorydb_*`.
- **Networking / DNS:** `route53_dnssec`, `route53_key_signing_key`.

> Caveat: `amazon.cloud` is `0.4.0` (early). Modules are CFN-registry-typed and behave differently from `amazon.aws` (create/update/delete via desired-state). Validate idempotency and check-mode behavior before adopting in production SRE flows.

---

## 7. Full Module-Usage Breakdown (all namespaces)

For context, the SRE repo's overall module usage (top items):

```
273  ansible.builtin.set_fact
111  ansible.builtin.env
 99  ansible.builtin.import_tasks
 80  ansible.builtin.assert
 70  ansible.builtin.include_tasks
 54  ansible.builtin.command
 49  ansible.builtin.uri
 46  ansible.builtin.debug
 35  amazon.aws.sts_assume_role
 33  ansible.builtin.file
 24  ansible.builtin.lineinfile
 23  amazon.aws.secretsmanager_secret     <-- FQCN mismatch (see §4)
 20  kubernetes.core.k8s_info
 ...
 14  community.aws.secretsmanager_secret
 10  amazon.aws.s3_object
  9  kubernetes.core.k8s
  5  cloud.terraform.terraform
  4  community.postgresql.postgresql_db
  2  community.aws.acm_certificate
  2  ansible.controller.job_template
  ...
```

Other dependency collections in play: `kubernetes.core`, `cloud.terraform`, `community.postgresql`, `community.general`, `ansible.controller`, `ansible.platform`, plus an internal `redhat.customer_lifecycle_aws`.

---

## 8. Recommendations (Prioritized)

1. **Fix the `secretsmanager_secret` FQCN inconsistency** (correctness blocker). Pick one resolvable FQCN and apply it repo-wide; add a version pin so the module actually resolves.
2. **Pin AWS collection versions** in `requirements.yml` (`amazon.aws`, `community.aws`, and — if adopted — `amazon.cloud`). Currently unpinned → drift risk exactly like the secretsmanager case.
3. **Replace low-risk CLI shell-outs with existing modules** where a module is already used/available: `aws s3 cp` → `amazon.aws.s3_object`; `aws ec2 describe-vpc-endpoints` → `community.aws.ec2_vpc_endpoint_info`; `aws ec2 create-vpc-endpoint` → `community.aws.ec2_vpc_endpoint`.
4. **Evaluate `amazon.cloud` for WAF tasks** — the largest cluster of CLI shell-outs with no `amazon.aws`/`community.aws` module equivalent.
5. **Leave genuinely uncovered CLI as-is** — VPC endpoint *service* permission/configuration management has no module; document why the CLI is used.

---

## Appendix — How this was gathered

- Repo metadata + file tree via `gh api` (repo is `internal`/private; public WebFetch returns 404).
- SRE repo cloned shallow via `git clone` with `gh auth token`.
- Module usage extracted with `grep -rhoE` over `playbooks/` for FQCN patterns and raw `aws` CLI invocations.
- Local collection inventories built from `plugins/modules/*.py`; redirects verified in each collection's `meta/runtime.yml`.
