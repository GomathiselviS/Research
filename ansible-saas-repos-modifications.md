# Modifications to the Ansible-SaaS Repos (Consolidated)

Changes the SaaS repos should make to **use existing collection modules / shared content** instead of
raw `aws` CLI, and to fix FQCN/version inconsistencies. Grouped by repo, then cross-cutting.

Legend — **Type:** Refactor (CLI→module) · Fix (correctness) · Adopt (use existing collection/content) ·
Consistency · Docs.

## Cross-cutting (all Ansible repos)

| Change | Type | Rationale |
|---|---|---|
| Standardize `secretsmanager_secret` on `community.aws` (module) + `amazon.aws` lookup (reads); drop deprecated `aws_secret` alias | Fix | `amazon.aws.secretsmanager_secret` **module** does not exist (lookup only) — mixed FQCN usage is a latent failure |
| Pin AWS collection versions consistently in every `requirements.yml` | Consistency | Repos mix pinned 8.x and unpinned → module-resolution drift |
| Keep genuinely-uncovered CLI (endpoint-service, route53resolver, Aurora switchover) but wrap + comment as tech debt | Docs/guard | No module exists yet; track against `aws-collections-new-modules.md` / `-module-modifications.md` |

## Per-repo

| Repo | Change | Type | Target module / content |
|---|---|---|---|
| `ansible-saas-sre` | Migrate VPC endpoint CRUD, `elbv2 describe`, `wafv2 get/list/update-web-acl` to modules | Refactor | `ec2_vpc_endpoint`(+`_info`), `elb_application_lb_info`, `community.aws.wafv2_web_acl`(+`_info`) |
| `ansible-saas-sre` | Adopt WAF logging module for `wafv2 put/delete-logging-configuration` | Adopt | `amazon.cloud.wafv2_logging_configuration` |
| `ansible-saas-sre-operations` | `aws s3 cp` → module; `ec2 describe-vpc-endpoints` → info; `ec2 create-vpc-endpoint` → module | Refactor | `s3_object`, `ec2_vpc_endpoint_info`, `ec2_vpc_endpoint` |
| `ansible-saas-sre-operations` | Evaluate `amazon.cloud` WAF modules for the `aws wafv2` CLI in IP-allowlist/dynamic-rule tasks | Adopt | `amazon.cloud.wafv2_*` |
| `management-lifecycle` | Replace `secretsmanager replicate-secret-to-regions` (28×) with module `replica:` param | Refactor | `community.aws.secretsmanager_secret` |
| `management-lifecycle` | Migrate read CLIs (`elbv2 describe`, `s3api list/head`, `ec2 describe-*`, `rds describe-*`, `iam list-*`) to `*_info` modules | Refactor | `elb_application_lb_info`, `s3_bucket_info`, `ec2_vpc_net_info`/`ec2_vpc_nat_gateway_info`/`ec2_eip_info`, `rds_instance_info`/`rds_cluster_info`, `iam_role_info` |
| `management-lifecycle` | `s3 mb`/`rb` → module; `s3 cp` → module; `sts assume-role` → module | Refactor | `s3_bucket`, `s3_object`, `sts_assume_role` |
| `customer-lifecycle-aws` | `rds describe-db-clusters`/`describe-global-clusters` → info modules; `s3api head-*` → info | Refactor | `rds_cluster_info`, `rds_global_cluster_info`, `s3_bucket_info`/`s3_object_info` |
| `customer-lifecycle-aws` | `secretsmanager describe/get/delete-secret` → module/lookup | Refactor | `community.aws.secretsmanager_secret` + `amazon.aws` lookup |
| `customer-lifecycle-aws` | `rds remove-from-global-cluster` → module option (already supported) | Refactor | `amazon.aws.rds_cluster` (`remove_from_global_db: true`) |
| `customer-lifecycle-aws` | `wafv2 disassociate-web-acl` → module (`state: absent`) | Adopt | `amazon.cloud.wafv2_web_acl_association` / `community.aws.wafv2_resources` |
| `customer-lifecycle-aws` | Adopt shared validated content for Aurora global-cluster create/delete and connectivity validation | Adopt | `cloud.aws_ops.create_rds_global_cluster`, `cloud.aws_troubleshooting.*` |
| `ansible-saas-sops` | Cross-link each runbook `aws` command to its module equivalent where one exists | Docs | (all modules above) |
| `ansible-saas-sops` | Update PrivateLink runbooks once VPC endpoint-service modules land | Docs (follow-up) | `ec2_vpc_endpoint_service`, `ec2_vpc_endpoint_connection` |

## Reuse within the SaaS repos (dedup)

| Observation | Suggested action |
|---|---|
| The same task logic recurs across repos — `sts_assume_role` credential setup, `get_rosa_credential`, `load_tfstate`, `save_to_repository`, `scale_up/down_deployments`, shared DNS firewall association, VPC PrivateLink endpoint setup | Extract into a **shared internal collection** (e.g. the existing `redhat.customer_lifecycle_aws`) and have `sre`, `sre-operations`, `management-lifecycle`, `customer-lifecycle-aws` import it instead of duplicating task files |
| `secretsmanager_secret` / STS / S3 patterns are re-implemented per repo | Provide one shared role/task that emits `module_defaults` for `group/aws` (mirrors `cloud.aws_ops.aws_setup_credentials`) so all AWS tasks share one credentials contract |

## Blocked-until-upstream (no repo change possible yet)

These stay as `aws` CLI until the corresponding collection work lands (see the other two docs):

| Repo(s) | CLI that must remain until module exists |
|---|---|
| sre, sre-operations, management-lifecycle, customer-lifecycle, sops | `ec2 *-vpc-endpoint-service-configuration` / `-permissions` / `*-vpc-endpoint-connections` (provider-side PrivateLink) |
| sre, sre-operations, management-lifecycle, customer-lifecycle | `route53resolver *-firewall-*` (DNS Firewall) |
| customer-lifecycle | `rds failover-global-cluster` / `switchover-global-cluster` (Aurora managed failover/switchover) |
| management-lifecycle | `account put-alternate-contact` / `list-regions`; `iam create/get-open-id-connect-provider`; `iam create-service-linked-role` |
