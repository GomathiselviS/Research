# New Module Additions to the AWS Collections (Consolidated)

New modules needed to close the **true gaps** — AWS operations the SaaS repos perform via raw `aws` CLI
because **no module exists in `amazon.aws`, `community.aws`, or `amazon.cloud`**.

Consolidated across: `ansible-saas-sre`, `ansible-saas-sre-operations`, `management-lifecycle`,
`customer-lifecycle-aws`, and `ansible-saas-sops` runbooks.

Legend — **Priority:** P1 (recurring/high impact) · P2 (single-repo / medium) · P3 (niche).

## Table — Proposed New Modules

| Priority | Proposed module | Target collection | Replaces AWS CLI | Used in repos |
|---|---|---|---|---|
| P1 | `ec2_vpc_endpoint_service` | `amazon.aws` | `ec2 create/modify-vpc-endpoint-service-configuration` | sre, sre-operations, management-lifecycle, customer-lifecycle, sops |
| P1 | `ec2_vpc_endpoint_service` — allowed-principals / permissions (option or sibling module) | `amazon.aws` | `ec2 modify-vpc-endpoint-service-permissions` | sre, management-lifecycle |
| P1 | `ec2_vpc_endpoint_connection` + `ec2_vpc_endpoint_connection_info` | `amazon.aws` | `ec2 describe/accept/reject-vpc-endpoint-connections` | sre, customer-lifecycle, sops |
| P1 | `route53resolver_firewall_rule_group` + `_info` | `amazon.aws` | `route53resolver *-firewall-rule-group(s)` | sre, sre-operations, management-lifecycle, customer-lifecycle |
| P1 | `route53resolver_firewall_rule_group_association` | `amazon.aws` | `route53resolver associate/disassociate/list-firewall-rule-group-associations` | sre, customer-lifecycle |
| P1 | `route53resolver_firewall_domain_list` + `_info` | `amazon.aws` | `route53resolver *-firewall-domain-list(s)` | management-lifecycle |
| P1 | `route53resolver_firewall_rule` | `amazon.aws` | `route53resolver create/delete/list-firewall-rule(s)` | management-lifecycle |
| P1 | `account_alternate_contact` | `amazon.aws` | `account put-alternate-contact` | management-lifecycle |
| P1 | `account_region_info` (+ enable/disable region) | `amazon.aws` | `account list-regions` / enable-region | management-lifecycle |
| P1 | `iam_openid_connect_provider` + `_info` | `amazon.aws` | `iam create/get-open-id-connect-provider` | management-lifecycle |
| P2 | `iam_service_linked_role` (or extend `iam_role` — see modifications doc) | `amazon.aws` | `iam create-service-linked-role` | management-lifecycle |
| P2 | `wafv2_logging_configuration` (port from `amazon.cloud` for parity) | `community.aws` | `wafv2 put/delete-logging-configuration` | sre |
| P3 | `sts_decode_authorization_message` | `amazon.aws` | `sts decode-authorization-message` | sops |

## Notes

- **VPC Endpoint Service (provider side)** and **Route 53 Resolver DNS Firewall** are the two gaps that
  appear in the most repos — highest leverage. `amazon.aws` already ships the *consumer* side
  (`ec2_vpc_endpoint`, `ec2_vpc_endpoint_info`, `ec2_vpc_endpoint_service_info`), so the provider-side
  modules are a natural complement.
- **Collection home:** these are proposed for `amazon.aws` as the primary target; `route53resolver_*`
  and `account_*` could equally land in `community.aws` depending on maintainer preference. The one
  clearly-`community.aws` item is `wafv2_logging_configuration` (parity with the existing
  `amazon.cloud.wafv2_logging_configuration`).
- **Not listed here** (handled elsewhere): RDS Aurora Global Cluster *failover/switchover* is an
  extension to an existing module — see `aws-collections-module-modifications.md`.
