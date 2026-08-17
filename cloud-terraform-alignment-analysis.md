# `cloud.terraform` Alignment Analysis — AAP SaaS Terraform & Go Repos

Separate companion to the AWS-module mapping documents. Those docs cover how the **Ansible** SaaS repos
call AWS (raw `aws` CLI + `amazon.aws` / `community.aws` / `amazon.cloud` modules). This document covers the
repos that were previously **out of scope** for AWS-module mapping — the **Terraform (HCL)** IaC repos and
the **Go** service repos — and analyses their alignment with the
[`cloud.terraform`](https://github.com/ansible-collections/cloud.terraform) collection (v5.0.0-dev0).

> `Research` is a private repo; the Ansible-SaaS source repos are internal Red Hat. Do not republish.

## `cloud.terraform` collection contents (v5.0.0-dev0)

| Type | Name | Purpose |
|---|---|---|
| Module | `terraform` | Run `init` / `plan` / `apply` / `destroy` (`state: present\|absent\|planned`) on a root module |
| Module | `terraform_output` | Read outputs (`terraform output -json`) into Ansible vars |
| Module | `plan_stash` | base64 stash/load a plan file across plays (`state: stash\|load`) |
| Inventory | `terraform_state` | Build inventory from resources in TF state (any backend) |
| Inventory | `terraform_provider` | Build inventory from a Terraform provider's data |
| Role | `git_plan` | Run a plan from a git-hosted root module |
| Role | `inventory_from_outputs` | Construct inventory from TF outputs |

`terraform` module option surface today: `project_path`, `binary_path`, `workspace`, `purge_workspace`,
`plan_file`, `state_file`, `variables` / `complex_vars` / `variables_files`, `targets`, `lock` /
`lock_timeout`, `force_init` / `overwrite_init`, `backend_config` / `backend_config_files`,
`provider_upgrade`, `check_destroy`, `parallelism`. **States:** `present`, `absent`, `planned` only.

---

## Part 1 — Terraform (HCL) repos: what they provision

All the HCL stacks are **ROSA HCP (managed OpenShift on AWS) deployment stacks**: they compose the public
`terraform-redhat/rosa-hcp/rhcs` modules with `hashicorp/aws` to stand up a cluster plus its supporting AWS
footprint (VPC, IAM/OIDC, KMS, S3, RDS/Aurora, Secrets Manager, ALB/WAF, VPC flow logs → Splunk/Kinesis).

| Repo | Kind | Providers | Notable AWS resources | Backend |
|---|---|---|---|---|
| `model-aap-deployment` | Flat single stack (model/customer plane) | `aws`, `rhcs`, `null` | IAM (role/policy/attach ×7–8), KMS ×6, VPC+subnets+NAT+IGW+routes, S3 (+repl/lifecycle/versioning), **`vpc_endpoint_service`**, `db_instance` (single RDS), `secretsmanager_secret`, **`wafv2_web_acl` + `wafv2_web_acl_logging_configuration` + `wafv2_ip_set`**, `flow_log` + `kinesis_firehose_delivery_stream`, `acm_certificate` | none in HCL |
| `model-customer-aws` | Modularized, primary/secondary (DR) | `aws` ×4, `rhcs` ×2, `null` | Same base **plus `rds_global_cluster` + `rds_cluster` + `rds_cluster_instance` (Aurora Global for MRBC/DR)** and dedicated `aap_storage` / `aap_storage_replication` modules (cross-region S3 repl) | none in HCL |
| `management-aap-deployment` | Flat single stack (mgmt plane) | `aws` `>=5.85,<6.45`, `rhcs`, `null` | Same base **plus `sqs_queue`**; single `db_instance`; `vpc_endpoint_service`; WAF + WAF logging; flow logs → Kinesis | none in HCL |
| `aws-core-infra-config` | Shared infra | `aws`, `null` | VPC + **`autoscaling_group` + `launch_template`** (self-hosted GitHub Actions runner fleet), IAM instance profile | **S3** (`ansible-saas-tfstates`) |
| `aoc-obs-iac` | Observability platform | `aws`, `rhcs`, `null`, `random`, `time` | ROSA HCP + VPC + KMS ×2 + S3 (state/observability) + IAM/OIDC | **S3** (commented out) |
| `cus-*` / `mgt-*` (per-instance) | Generated copies of the model stacks | — | Deduplicated — same shape as `model-*`, one per deployed instance | per-instance |

**Cross-confirmation with the AWS-module docs:** the HCL stacks provision, at **create** time, exactly the
resources whose **day-2** operations the Ansible repos cannot yet express as modules:

| Resource in Terraform | Day-2 gap flagged in AWS-module docs |
|---|---|
| `aws_vpc_endpoint_service` (in 3 stacks) | **VPC Endpoint *Service* / provider-side PrivateLink** — top recurring gap (`aws-collections-new-modules.md`) |
| `aws_rds_global_cluster` + `aws_rds_cluster` | **Aurora Global Cluster failover/switchover** (`aws-collections-module-modifications.md`) |
| `aws_wafv2_web_acl_logging_configuration` | WAF logging — only in `amazon.cloud` today |
| `aws_secretsmanager_secret` (+ `_version`) | `secretsmanager_secret` FQCN/packaging inconsistency |

So the create-path (Terraform) and the operate-path (Ansible) are split across the same resource set — the
collection gaps are what force the Ansible side back to raw `aws` CLI for day-2.

---

## Part 2 — Account-config repos are NOT Terraform

Four of the "aws-*-account-config" repos have **no `.tf` files**. They configure AWS Organizations member
accounts with **Ansible + Cloud Custodian + GitHub Actions**, not Terraform:

| Repo | Mechanism | Content |
|---|---|---|
| `aws-management-account-config` | Ansible + Cloud Custodian | `configure_management_account.yml`, `cloud-custodian/mgt-required-tags.yml` |
| `aws-customer-account-config` | Ansible + Cloud Custodian | `configure_customer_account.yml`, `cus-required-tags.yml`, `aws-quota-report.yml` |
| `aws-audit-account-config` | Ansible | `configure_audit_account.yml` |
| `aws-identity-account-config` | Ansible | `configure_identity_account.yml` (+ IT-cloud variant) |

**Implication:** these belong to the **AWS-module mapping** exercise, not `cloud.terraform`. Their
`configure_*_account.yml` playbooks are candidates for the same `amazon.aws` / `community.aws` modules and
gaps identified in the other docs (Account management, IAM, tagging). They are called out here only to
correct the earlier "Terraform, out of scope" label — the label was accurate for `model-*` /
`management-aap-deployment` but wrong for these four.

---

## Part 3 — Go repos

`ansible-saas-management-service` and sibling `*-service` repos are Go services using the **AWS Go SDK**
directly (control-plane APIs, reconciliation, orchestration). They are **not** Ansible or Terraform and are
**not** a `cloud.terraform` target. No alignment action; documented for completeness.

---

## Part 4 — How the Ansible repos already drive Terraform

The Ansible repos already depend on `cloud.terraform` **and** shell out to the raw `terraform` binary for
operations the module doesn't cover:

| Repo | `cloud.terraform.terraform` uses | Raw `terraform` shell-outs (module-relevant) |
|---|---|---|
| `customer-lifecycle-aws` | 8 | `state push errored.tfstate` ×4, `show -json` ×4, `init` ×2, `untaint …rosa_hcp_cluster`, `state rm`, `refresh`, `output -json` |
| `ansible-saas-sre` | 7 | `refresh` ×4, `init` ×4, `show -json` ×3, `state rm` ×2, `state push errored.tfstate` ×2, `untaint …` |
| `management-lifecycle` | 5 | `show -json` ×3, `state rm` ×2, `state push errored.tfstate` ×2, `refresh` ×2, `init` ×2, `untaint …` |
| `ansible-saas-sre-operations` | 5 | `refresh` ×2, `plan -no-color -input=false` ×2, `init -input=false` ×2, `show -json` |

Every repo uses **`cloud.terraform.terraform`** for the core init/plan/apply/destroy path, but **none** uses
`terraform_output`, `plan_stash`, or the `terraform_state` / `terraform_provider` inventory plugins — even
though they clearly need them (`terraform output -json`, saving/restoring `errored.tfstate`, reading state).

### Day-1 vs Day-2 — Terraform is used for both

The Terraform invocations are **not** limited to initial provisioning. A majority are **day-2 operations**
against already-running instances. Verified from the task flow inside each playbook (not just filenames):

| Class | Playbook(s) | What Terraform does |
|---|---|---|
| **Day-1 provision** | `create_instance_infrastructure.yml`, `create_storage_infrastructure.yml` | Initial `init` → `apply` of the ROSA-HCP-on-AWS stack |
| **Day-2 · IP allowlist reconfig** | `configure_aap_ip_allowlist.yml` (`_legacy`) — sre, sre-operations | Rewrites `terraform.tfvars` (`AAP_IP_ALLOWLIST`, `enable_aap_restricted_access`), then `init` → `refresh` → `apply` to push WAF/ALB rule changes to a live instance |
| **Day-2 · DR restore/recovery** | `restore_recover_ansible_platform.yml` — sre | Updates S3 bucket + KMS key policies for cross-region use, creates recovery job, `init` → `refresh` to reconcile the recovered instance |
| **Day-2 · drift/upgrade reconcile** | `update_terraform.yml` — management-lifecycle, sre, sre-operations | `init` → `refresh` (sync state to real AWS) → `apply` → `show -json` to re-read state |
| **Day-2 · deprovision/teardown** | `deprovision_management_instance.yml`, `destroy_instance_infrastructure.yml`, `destroy_storage_infrastructure.yml` | `state: absent` full-stack destroy |
| **Day-2 · errored-state recovery** | across customer-lifecycle, sre, management-lifecycle | `state push errored.tfstate`, `state rm`, `untaint …rosa_hcp_cluster` after a failed apply |

### Which day-2 operations can be converted from Terraform → Ansible AWS module calls

Some of these day-2 flows re-run a whole-stack `terraform apply` only to change a **bounded set of AWS
resources** that already have first-class Ansible modules. Those are candidates to bypass Terraform and call
the AWS module directly. Others touch the ROSA HCP cluster (`rhcs` provider) or the TF state file itself and
**must** stay on Terraform. Module existence verified against the local collections.

| Day-2 operation | Underlying AWS change | Convertible? | Direct module(s) |
|---|---|---|---|
| **IP allowlist reconfig** | Add/remove IPs in a WAF IP set; attach/detach Allow/Block rules on the Web ACL | ✅ **Yes** | `community.aws.wafv2_ip_set`, `community.aws.wafv2_web_acl` |
| **DR restore — policy prep** | S3 bucket policy + KMS key policy edits for cross-region recovery | ✅ **Yes** (playbook already partly does this) | `amazon.aws.s3_bucket` (`policy`), `amazon.aws.kms_key` (`policy`) |
| **DR restore — Aurora failover/switchover** | Promote secondary / switch Aurora Global Cluster primary | ⚠️ **Gap** | `amazon.cloud.rds_global_cluster` is **declarative-only** — no imperative failover (see `aws-collections-module-modifications.md`) |
| **WAF logging reconfig** | Enable/disable Web ACL logging | ✅ **Yes** | `amazon.cloud.wafv2_logging_configuration` (only in `amazon.cloud`, currently unused) |
| **Drift/upgrade reconcile** (`update_terraform.yml`) | Whole-stack reconcile incl. ROSA HCP cluster | ❌ **No** | Multi-resource declarative reconcile + `rhcs` provider — Terraform's job; keep on `cloud.terraform.terraform` |
| **Deprovision/teardown** | Whole-stack destroy incl. ROSA HCP cluster | ❌ **No** | Same — keep as `state: absent` |
| **Errored-state recovery / taint / state rm** | Manipulates the Terraform **state file** | ❌ **No** (not an AWS-resource op) | Terraform-state plumbing — close via the `cloud.terraform` additions in Part 5A |

**Drift caveat:** the ✅ rows manage resources that Terraform also owns. Making the change out-of-band with an
Ansible module introduces **state drift** — the next `terraform apply` reverts it unless `terraform.tfvars`
is kept in sync (which the allowlist playbook already does today). So a clean conversion means either (a)
**move that resource out of Terraform management** and own it in Ansible, or (b) treat the module call as a
fast-path and still write the tfvars so the two agree. Whole-stack reconcile, cluster lifecycle, and
state-file ops (❌) should stay on Terraform regardless.

---

## Part 5 — Recommendations

### A. Additions to `cloud.terraform` (close the shell-out gaps)

These are the raw `terraform` subcommands the SaaS repos shell out to because no module/option exists:

| Priority | Proposed capability | Target | Replaces raw CLI | Repos |
|---|---|---|---|---|
| P1 | **`terraform_state` module** (imperative `state rm` / `state mv` / `state pull` / `state push`) — distinct from the existing *inventory* plugin of the same name | `cloud.terraform` | `terraform state rm`, `terraform state push errored.tfstate` | customer-lifecycle, sre, management-lifecycle |
| P1 | **`taint` / `untaint`** support (new module `terraform_taint`, or a `taint:` / `untaint:` option on `terraform`) | `cloud.terraform` | `terraform untaint module.…rosa_hcp_cluster` | all four |
| P1 | **Errored-state recovery** in `terraform` (auto-detect/emit `errored.tfstate`, and a `recover_state`/`state_push` path) | `cloud.terraform` | manual `terraform state push errored.tfstate` after failed apply | customer-lifecycle, sre, management-lifecycle |
| P2 | **`refresh`-only mode** (`state: refreshed`, i.e. `apply -refresh-only`) | `cloud.terraform` | `terraform refresh` | all four |
| P2 | **State read** — return parsed state (`terraform show -json`) from `terraform`, or a `terraform_state_info` module | `cloud.terraform` | `terraform show -json` | all four |
| P3 | **`import`** support (module or option) | `cloud.terraform` | (implied by state-manipulation workflows) | — |

### B. Adoptions in the SaaS repos (use what already exists)

| Change | Repos | Adopt |
|---|---|---|
| Replace `terraform output -json` + `terraform show -json`-for-outputs with the module | customer-lifecycle, all | `cloud.terraform.terraform_output` |
| Replace bespoke `errored.tfstate` base64/save-restore glue with the stash module | customer-lifecycle, sre, management-lifecycle | `cloud.terraform.plan_stash` |
| Build host/resource inventory from TF state instead of parsing `show -json` | sre, sre-operations | `cloud.terraform.terraform_state` inventory plugin |
| Keep `init`/`plan`/`apply`/`destroy` on the module (already done) — extend `backend_config` use for the S3 backends | all | `cloud.terraform.terraform` |
| Pin `cloud.terraform` in every `requirements.yml` consistently | all | — |

### C. Boundary note (create vs. operate)

`cloud.terraform` **wraps** the HCL stacks (`model-*`, `management-aap-deployment`) — it does not replace or
consume the HCL. The alignment work is entirely on the **Ansible driver side**: make the collection able to
express the full lifecycle (state ops, taint, errored-state recovery, refresh, state read) so the SaaS
playbooks stop shelling out. The HCL itself stays as-is; the only HCL-adjacent recommendation is to give
`model-*` / `management-aap-deployment` an explicit **S3 backend** block (as `aws-core-infra-config` already
has) rather than leaving state backend undeclared.

---

## Summary

- **Terraform-in-scope:** `model-aap-deployment`, `model-customer-aws`, `management-aap-deployment`,
  `aws-core-infra-config`, `aoc-obs-iac` (+ generated `cus-*`/`mgt-*`). All ROSA-HCP-on-AWS stacks.
- **Mislabeled earlier:** the four `aws-*-account-config` repos are **Ansible + Cloud Custodian**, not
  Terraform → they belong to the AWS-module mapping, not this doc.
- **Go repos:** AWS Go SDK, no Ansible/Terraform alignment.
- **Biggest `cloud.terraform` gaps:** imperative **state manipulation**, **taint/untaint**, **errored-state
  recovery**, **refresh-only**, and **state read** — every SaaS repo shells out to the `terraform` binary for
  these today.
- **Quick wins (no upstream work):** adopt `terraform_output`, `plan_stash`, and the `terraform_state`
  inventory plugin, which already exist and are currently unused.
- **Day-2 is real:** most Terraform invocations are day-2 (allowlist reconfig, DR restore, drift/upgrade
  reconcile, deprovision, errored-state recovery), not just day-1 provisioning.
- **Convertible day-2 ops:** IP-allowlist changes (`community.aws.wafv2_ip_set`/`wafv2_web_acl`), DR policy
  prep (`amazon.aws.s3_bucket`/`kms_key`), and WAF logging (`amazon.cloud.wafv2_logging_configuration`) can
  move off Terraform to direct module calls — subject to the tfvars drift caveat. Whole-stack reconcile,
  cluster lifecycle, Aurora failover (gap), and state-file ops must stay on Terraform.
- **Strategic overlap:** the HCL stacks create `vpc_endpoint_service`, `rds_global_cluster`, and
  `wafv2_web_acl_logging_configuration` — the same resources whose day-2 operations are the top AWS-module
  gaps. Closing both sides removes the raw-CLI/raw-terraform debt end to end.
