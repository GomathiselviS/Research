# customer-lifecycle-aws → Ansible AWS Collections Module Mapping

Analysis of [`Ansible-SaaS/customer-lifecycle-aws`](https://github.com/Ansible-SaaS/customer-lifecycle-aws)
mapping the modules and raw `aws` CLI calls used in its playbooks/tasks to modules available in
[`amazon.aws`](https://github.com/ansible-collections/amazon.aws),
[`amazon.cloud`](https://github.com/ansible-collections/amazon.cloud), and
[`community.aws`](https://github.com/ansible-collections/community.aws).

_Scope: all 103 YAML files (playbooks + `playbooks/tasks/*`) were fetched and scanned._

## Overview

The repo is SaaS instance-lifecycle automation. AWS interaction falls into two categories:

1. **Native Ansible AWS modules** (already using FQCNs).
2. **Raw `aws` CLI calls** wrapped in `ansible.builtin.command` / `ansible.builtin.shell` — the prime
   candidates for mapping to real modules.

## AWS modules already in use

| Module (FQCN) | Uses | Collection |
|---|---|---|
| `amazon.aws.sts_assume_role` | 41 | amazon.aws |
| `community.aws.secretsmanager_secret` | 23 | community.aws |
| `amazon.aws.s3_object` | 10 | amazon.aws |
| `amazon.aws.kms_key` | 8 | amazon.aws |
| `amazon.aws.route53` | 5 | amazon.aws |
| `amazon.aws.aws_caller_info` | 5 | amazon.aws |
| `amazon.aws.elb_application_lb` / `elb_application_lb_info` | 3 + 3 | amazon.aws |
| `amazon.aws.ec2_vpc_net_info` | 3 | amazon.aws |
| `amazon.aws.ec2_vpc_endpoint_service_info` | 3 | amazon.aws |
| `amazon.aws.s3_object_info` / `kms_key_info` | 2 + 2 | amazon.aws |
| `community.aws.route53_wait` | 2 | community.aws |
| `community.aws.acm_certificate_info` | 2 | community.aws |
| `amazon.aws.route53_info`, `ec2_vpc_endpoint`, `ec2_vpc_endpoint_info`, `ec2_tag`, `ec2_security_group`, `ec2_security_group_info` | 1 each | amazon.aws |

**Non-AWS content used:** `kubernetes.core.k8s` / `k8s_info` / `k8s_cp`, `cloud.terraform.terraform`,
`community.postgresql.*`, `community.crypto.openssh_keypair`, `community.general.slack` / `archive`,
`ansible.controller.license`, plus heavy `ansible.builtin.*`.

**External CLIs shelled out** (not AWS): `terraform`, `rosa`, `oc` / `kubectl`, `flux`, `git`, `gh`, `psql`.

## Raw `aws` CLI calls → module mapping

| `aws` CLI call | Mappable? | Best target module |
|---|---|---|
| `aws rds describe-db-clusters` | ✅ Yes | `amazon.aws.rds_cluster_info` |
| `aws rds describe-global-clusters` | ✅ Yes | `amazon.aws.rds_global_cluster_info` |
| `aws s3api head-bucket` | ✅ Yes | `amazon.aws.s3_bucket_info` |
| `aws s3api head-object` | ✅ Yes | `amazon.aws.s3_object_info` |
| `aws elbv2 describe-load-balancers` | ✅ Yes | `amazon.aws.elb_application_lb_info` |
| `aws elbv2 add-tags` | ⚠️ Partial | `amazon.aws.elb_application_lb` (declarative `tags:`; no standalone ELB-tag module) |
| `aws secretsmanager describe-secret` / `get-secret-value` | ✅ Yes | `community.aws.secretsmanager_secret` (+ `amazon.aws.aws_secret` lookup for values) |
| `aws secretsmanager delete-secret` | ✅ Yes | `community.aws.secretsmanager_secret` (`state: absent`) |
| `aws secretsmanager replicate-secret-to-regions` / `remove-regions-from-replication` | ⚠️ Partial | `community.aws.secretsmanager_secret` has a declarative `replica:` region-list param; imperative `stop-replication-to-replica` (promote replica) has no equivalent |
| `aws wafv2 disassociate-web-acl` | ✅ Yes | `amazon.cloud.wafv2_web_acl_association` (`state: absent`); or `community.aws.wafv2_resources` |
| `aws rds failover-global-cluster` / `switchover-global-cluster` / `remove-from-global-cluster` | ❌ No | `amazon.cloud.rds_global_cluster` is declarative (Cloud Control) create/update only — no failover/switchover/detach operations. **Gap.** |
| `aws ec2 describe/delete/reject vpc-endpoint-service-configurations`, `describe-vpc-endpoint-connections` | ❌ No | Only consumer-side modules exist (`ec2_vpc_endpoint`, `ec2_vpc_endpoint_info`, `ec2_vpc_endpoint_service_info`); no provider-side PrivateLink endpoint-*service* management. **Gap.** |
| `aws route53resolver associate/disassociate/list-firewall-rule-group(-associations)` | ❌ No | No Route 53 Resolver / DNS Firewall module in any of the three collections. (`community.aws.networkfirewall*` is AWS **Network** Firewall — a different service.) **Gap.** |

## Summary

- **Fully mappable today:** RDS *info*, S3 *info*, ELB describe, and most Secrets Manager CLI calls —
  these `command`/`shell` tasks could be replaced with modules the repo already uses elsewhere.
- **Partially mappable:** ELB tagging and Secrets Manager cross-region replication (declarative module
  params cover the common case, not imperative operations like promoting a replica).
- **True gaps (no module in any of the three collections):**
  1. **RDS Aurora Global Cluster failover / switchover / detach** — imperative ops; `amazon.cloud.rds_global_cluster` is declarative-only.
  2. **VPC Endpoint Service (provider-side PrivateLink) configuration + connection accept/reject.**
  3. **Route 53 Resolver DNS Firewall rule-group associations.**

These three gaps are exactly the CLI calls that must remain `aws` CLI shell-outs, and are the strongest
candidates for new modules to contribute upstream to `amazon.aws` / `community.aws`.

## Use cases captured in this repo

The repo automates the full lifecycle of SaaS customer deployments (Ansible Automation Platform as a
Service) on AWS/ROSA. Derived from the 24 top-level playbooks and their supporting tasks:

### 1. Instance provisioning & lifecycle
- **Provision primary/active instance** (`create_primary_instance.yml`) — stand up a new active
  customer instance on AWS, optionally with a custom FQDN / Route 53 hosted zone.
- **Provision secondary/passive instance** (`create_secondary_instance.yml`) — stand up a passive
  standby instance for DR (cross-region).
- **Delete active instance** (`delete_active_instance.yml`) and **delete passive instance**
  (`delete_passive_instance.yml`) — decommission instances and their DNS.
- **Create / delete storage infrastructure** (`delete_storage_infrastructure.yml`, plus
  `create_storage_infrastructure` / `destroy_storage_infrastructure` tasks) — manage KMS keys, S3, and
  Aurora/RDS storage backing an instance.

### 2. Customer configuration (post-deployment)
- **Apply post-deployment customer configuration** (`apply_customer_configuration.yml`) — configure the
  AAP instance after provisioning, validate instance naming (`cus-`/`inst-`), and report status to the
  inventory service.

### 3. Release management
- **Create a customer release** (`create_customer_release.yml`) — generate a versioned SaaS release
  (`cus-` date/time naming), version all component repos, build a manifest.
- **Promote release to production/pre-production** (`create_customer_production_release.yml`) — cut a
  GitHub release from the manifest.
- **Cherry-pick to stable branch** (`cherry_pick_to_stable.yml`) — port commits from main to a stable
  branch via git CLI.

### 4. MRBC — Multi-Region Backup & Continuity (DR / failover)
- **Failover deployment** (`failover_deployment.yml`) — planned switchover (zero data loss) or
  unplanned failover from active to passive, with optional Dynatrace maintenance-window hooks.
- **MRBC migration / DB restore** (`mrbc_migration.yml`) — restore the AAP database onto an MRBC
  instance from backup.
- **Refresh MRBC Aurora writer region** (`refresh_mrbc_aurora_writer.yml`) — detect the live Aurora
  writer region and push it as a Dynatrace event.

### 5. Backup
- **Backup Ansible Platform** (`backup_ansible_platform.yml`) — back up AAP-as-a-Service on AWS
  (flux config, environment values, cluster resources).

### 6. Licensing
- **Add license manifest** (`add_license_manifest.yml`) — upload a subscription/license manifest to an
  instance.

### 7. Notifications
- **Send customer notification** (`send_customer_notification.yml`) — cluster notifications to
  customers (secrets read via assumed AWS role).
- **Send message to Slack** (`send_message_to_slack.yml`) — post subscription/inventory status to Slack.

### 8. Inventory & subscription synchronization
- **Sync inventories status** (`sync_inventories_status.yml`) — reconcile "provisioning" inventory
  records against actual job status / GitHub project state.
- **Sync delete failed clusters** (`sync_delete_failed_clusters.yml`) — clean up clusters that failed
  provisioning.
- **Sync unsubscribed subscriptions** (`sync_unsubscribed_subscriptions.yml`) — check Red Hat
  Marketplace subscription status and delete subscriptions when due.

### 9. Capacity / pool management
- **Apply pool service** (`apply_pool_service.yml`) — call a pool-service API to reconcile the desired
  vs. effective number of warm/available instances.

### 10. Test scaffolding (MRBC)
- **Dummy provision / deprovision / apply-config** (`dummy_provision_customer_instance.yml`,
  `dummy_deprovision_customer_instance.yml`, `dummy_apply_customer_configuration.yml`) — no-op
  playbooks that skip real infrastructure work and only report success/failure to the inventory
  service, used for MRBC testing.

### Supporting task-level capabilities (`playbooks/tasks/`)
Cross-cutting building blocks reused by the above: ROSA/OpenShift cluster admin user management,
KMS key + secret creation and replication, operator secret replication, ACM/Route 53 DNS record and
certificate validation, ALB ingress + Route 53 wiring, VPC PrivateLink endpoint setup/teardown,
shared DNS firewall association, Aurora writer region discovery/failover, Dynatrace event +
maintenance-window management, deployment-fork git operations, Terraform state load/apply, and
inventory-service status reporting.

## Coverage in `cloud.aws_ops` and `cloud.aws_troubleshooting`

Both are Red Hat validated-content collections of general-purpose AWS **operations** (`aws_ops`) and
**diagnostics** (`aws_troubleshooting`) roles. They sit at the low-level AWS infrastructure layer,
while `customer-lifecycle-aws` is a SaaS/AAP **lifecycle orchestration** layer (ROSA/OpenShift, AAP
config, inventory service, releases, MRBC). Below: what is covered, how covered items can be tightened
to map exactly, and what is left out.

### A. Use cases that ARE covered (and how to map them exactly)

#### 1. MRBC — Aurora global cluster provisioning  → `cloud.aws_ops.create_rds_global_cluster`
- **Covered today:** creates an Aurora global cluster with a primary cluster + instance and a
  cross-region replica cluster + instance (`amazon.cloud.rds_global_cluster` +
  `amazon.aws.rds_cluster`/`rds_instance`); delete path also handles **detach**
  (`amazon.aws.rds_cluster` with `remove_from_global_db: true`).
- **Maps to repo use cases:** the storage-infrastructure creation portion of `create_primary_instance` /
  `create_secondary_instance` and the Aurora side of MRBC.
- **How to improve mapping to the repo exactly:**
  - The role hardcodes a one-primary + one-replica topology with derived names
    (`<global>-primary`, `<global>-replica`). The repo drives naming from customer/instance identifiers
    and Terraform state — parameterize the role's cluster/instance names and allow **N replica
    regions** (the repo tracks multiple candidate writer regions in
    `try_mrbc_aurora_writer_candidates`).
  - Add KMS `storage_encrypted` / `kms_key_id` inputs (repo creates a customer KMS key in
    `create_kms_key_and_secrets`) — the role currently omits encryption params.
  - Return the writer endpoint / region as role facts so it can feed
    `refresh_mrbc_aurora_writer` / Dynatrace steps.

#### 2. RDS Aurora **detach** (`remove-from-global-cluster`) → `amazon.aws.rds_cluster`
- Corrects the earlier CLI-mapping "gap": `amazon.aws.rds_cluster` **does** expose
  `remove_from_global_db: true` (used by the delete path of `create_rds_global_cluster`) and a
  `promote` option.
- **Maps to repo use case:** `aws rds remove-from-global-cluster` in the MRBC failover flow → replace
  with `amazon.aws.rds_cluster` module call.
- **Improve:** wrap this in a small reusable task/role so the repo's `failover_deployment` /
  `refresh_mrbc_aurora_writer` playbooks call the module instead of the raw CLI.

#### 3. Credential / assume-role setup (cross-cutting) → `aws_setup_credentials` (both collections)
- **Covered:** `aws_setup_credentials` builds an `aws_setup_credentials__output` dict used as
  `module_defaults` for `group/aws` — the same pattern the repo needs for its 41 `sts_assume_role`
  calls.
- **Improve to map exactly:** the repo assumes *chained* roles per management/customer account
  (`set_management_cloud_account`, `get_rosa_credential`). Extend/wrap `aws_setup_credentials` to
  accept an assumed-role ARN + session and emit the same `module_defaults` dict, so all AWS tasks in
  the repo share one credentials contract.

#### 4. Network / connectivity validation (cross-cutting) → validation & troubleshooting modules/roles
- **Covered:** `validate_network_acls`, `validate_route_tables`, `validate_security_group_rules`
  (aws_ops); `eval_network_acls`, `eval_nat_network_acls`, `eval_security_groups`, `eval_src_igw_route`,
  `eval_vpc_peering`, `get_connection_next_hop` (aws_troubleshooting).
- **Maps to repo use cases:** ad-hoc reachability checks around ALB/Route 53 setup, PrivateLink
  endpoints, and shared DNS firewall association.
- **Improve:** use these instead of bespoke `assert`/CLI checks in `setup_alb_route53`,
  `shared_dns_firewal_association`, and endpoint tasks.

#### 5. Post-action connectivity troubleshooting → `cloud.aws_troubleshooting` roles
- **Covered (complementary):** `troubleshoot_rds_connectivity` (EC2→RDS: availability + SG + NACL +
  route tables) and `connectivity_troubleshooter` (+ `_local`, `_igw`, `_nat`, `_peering`,
  `_validate`).
- **Maps to repo use cases:** validation after `create_primary_instance`, after MRBC failover/DB
  restore, and when `sync_delete_failed_clusters` detects a bad instance.
- **Note:** the repo does not currently do this at all — this is net-new capability these collections
  add, not an existing use case they replace.

### B. Partially covered / different scope

- **Backup (use case 5):** `cloud.aws_ops.backup_create_plan` + `backup_select_resources` cover the
  **AWS Backup service** (plans/selections). The repo's `backup_ansible_platform` backs up the **AAP
  application** (flux config, k8s configmaps, cluster resources) — a *different* backup domain. Only
  useful if the team also wants AWS-native volume/RDS backups alongside the app backup.
- **Instance infrastructure building blocks (use case 1):** `ec2_networking_resources`,
  `manage_vpc_peering`, `manage_transit_gateway`, `manage_ec2_instance` provide low-level AWS infra,
  but the repo provisions **ROSA/OpenShift** clusters via Terraform — these roles could only replace
  isolated networking primitives, not the instance lifecycle.

### C. Left out (no coverage in either collection)

These are SaaS/AAP/ROSA-specific and outside both collections' scope:

- **Aurora global-cluster planned switchover / managed failover** — `create_rds_global_cluster` has no
  failover/switchover; `amazon.aws.rds_cluster` offers only `promote`/detach, not
  `switchover-global-cluster` (zero-data-loss). **Still a genuine gap** and the strongest candidate for
  new shared content.
- **ROSA/OpenShift instance provisioning & deletion** (primary/secondary, delete active/passive).
- **Post-deployment AAP configuration** (`apply_customer_configuration`, operator secrets, kustomize,
  installplan approval).
- **Release management** — customer/production releases, cherry-pick to stable, GitHub releases.
- **MRBC AAP database restore migration** (`mrbc_migration`) — k8s-side DB restore.
- **VPC Endpoint Service (provider-side PrivateLink)** setup/teardown + connection accept/reject.
- **Route 53 Resolver DNS Firewall** rule-group associations.
- **Licensing** — manifest upload.
- **Notifications** — Slack / customer notifications.
- **Inventory & subscription sync** — inventory service + Red Hat Marketplace APIs.
- **Capacity / pool management** — pool-service reconciliation.
- **Dynatrace** maintenance-window / event integration.
- **Test scaffolding** — dummy provision/deprovision playbooks.

### Bottom line

`cloud.aws_ops` **directly covers one repo use case** — Aurora global-cluster create/delete/detach
(plus cross-cutting credential setup and networking validation). `cloud.aws_troubleshooting` covers
**none of the existing use cases** but is a strong **complementary add-on** for post-provision and
post-failover validation. The SaaS/AAP/ROSA lifecycle that defines `customer-lifecycle-aws` — and the
Aurora **switchover/failover** operation specifically — remains out of scope for both.
