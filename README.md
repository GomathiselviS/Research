# Research — AAP SaaS AWS Collection Mapping

Analysis of how the **Ansible-SaaS** (AAP SaaS) repositories use AWS — raw `aws` CLI and Ansible
modules — mapped against the upstream AWS collections
[`amazon.aws`](https://github.com/ansible-collections/amazon.aws),
[`community.aws`](https://github.com/ansible-collections/community.aws), and
[`amazon.cloud`](https://github.com/ansible-collections/amazon.cloud), plus recommendations for what to
add/modify in those collections and in the SaaS repos.

## Documents in this repo

| File | Scope |
|---|---|
| [`sre-ops-aws-collections-comparison.md`](sre-ops-aws-collections-comparison.md) | AWS module/CLI usage in `ansible-saas-sre-operations` vs the three collections |
| [`management-lifecycle-cli-to-collection-mapping.md`](management-lifecycle-cli-to-collection-mapping.md) | Raw CLI → module mapping + gap analysis for `management-lifecycle` |
| [`customer-lifecycle-aws-module-mapping.md`](customer-lifecycle-aws-module-mapping.md) | Module/CLI mapping for `customer-lifecycle-aws` (+ `cloud.aws_ops` / `cloud.aws_troubleshooting` coverage) |
| [`ansible-saas-sops-sre-cli-to-collection-mapping.md`](ansible-saas-sops-sre-cli-to-collection-mapping.md) | `ansible-saas-sops` (docs) + `ansible-saas-sre` mapping and summary tables |
| [`cloud-terraform-alignment-analysis.md`](cloud-terraform-alignment-analysis.md) | **Separate exercise:** `cloud.terraform` alignment for the Terraform (HCL) IaC repos + note on Go repos |

### Modifications requested (`modifications/`)

The actionable change requests — grouped in the [`modifications/`](modifications/) folder:

| File | Scope |
|---|---|
| [`modifications/aws-collections-new-modules.md`](modifications/aws-collections-new-modules.md) | **Consolidated:** new modules to add (primarily `amazon.aws`) |
| [`modifications/aws-collections-module-modifications.md`](modifications/aws-collections-module-modifications.md) | **Consolidated:** modifications to existing `amazon.aws` / `community.aws` / `amazon.cloud` modules |
| [`modifications/ansible-saas-repos-modifications.md`](modifications/ansible-saas-repos-modifications.md) | **Consolidated:** changes to the SaaS repos to use existing modules/playbooks (incl. validated content to adopt now) |
| [`modifications/saas-validated-content-to-develop.md`](modifications/saas-validated-content-to-develop.md) | **New validated content** to build (`cloud.aws_ops` / `redhat.customer_lifecycle_aws` roles) from recurring SaaS use cases |

## Repos analysed (and what each document covers)

| Repo | Type / Language | Role in AAP SaaS | Covered by |
|---|---|---|---|
| [`ansible-saas-sre-operations`](https://github.com/Ansible-SaaS/ansible-saas-sre-operations) | Ansible (Jinja) | Day-2 SRE operations on customer instances | `sre-ops-aws-collections-comparison.md` |
| [`management-lifecycle`](https://github.com/Ansible-SaaS/management-lifecycle) | Ansible + Shell | Management-plane lifecycle (provision/configure/upgrade/deprovision) | `management-lifecycle-cli-to-collection-mapping.md` |
| [`customer-lifecycle-aws`](https://github.com/Ansible-SaaS/customer-lifecycle-aws) | Ansible (Jinja) | Customer instance lifecycle (provision, MRBC/DR, releases) | `customer-lifecycle-aws-module-mapping.md` |
| [`ansible-saas-sops`](https://github.com/Ansible-SaaS/ansible-saas-sops) | Docs (Markdown) | Central SOPs / runbooks / architecture / onboarding | `ansible-saas-sops-sre-cli-to-collection-mapping.md` |
| [`ansible-saas-sre`](https://github.com/Ansible-SaaS/ansible-saas-sre) | Ansible + Python CLI | Core SRE tooling, playbooks, plugins, `ansible-saas` CLI | `ansible-saas-sops-sre-cli-to-collection-mapping.md` |

### Sibling repos — covered by the separate `cloud.terraform` analysis (not AWS-module mapping)

See [`cloud-terraform-alignment-analysis.md`](cloud-terraform-alignment-analysis.md).

| Repo | Type | Disposition |
|---|---|---|
| [`model-aap-deployment`](https://github.com/Ansible-SaaS/model-aap-deployment) | Terraform (HCL) | ROSA-HCP-on-AWS stack → `cloud.terraform` (driven by the Ansible repos) |
| [`management-aap-deployment`](https://github.com/Ansible-SaaS/management-aap-deployment) | Terraform (HCL) | Same — mgmt-plane stack |
| [`model-customer-aws`](https://github.com/Ansible-SaaS/model-customer-aws) + per-instance `cus-*` / `mgt-*` | Terraform (HCL) | Modularized primary/secondary (Aurora Global DR); `cus-*`/`mgt-*` are generated copies |
| [`aws-core-infra-config`](https://github.com/Ansible-SaaS/aws-core-infra-config), [`aoc-obs-iac`](https://github.com/Ansible-SaaS/aoc-obs-iac) | Terraform (HCL) | Shared runner infra / observability stack (S3 backend) |
| `aws-management/customer/audit/identity-account-config` | **Ansible + Cloud Custodian** | **Not Terraform** — belong to the AWS-module mapping (correction of earlier label) |
| [`ansible-saas-management-service`](https://github.com/Ansible-SaaS/ansible-saas-management-service) (+ other `*-service`) | Go | AWS Go SDK — no Ansible/Terraform alignment |

## Headline findings

- **Most `aws` CLI usage is already covered** by existing modules (VPC endpoint CRUD, ELB/RDS/S3 describe, STS, Secrets Manager, WAF web-ACL) and should be migrated for idempotency/check-mode.
- **Recurring true gaps** (no module in any collection), seen across multiple repos:
  1. **VPC Endpoint *Service* (provider-side PrivateLink)** — config, permissions, connection accept/reject.
  2. **Route 53 Resolver DNS Firewall** — rule groups, domain lists, associations.
  3. **RDS Aurora Global Cluster failover / switchover** — imperative ops (`amazon.cloud.rds_global_cluster` is declarative-only).
  4. **AWS Account management** and **IAM OIDC identity provider**.
- **`secretsmanager_secret` FQCN inconsistency** across every repo (module lives in `community.aws`; `amazon.aws` ships only a lookup + a dangling redirect).
- **`amazon.cloud` is unused** by all repos, yet is the only collection with `wafv2_logging_configuration`.

> Note: `Research` is a private repo — the SaaS repos are internal Red Hat.
