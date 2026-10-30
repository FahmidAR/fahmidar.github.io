# Terraform Spec Improvement ... Rules & Fixes Catalog

Two views per item:
- **[Rxx]** = generic rule (statement with ref markers, reference table, DSL draft)
- **[Sxx]** = concrete fix (purpose + exact portion added to the spec)

S-number matches R-number 1:1. 24 rules total: syntax rules R01 to R05 / S01 to S05, core security rules R06 to R24 / S06 to S24.

Every one of the 70 security findings now carries tfsec verified good and bad example pairs in the Security Error Fix tab, drawn from three sources: our dataset run (real old versus new code), online community reproducer patterns, and the official Terraform AWS provider registry form. Only pairs where tfsec fires on bad and clears on good are kept (207 verified pairs: 169 dataset, 28 online, 10 official); a pair that does not reproduce is dropped, not shown.

Rule frequency across the 458 prompt dataset (how many prompts each rule fired on):

| Rule | Title | Fired on (prompts) |
|---|---|---|
| R01 | Pin versions | 458 |
| R02 | Declare real resources | 227 |
| R03 | One routing-policy block | 7 |
| R04 | One alias or records | 22 |
| R05 | Required args, drop deprecated | 142 |
| R06 | Central encryption key | 309 |
| R07 | Encrypt log groups | 394 |
| R08 | Network flow logs | 347 |
| R09 | Restrict firewall rules | 129 |
| R10 | No public subnet IPs | 337 |
| R11 | Internal load balancer | 33 |
| R12 | Harden managed databases | 80 |
| R13 | Harden object storage | 167 |
| R14 | Managed platform service role | 50 |
| R15 | Encrypt remaining data stores | 138 |
| R16 | Backups and point in time recovery | 25 |
| R17 | Enforce TLS in transit | 38 |
| R18 | Service and audit logging | 82 |
| R19 | Keep managed services private | 56 |
| R20 | Require tokenized instance metadata | 39 |
| R21 | Protect edge endpoints | 13 |
| R22 | Identity least privilege and MFA | 149 |
| R23 | Enable function tracing | 25 |
| R24 | Drop invalid headers on load balancer | 15 |

---

## S0n conversion of the spec additions

The Generic/Exact/Why bullets, each given its S-number:

| S0n | Generic | maps to |
|---|---|---|
| S01 | provider and version baseline | R01 |
| S02 | resource schema validity | R02 |
| S03 | DNS routing schema (one routing-policy block per record, unique set_identifier) | R03 |
| S04 | DNS alias vs records (records+ttl OR one alias block, never both) | R04 |
| S05 | argument hygiene (all required args, remove deprecated) | R05 |
| S06 | encryption key (one rotated customer-managed KMS, reused) | R06 |
| S07 | log protection (kms_key_id + retention_in_days >= 365) | R07 |
| S08 | network traffic logging (aws_flow_log + IAM role per VPC) | R08 |
| S09 | firewall rules (description on every rule, no 0.0.0.0/0 or ::/0 ingress+egress) | R09 |
| S10 | subnet addressing (map_public_ip_on_launch = false) | R10 |
| S11 | load balancer exposure (internal = true, access logging) | R11 |
| S12 | database hardening (RDS controls) | R12 |
| S13 | object storage hardening (S3 separate resources) | R13 |
| S14 | platform service role (Elastic Beanstalk appversion_lifecycle) | R14 |
| S15 | remaining data store encryption (DynamoDB, EBS, EFS, Kinesis, SNS, SQS, Secrets, OpenSearch, DocumentDB, MSK) | R15 |
| S16 | backups and point in time recovery (DynamoDB PITR, Redshift snapshots) | R16 |
| S17 | TLS in transit (OpenSearch, CloudFront, ALB listeners, DocumentDB) | R17 |
| S18 | service and audit logging (CloudFront, EKS, OpenSearch, DocumentDB) | R18 |
| S19 | private managed services (EKS, Redshift, network ACLs, OpenSearch) | R19 |
| S20 | tokenized instance metadata (EC2 + launch template IMDSv2) | R20 |
| S21 | edge endpoint protection (API Gateway authorization, CloudFront WAF) | R21 |
| S22 | identity least privilege and MFA (IAM resource scoping, group MFA) | R22 |
| S23 | function tracing (Lambda X-Ray active tracing) | R23 |
| S24 | drop invalid headers on load balancer (ALB drop_invalid_header_fields) | R24 |

---

# RULES ALL

## Syntax Rules

### [R01] Pin versions
Add a `terraform` block with required version `>= n` **(a,c,e)** and required cloud-service-provider version `~> k` **(b,d)** nested inside it, plus a provider block with a region `r`. Here `n` and `k` should be stable and cover the latest version, and `r` should be default / user-given / selected for the user's region. For our dataset, 99% of providers are AWS, which needs a region defined.

| Ref | R01 part | source | Details |
|---|---|---|---|
| a | pin Terraform + providers | [Terraform Style Guide](https://developer.hashicorp.com/terraform/language/style) | explicitly says pin `required_version` and provider versions |
| b | required_providers `~> k` | [Provider Requirements](https://developer.hashicorp.com/terraform/language/providers/requirements) | `source` + `version` nested in `terraform{}` block |
| c | why pin | [Dependency Lock File](https://developer.hashicorp.com/terraform/language/files/dependency-lock) | records version+checksum so later `init` reproducible + syntax consistent |
| d | provider block + region | [AWS Provider docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) | `region` argument, `~> 5.x` pin convention |
| e | configuration syntax | [Manage Terraform versions](https://developer.hashicorp.com/terraform/tutorials/configuration-language/versions) | official version-configuration syntax |

**DSL intuition**
It fires when the file has no pinned terraform block. It scans the top of the file, then prepends a terraform block that pins the core version and the provider version (kept nested) plus a provider block carrying the region, so the configuration pins versions and names a region before anything else runs.

**DSL draft**
```
WHEN  file has no pinned terraform block
MATCH start of file
REWRITE prepend terraform{} with required_version >= n AND required_providers version ~> k (nested),
        AND a provider block with region r (for aws, region required)
```

### [S01] provider and version baseline
Keeps generated code on a current provider and removes the init failure caused by an unnested `required_providers` block. **Exact portion added** ... begin the file with EXACTLY this block; `required_providers` MUST stay nested inside the `terraform` block:
```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.60" }
  }
}

provider "aws" {
  region = var.aws_region   # or an explicit region
}
```

---

### [R02] Declare real resources
Use only the cloud provider's real resource types and arguments **(a)**, and declare every resource that is referenced **(b)**, so the configuration passes `terraform validate` **(c)** instead of failing on unknown types or undeclared references.

| Ref | R02 part | source | Details |
|---|---|---|---|
| a | real types + arguments | [AWS Provider docs (Registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) | authoritative v5 resource/argument schema |
| b | declare referenced resources | [Resource Blocks](https://developer.hashicorp.com/terraform/language/resources/syntax) / [References to Values](https://developer.hashicorp.com/terraform/language/expressions/references) | every `type.name` referenced must be declared |
| c | validate gate | [`terraform validate`](https://developer.hashicorp.com/terraform/cli/commands/validate) | checks config internally consistent (types, refs, args) |

**DSL intuition**
It fires when validate reports an undeclared reference, a made up resource type, or an unsupported argument or block. It locates the fake type or the reference that points at nothing, then swaps the fake type for a real provider type and declares a real resource for every dangling reference, so validate stops failing.

**DSL draft**
```
WHEN  error in {Reference to undeclared resource, Invalid resource type, Unsupported argument, Unsupported block}
MATCH resource "<fake_type>" ...  |  <ref>.<name> with no declaration
REWRITE replace fake type with real AWS v5 type; add resource "<type>" "<name>" {...} for each dangling reference
```

### [S02] resource schema validity
Only real AWS v5 types/arguments, every referenced resource declared, so `terraform validate` passes. **Exact portion added** (transform, not boilerplate): swap invented types for real registry types; for each `X.name` with no declaration, add `resource "X" "name" { # required args per v5 schema }`.

---

### [R03] One routing-policy block
Use one routing-policy block per `aws_route53_record` with a unique `set_identifier` **(a)**, one record resource per target **(b)**.

| Ref | R03 part | source | Details |
|---|---|---|---|
| a | routing policy + set_identifier | AWS Provider: [`aws_route53_record`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53_record) | `weighted/geolocation/latency_routing_policy` + `set_identifier` |
| b | one record per target | AWS Provider: [`aws_route53_record`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53_record) | multiple routing blocks in one record is invalid |

**DSL intuition**
It fires when one DNS record holds more than one routing policy block, or validate complains about too many policy blocks. It finds the record stuffed with several policy blocks, then splits them into one record per target, each with a single policy block and its own set_identifier.

**DSL draft**
```
WHEN  record has >1 routing_policy block OR error "Too many weighted_routing_policy blocks"
MATCH aws_route53_record { <policy>{} <policy>{} ... }
REWRITE one aws_route53_record per target: { <policy>{} set_identifier="<id>" }
```

### [S03] DNS routing schema
Matches the route53 schema so the record validates. **Exact portion added**: one routing-policy block per record, unique `set_identifier`, one record resource per target.

---

### [R04] One alias or records, never both
Keep a single `alias` block, OR `records` + `ttl`, per record ... never both **(a)**.

| Ref | R04 part | source | Details |
|---|---|---|---|
| a | alias vs records | AWS Provider: [`aws_route53_record`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/route53_record) | `records`+`ttl` for non-alias, exactly one `alias{}` for alias |

**DSL intuition**
It fires when a DNS record has more than one alias block, or mixes an alias block with plain records. It finds that conflicting record, then reduces it to a single alias block, or to records plus a ttl, never both at once.

**DSL draft**
```
WHEN  record has >1 alias block OR alias together with records
MATCH aws_route53_record { alias{} alias{} }  |  { records=[...] alias{} }
REWRITE aws_route53_record { alias{ name zone_id evaluate_target_health } }  (or records+ttl)
```

### [S04] DNS alias vs records
Record validates with one addressing style. **Exact portion added**: use `records = [...]` + `ttl` for non-alias records, or exactly one `alias {}` block for alias records; never both.

---

### [R05] Required args, drop deprecated
Supply every required argument **(a)** and remove/relocate deprecated inline arguments **(b)**.

| Ref | R05 part | source | Details |
|---|---|---|---|
| a | required arguments | [AWS Provider resource pages](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) | each resource's mandatory args |
| b | deprecated arguments | [AWS Provider upgrade guides](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/version-5-upgrade) | move deprecated inline blocks to their v5 resources |

**DSL intuition**
It fires when validate reports a missing required argument or a deprecated inline block. It finds the resource that is missing the argument, or the deprecated inline block, then adds the required argument, or replaces the old inline block with its current standalone resource.

**DSL draft**
```
WHEN  error in {Missing required argument, Argument is deprecated}
MATCH resource {... missing required arg ...}  |  deprecated inline block
REWRITE add the required argument; replace deprecated inline block with its v5 resource
```

### [S05] argument hygiene
Clears missing-argument errors and deprecation warnings. **Exact portion added**: supply all required arguments for each resource; remove deprecated arguments.

---

## Security Rules

### [R06] Central encryption key  ... Fixes: AVD-AWS-0065 (underpins 0017, 0080, 0088, 0132)
Add one rotated managed encryption key (`aws_kms_key` with `enable_key_rotation = true`) **(a)** and reference its arn from every encryptable resource **(b)**, so encryption-at-rest findings clear with a single managed, rotated key **(c)**.

| Ref | R06 part | source | Details |
|---|---|---|---|
| a | rotated CMK | AWS Provider: [`aws_kms_key`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_key) | `enable_key_rotation` + `deletion_window_in_days` |
| b | reuse arn for encryption | [`aws_cloudwatch_log_group`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group), S3 SSE, [`aws_db_instance`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance) | `kms_key_id = aws_kms_key.main.arn` on each |
| c | why rotate | Aqua AVD ... [AVD-AWS-0065](https://avd.aquasec.com/misconfig/avd-aws-0065/) | one key satisfies logs 0017, S3 0088/0132, RDS 0080 |

**DSL intuition**
It fires when any resource needs encryption at rest, or a key related finding shows up. It checks whether a managed encryption key already exists, and if none does it adds one rotated key, then points every encryptable resource (logs, object storage, databases) at that single key's arn.

**DSL draft**
```
WHEN  any resource needs encryption at rest OR error in {AVD-AWS-0065, 0017, 0080, 0088, 0132}
MATCH (no aws_kms_key present)
REWRITE add resource "aws_kms_key" "main" { enable_key_rotation = true };
        set kms_key_id = aws_kms_key.main.arn on every log group, S3 SSE, RDS
```

### [S06] central encryption key
One customer-managed, rotated CMK reused everywhere satisfies all encryption-at-rest checks and avoids multiple unmanaged keys. **Exact portion added** ... add this once at the top level and reference `aws_kms_key.main.arn` from every encryptable resource:
```hcl
resource "aws_kms_key" "main" {
  description             = "customer managed key"
  enable_key_rotation     = true
  deletion_window_in_days = 7
}
```
Then on each encryptable resource: `kms_key_id = aws_kms_key.main.arn`.

---

### [R07] Encrypt log groups  ... Fixes: AVD-AWS-0017
Set `kms_key_id` **(a)** and `retention_in_days >= 365` **(b)** on every managed log group.

| Ref | R07 part | source | Details |
|---|---|---|---|
| a | log encryption | AWS Provider: [`aws_cloudwatch_log_group`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group) | `kms_key_id = aws_kms_key.main.arn` |
| b | retention | Aqua AVD ... [AVD-AWS-0017](https://avd.aquasec.com/misconfig/avd-aws-0017/) | `retention_in_days >= 365` |

**DSL intuition**
It fires when a managed log group is present, or its encryption finding shows up. It matches the log group, then attaches the managed encryption key and sets retention to 365 days.

**DSL draft**
```
WHEN  resource aws_cloudwatch_log_group present OR error AVD-AWS-0017
MATCH aws_cloudwatch_log_group { ... }
REWRITE add kms_key_id = aws_kms_key.main.arn; set retention_in_days = 365
```

### [S07] log protection
Logs are encrypted and kept long enough for the checks. **Exact portion added** ... add `kms_key_id` and `retention_in_days` to EVERY `aws_cloudwatch_log_group`, pointing at the central key:
```hcl
resource "aws_cloudwatch_log_group" "example" {
  name              = "example-logs"
  retention_in_days = 365
  kms_key_id        = aws_kms_key.main.arn
}
```

---

### [R08] Network flow logs  ... Fixes: AVD-AWS-0178
Add a network flow log (`aws_flow_log` with its logging role) per virtual network **(a)**, using the correct schema with no `resource_type` argument **(b)**.

| Ref | R08 part | source | Details |
|---|---|---|---|
| a | flow log + role | AWS Provider: [`aws_flow_log`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/flow_log), [`aws_iam_role`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | `vpc_id`, `iam_role_arn`, log to encrypted group |
| b | correct schema | Aqua AVD ... [AVD-AWS-0178](https://avd.aquasec.com/misconfig/avd-aws-0178/) | [`aws_flow_log`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/flow_log) has NO `resource_type` argument |

**DSL intuition**
It fires when a virtual network is present, or its flow log finding shows up. It matches the network, then adds a flow log for it (all traffic, sent to the encrypted log group) together with the logging role the flow log needs.

**DSL draft**
```
WHEN  resource aws_vpc present OR error AVD-AWS-0178
MATCH aws_vpc "<n>" {}
REWRITE add aws_flow_log { vpc_id=aws_vpc.<n>.id traffic_type="ALL" log_destination_type="cloud-watch-logs"
        log_destination=<group>.arn iam_role_arn=<role>.arn } + its IAM role
```

### [S08] network traffic logging
Enables Network flow logs with a valid resource. **Exact portion added** ... add one `aws_flow_log` plus its IAM role for EVERY VPC, using no `resource_type` argument:
```hcl
resource "aws_iam_role" "flow" {
  name               = "vpc-flow-role"
  assume_role_policy = jsonencode({ Version = "2012-10-17", Statement = [{
    Effect = "Allow", Principal = { Service = "vpc-flow-logs.amazonaws.com" }, Action = "sts:AssumeRole" }] })
}
resource "aws_flow_log" "vpc" {
  vpc_id               = aws_vpc.main.id
  traffic_type         = "ALL"
  log_destination_type = "cloud-watch-logs"
  log_destination      = aws_cloudwatch_log_group.example.arn
  iam_role_arn         = aws_iam_role.flow.arn
}
```

---

### [R09] Restrict firewall rules  ... Fixes: AVD-AWS-0099, 0104, 0107, 0124
Give every rule a `description` **(a)** and replace `0.0.0.0/0` or `::/0` (ingress AND egress) with the private network CIDR **(b)**.

| Ref | R09 part | source | Details |
|---|---|---|---|
| a | rule description | Aqua AVD ... [AVD-AWS-0099](https://avd.aquasec.com/misconfig/avd-aws-0099/)/0124 | every ingress/egress rule needs `description` |
| b | restrict CIDR | Aqua AVD ... [AVD-AWS-0107](https://avd.aquasec.com/misconfig/avd-aws-0107/) (ingress), 0104 (egress) | no public CIDR; use `aws_vpc.main.cidr_block` |

**DSL intuition**
It fires when a firewall rule opens to the whole internet (0.0.0.0/0 or ::/0), lacks a description, or raises a rule finding. It matches that open ingress or egress rule, then adds a description and narrows the CIDR to the private network range.

**DSL draft**
```
WHEN  rule uses 0.0.0.0/0 or ::/0, or lacks description, or error AVD-AWS-0107/0104/0099/0124
MATCH ingress/egress { cidr_blocks=["0.0.0.0/0"] }
REWRITE { description="..." cidr_blocks=[aws_vpc.main.cidr_block] }
```

### [S09] firewall rules
Removes public exposure and undescribed rules. **Exact portion added** ... give EVERY security group rule a `description` and a non-public CIDR, on ingress AND egress:
```hcl
resource "aws_security_group" "main" {
  name        = "main"
  description = "managed security group"
  vpc_id      = aws_vpc.main.id
  ingress {
    description = "app traffic from within VPC"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }
  egress {
    description = "egress within VPC"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }
}
```

---

### [R10] No public subnet IPs  ... Fixes: AVD-AWS-0164
Set `map_public_ip_on_launch = false` on subnets **(a)**.

| Ref | R10 part | source | Details |
|---|---|---|---|
| a | no auto public IP | AWS Provider: [`aws_subnet`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet), Aqua AVD ... [AVD-AWS-0164](https://avd.aquasec.com/misconfig/avd-aws-0164/) | `map_public_ip_on_launch = false` |

**DSL intuition**
It fires when a subnet is present, or its public IP finding shows up. It matches the subnet, then turns off automatic public IP assignment on launch.

**DSL draft**
```
WHEN  resource aws_subnet present OR error AVD-AWS-0164
MATCH aws_subnet { ... }
REWRITE set map_public_ip_on_launch = false
```

### [S10] subnet addressing
Instances do not receive an automatic public IP. **Exact portion added**: set `map_public_ip_on_launch = false` on subnets.

---

### [R11] Internal load balancer  ... Fixes: AVD-AWS-0053
Set `internal = true` **(a)** and enable access logging **(b)** on the load balancer.

| Ref | R11 part | source | Details |
|---|---|---|---|
| a | not public | AWS Provider: [`aws_lb`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb)/[`aws_elb`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elb), Aqua AVD ... [AVD-AWS-0053](https://avd.aquasec.com/misconfig/avd-aws-0053/) | `internal = true` |
| b | access logging | AWS Provider: [`aws_lb`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb) | `access_logs { bucket=... enabled=true }` |

**DSL intuition**
It fires when a load balancer is present, or its exposure finding shows up. It matches the public balancer, then makes it internal and turns on access logging.

**DSL draft**
```
WHEN  resource aws_elb/aws_lb present OR error AVD-AWS-0053
MATCH aws_elb { ...public... }
REWRITE aws_lb { internal=true access_logs{ bucket=... enabled=true } }
```

### [S11] load balancer exposure
The balancer is not public and records access logs. **Exact portion added**: set `internal = true`; enable access logging on `aws_elb`/`aws_lb`.

---

### [R12] Harden managed databases  ... Fixes: AVD-AWS-0077, 0080, 0133, 0176, 0177 (also 0079, 0082)
Set `storage_encrypted`+`kms_key_id`, `iam_database_authentication_enabled`, `deletion_protection`, `performance_insights_enabled`, `backup_retention_period >= 7`, `multi_az` **(a)**; remove hardcoded passwords **(b)**.

| Ref | R12 part | source | Details |
|---|---|---|---|
| a | six hardening args | AWS Provider: [`aws_db_instance`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance), Aqua AVD ... 0080/0077/0176/0177/0133 | encryption, IAM auth, deletion protection, PI, backups, multi-AZ |
| b | no plaintext password | Aqua AVD | never hardcode `password` |

**DSL intuition**
It fires when a managed relational database is present, or one of its hardening findings shows up. It matches the database, including any plaintext password, then adds the six hardening arguments (encryption, database auth, deletion protection, performance insights, backups, multi az) and removes the plaintext password.

**DSL draft**
```
WHEN  resource aws_db_instance present OR error AVD-AWS-0080/0077/0176/0177/0133
MATCH aws_db_instance { ... password="..." }
REWRITE add the six hardening args; drop the plaintext password
```

### [S12] database hardening
The database is encrypted, authenticated, backed up, and protected. **Exact portion added** ... add these arguments to EVERY `aws_db_instance`, and never hardcode a password:
```hcl
resource "aws_db_instance" "main" {
  # ...
  storage_encrypted                   = true
  kms_key_id                          = aws_kms_key.main.arn
  iam_database_authentication_enabled = true
  deletion_protection                 = true
  performance_insights_enabled        = true
  performance_insights_kms_key_id     = aws_kms_key.main.arn
  backup_retention_period             = 7
  multi_az                            = true
  # never hardcode password
}
```

---

### [R13] Harden object storage  ... Fixes: AVD-AWS-0086, 0087, 0088, 0089, 0090, 0091, 0092, 0093, 0094, 0131, 0132
Add SEPARATE resources: public access block (all four true) **(a)**, server-side encryption with the managed key **(b)**, versioning Enabled **(c)**, bucket logging **(d)**.

| Ref | R13 part | source | Details |
|---|---|---|---|
| a | public access block | [`aws_s3_bucket_public_access_block`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_public_access_block), [AVD-AWS-0086](https://avd.aquasec.com/misconfig/avd-aws-0086/)/0087/0091/0093/0094 | all four settings `true` |
| b | encryption | [`aws_s3_bucket_server_side_encryption_configuration`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_server_side_encryption_configuration), [AVD-AWS-0088](https://avd.aquasec.com/misconfig/avd-aws-0088/)/0132/0131 | KMS CMK, `sse_algorithm = "aws:kms"` |
| c | versioning | [`aws_s3_bucket_versioning`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_versioning), [AVD-AWS-0090](https://avd.aquasec.com/misconfig/avd-aws-0090/) | `status = "Enabled"` |
| d | logging | [`aws_s3_bucket_logging`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_logging), [AVD-AWS-0089](https://avd.aquasec.com/misconfig/avd-aws-0089/) | log to a dedicated bucket |

**DSL intuition**
It fires when an object storage bucket is present, or one of its findings shows up. It matches the bucket, then adds the separate companion resources (public access block, server side encryption, versioning, logging) that each reference that bucket.

**DSL draft**
```
WHEN  resource aws_s3_bucket present OR error AVD-AWS-0086/0087/0088/0089/0090/0132
MATCH aws_s3_bucket "<b>" { acl=... }
REWRITE add aws_s3_bucket_public_access_block / _server_side_encryption_configuration / _versioning / _logging for <b>
```

### [S13] object storage hardening
Blocks public access and encrypts, versions, and logs data. **Exact portion added** ... for EVERY bucket add these SEPARATE companion resources, not nested blocks:
```hcl
resource "aws_s3_bucket" "b" { bucket = "my-bucket" }
resource "aws_s3_bucket_public_access_block" "b" {
  bucket                  = aws_s3_bucket.b.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
resource "aws_s3_bucket_server_side_encryption_configuration" "b" {
  bucket = aws_s3_bucket.b.id
  rule { apply_server_side_encryption_by_default {
    kms_master_key_id = aws_kms_key.main.arn
    sse_algorithm     = "aws:kms"
  } }
}
resource "aws_s3_bucket_versioning" "b" {
  bucket = aws_s3_bucket.b.id
  versioning_configuration { status = "Enabled" }
}
```

---

### [R14] Managed platform service role  ... Fixes: missing-required-argument (validate)
Give the `appversion_lifecycle` block a `service_role` (a service role arn) **(a)**, or omit the block **(b)**.

| Ref | R14 part | source | Details |
|---|---|---|---|
| a | service_role | AWS Provider: [`aws_elastic_beanstalk_application`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elastic_beanstalk_application) | `appversion_lifecycle` requires `service_role` |
| b | omit block | AWS Provider | drop `appversion_lifecycle` if no role |

**DSL intuition**
It fires when a managed platform application has a lifecycle block with no service role. It matches that lifecycle block, then adds the service role arn, or removes the block if it is not needed.

**DSL draft**
```
WHEN  aws_elastic_beanstalk_application has appversion_lifecycle without service_role
MATCH appversion_lifecycle { max_age_in_days=... }
REWRITE appversion_lifecycle { service_role=aws_iam_role.eb_service.arn max_age_in_days=... }  (or remove the block)
```

### [S14] platform service role
Clears the missing-required-argument error. **Exact portion added**: include `service_role = aws_iam_role.eb_service.arn` in `appversion_lifecycle` (and declare that role), or omit the block entirely.


---

## Security Rules (R15 to R24)


### [R15] Encrypt remaining data stores  ... Fixes: AVD-AWS-0023, AVD-AWS-0025, AVD-AWS-0026, AVD-AWS-0027, AVD-AWS-0037, AVD-AWS-0039, AVD-AWS-0048, AVD-AWS-0064, AVD-AWS-0076, AVD-AWS-0084, AVD-AWS-0095, AVD-AWS-0096, AVD-AWS-0098, AVD-AWS-0128, AVD-AWS-0179
Enable encryption at rest with the customer managed key on every remaining store (managed NoSQL tables, block and file storage, streaming, messaging, secrets, search and document databases).

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_dynamodb_table`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0023](https://avd.aquasec.com/misconfig/avd-aws-0023/), [AVD-AWS-0025](https://avd.aquasec.com/misconfig/avd-aws-0025/), [AVD-AWS-0026](https://avd.aquasec.com/misconfig/avd-aws-0026/), [AVD-AWS-0027](https://avd.aquasec.com/misconfig/avd-aws-0027/), [AVD-AWS-0037](https://avd.aquasec.com/misconfig/avd-aws-0037/), [AVD-AWS-0039](https://avd.aquasec.com/misconfig/avd-aws-0039/), [AVD-AWS-0048](https://avd.aquasec.com/misconfig/avd-aws-0048/), [AVD-AWS-0064](https://avd.aquasec.com/misconfig/avd-aws-0064/), [AVD-AWS-0076](https://avd.aquasec.com/misconfig/avd-aws-0076/), [AVD-AWS-0084](https://avd.aquasec.com/misconfig/avd-aws-0084/), [AVD-AWS-0095](https://avd.aquasec.com/misconfig/avd-aws-0095/), [AVD-AWS-0096](https://avd.aquasec.com/misconfig/avd-aws-0096/), [AVD-AWS-0098](https://avd.aquasec.com/misconfig/avd-aws-0098/), [AVD-AWS-0128](https://avd.aquasec.com/misconfig/avd-aws-0128/), [AVD-AWS-0179](https://avd.aquasec.com/misconfig/avd-aws-0179/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S15] fix block, which adds encryption at rest with the customer managed key to the remaining data stores.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0023, AVD-AWS-0025, AVD-AWS-0026, AVD-AWS-0027, AVD-AWS-0037, AVD-AWS-0039, AVD-AWS-0048, AVD-AWS-0064, AVD-AWS-0076, AVD-AWS-0084, AVD-AWS-0095, AVD-AWS-0096, AVD-AWS-0098, AVD-AWS-0128, AVD-AWS-0179}
MATCH resource missing the control
REWRITE apply the encrypt remaining data stores control shown in [S15]
```

### [S15] encrypt remaining data stores
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_sqs_queue" "jobs" {
  name = "jobs"
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_sqs_queue" "jobs" {
  name              = "jobs"
  kms_master_key_id = aws_kms_key.main.arn
}
resource "aws_kms_key" "main" {
  description         = "cmk"
  enable_key_rotation = true
}

```

### [R16] Backups and point in time recovery  ... Fixes: AVD-AWS-0024, AVD-AWS-0050
Enable point in time recovery (managed NoSQL tables) and snapshot retention (the data warehouse) so data can be restored.

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_dynamodb_table`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/dynamodb_table) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0024](https://avd.aquasec.com/misconfig/avd-aws-0024/), [AVD-AWS-0050](https://avd.aquasec.com/misconfig/avd-aws-0050/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S16] fix block, which turns on point in time recovery and snapshot retention so data can be restored.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0024, AVD-AWS-0050}
MATCH resource missing the control
REWRITE apply the backups and point in time recovery control shown in [S16]
```

### [S16] backups and point in time recovery
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_elasticache_cluster" "c" {
  cluster_id        = "cache"
  engine            = "redis"
  node_type         = "cache.t3.micro"
  num_cache_nodes   = 1
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_elasticache_cluster" "c" {
  cluster_id               = "cache"
  engine                   = "redis"
  node_type                = "cache.t3.micro"
  num_cache_nodes          = 1
  snapshot_retention_limit = 7
}
```

### [R17] Enforce TLS in transit  ... Fixes: AVD-AWS-0012, AVD-AWS-0013, AVD-AWS-0043, AVD-AWS-0046, AVD-AWS-0047, AVD-AWS-0054, AVD-AWS-0073, AVD-AWS-0126
Require HTTPS and a modern TLS policy in transit on search, the CDN, load balancer listeners and managed database endpoints.

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_elasticsearch_domain`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elasticsearch_domain) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0012](https://avd.aquasec.com/misconfig/avd-aws-0012/), [AVD-AWS-0013](https://avd.aquasec.com/misconfig/avd-aws-0013/), [AVD-AWS-0043](https://avd.aquasec.com/misconfig/avd-aws-0043/), [AVD-AWS-0046](https://avd.aquasec.com/misconfig/avd-aws-0046/), [AVD-AWS-0047](https://avd.aquasec.com/misconfig/avd-aws-0047/), [AVD-AWS-0054](https://avd.aquasec.com/misconfig/avd-aws-0054/), [AVD-AWS-0073](https://avd.aquasec.com/misconfig/avd-aws-0073/), [AVD-AWS-0126](https://avd.aquasec.com/misconfig/avd-aws-0126/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S17] fix block, which requires HTTPS and a modern TLS policy on the service endpoints.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0012, AVD-AWS-0013, AVD-AWS-0043, AVD-AWS-0046, AVD-AWS-0047, AVD-AWS-0054, AVD-AWS-0073, AVD-AWS-0126}
MATCH resource missing the control
REWRITE apply the enforce tls in transit control shown in [S17]
```

### [S17] enforce tls in transit
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_elasticsearch_domain" "e" {
  domain_name = "e"
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_elasticsearch_domain" "e" {
  domain_name = "e"
  domain_endpoint_options {
    enforce_https       = true
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"
  }
}
```

### [R18] Service and audit logging  ... Fixes: AVD-AWS-0010, AVD-AWS-0038, AVD-AWS-0042, AVD-AWS-0074, AVD-AWS-0075
Turn on access or audit logging on the CDN, managed Kubernetes, search and document database services so activity is recorded.

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_eks_cluster`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_cluster) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0010](https://avd.aquasec.com/misconfig/avd-aws-0010/), [AVD-AWS-0038](https://avd.aquasec.com/misconfig/avd-aws-0038/), [AVD-AWS-0042](https://avd.aquasec.com/misconfig/avd-aws-0042/), [AVD-AWS-0074](https://avd.aquasec.com/misconfig/avd-aws-0074/), [AVD-AWS-0075](https://avd.aquasec.com/misconfig/avd-aws-0075/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S18] fix block, which turns on access or audit logging so activity is recorded.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0010, AVD-AWS-0038, AVD-AWS-0042, AVD-AWS-0074, AVD-AWS-0075}
MATCH resource missing the control
REWRITE apply the service and audit logging control shown in [S18]
```

### [S18] service and audit logging
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_eks_cluster" "c" {
  name     = "c"
  role_arn = "arn:aws:iam::111122223333:role/eks"
  vpc_config { subnet_ids = ["subnet-1", "subnet-2"] }
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_eks_cluster" "c" {
  name                      = "c"
  role_arn                  = "arn:aws:iam::111122223333:role/eks"
  enabled_cluster_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
  vpc_config { subnet_ids = ["subnet-1", "subnet-2"] }
}
```

### [R19] Keep managed services private  ... Fixes: AVD-AWS-0040, AVD-AWS-0041, AVD-AWS-0105, AVD-AWS-0127, AVD-AWS-0180
Disable public access and public CIDRs on managed Kubernetes, the data warehouse, network ACLs and search; keep them inside the virtual network.

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_redshift_cluster`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/redshift_cluster) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0040](https://avd.aquasec.com/misconfig/avd-aws-0040/), [AVD-AWS-0041](https://avd.aquasec.com/misconfig/avd-aws-0041/), [AVD-AWS-0105](https://avd.aquasec.com/misconfig/avd-aws-0105/), [AVD-AWS-0127](https://avd.aquasec.com/misconfig/avd-aws-0127/), [AVD-AWS-0180](https://avd.aquasec.com/misconfig/avd-aws-0180/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S19] fix block, which disables public access and keeps the service inside the virtual network.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0040, AVD-AWS-0041, AVD-AWS-0105, AVD-AWS-0127, AVD-AWS-0180}
MATCH resource missing the control
REWRITE apply the keep managed services private control shown in [S19]
```

### [S19] keep managed services private
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_network_acl_rule" "ssh" {
  network_acl_id = aws_network_acl.a.id
  rule_number    = 110
  egress         = false
  protocol       = "tcp"
  from_port      = 22
  to_port        = 22
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_network_acl_rule" "ssh" {
  network_acl_id = aws_network_acl.a.id
  rule_number    = 110
  egress         = false
  protocol       = "tcp"
  from_port      = 22
  to_port        = 22
  rule_action    = "allow"
  cidr_block     = "10.0.0.0/16"
}
```

### [R20] Require tokenized instance metadata  ... Fixes: AVD-AWS-0028, AVD-AWS-0130
Require tokenized instance metadata (IMDSv2 session tokens) on compute instances and launch templates so instance metadata is not open.

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_instance`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0028](https://avd.aquasec.com/misconfig/avd-aws-0028/), [AVD-AWS-0130](https://avd.aquasec.com/misconfig/avd-aws-0130/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S20] fix block, which requires tokenized instance metadata so instance metadata is not open.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0028, AVD-AWS-0130}
MATCH resource missing the control
REWRITE apply the require imdsv2 tokens control shown in [S20]
```

### [S20] require imdsv2 tokens
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_instance" "i" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.micro"
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_instance" "i" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.micro"
  metadata_options {
    http_tokens   = "required"
    http_endpoint = "enabled"
  }
}
```

### [R21] Protect edge endpoints  ... Fixes: AVD-AWS-0004, AVD-AWS-0011
Require authorization on API gateway methods and attach a web application firewall to the CDN so public endpoints are not open.

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_api_gateway_method`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/api_gateway_method) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0004](https://avd.aquasec.com/misconfig/avd-aws-0004/), [AVD-AWS-0011](https://avd.aquasec.com/misconfig/avd-aws-0011/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S21] fix block, which requires authorization on the API methods and puts a web application firewall in front of the CDN.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0004, AVD-AWS-0011}
MATCH resource missing the control
REWRITE apply the protect edge endpoints control shown in [S21]
```

### [S21] protect edge endpoints
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_api_gateway_rest_api" "a" {
  name = "api"
}
resource "aws_api_gateway_resource" "r" {
  rest_api_id = aws_api_gateway_rest_api.a.id
  parent_id   = aws_api_gateway_rest_api.a.root_resource_id
  path_part   = "items"
}
resource "aws_api_gateway_method" "post" {
  rest_api_id   = aws_api_gateway_rest_api.a.id
  resource_id   = aws_api_gateway_resource.r.id
  http_method   = "POST"
  authorization = "NONE"
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_api_gateway_rest_api" "a" {
  name = "api"
}
resource "aws_api_gateway_resource" "r" {
  rest_api_id = aws_api_gateway_rest_api.a.id
  parent_id   = aws_api_gateway_rest_api.a.root_resource_id
  path_part   = "items"
}
resource "aws_api_gateway_method" "post" {
  rest_api_id   = aws_api_gateway_rest_api.a.id
  resource_id   = aws_api_gateway_resource.r.id
  http_method   = "POST"
  authorization = "AWS_IAM"
}
```

### [R22] Identity least privilege and MFA  ... Fixes: AVD-AWS-0057, AVD-AWS-0123
Scope identity policy resources instead of wildcards for sensitive actions, and enforce MFA on groups.

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_iam_policy`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0057](https://avd.aquasec.com/misconfig/avd-aws-0057/), [AVD-AWS-0123](https://avd.aquasec.com/misconfig/avd-aws-0123/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S22] fix block, which scopes identity policy resources instead of wildcards and enforces MFA on groups.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0057, AVD-AWS-0123}
MATCH resource missing the control
REWRITE apply the iam least privilege and mfa control shown in [S22]
```

### [S22] iam least privilege and mfa
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_iam_policy" "p" {
  name   = "p"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow", Action = "s3:GetObject", Resource = "*" }]
  })
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_iam_policy" "p" {
  name   = "p"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow", Action = "s3:GetObject", Resource = "arn:aws:s3:::my-bucket/*" }]
  })
}
```

### [R23] Enable function tracing  ... Fixes: AVD-AWS-0066
Enable active request tracing on serverless functions for observability.

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_lambda_function`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_function) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0066](https://avd.aquasec.com/misconfig/avd-aws-0066/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S23] fix block, which turns on active request tracing on the serverless functions.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0066}
MATCH resource missing the control
REWRITE apply the enable lambda tracing control shown in [S23]
```

### [S23] enable lambda tracing
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_lambda_function" "f" {
  function_name = "f"
  role          = "arn:aws:iam::111122223333:role/l"
  handler       = "index.handler"
  runtime       = "python3.12"
  filename      = "f.zip"
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_lambda_function" "f" {
  function_name = "f"
  role          = "arn:aws:iam::111122223333:role/l"
  handler       = "index.handler"
  runtime       = "python3.12"
  filename      = "f.zip"
  tracing_config { mode = "Active" }
}
```

### [R24] Drop invalid headers on load balancer  ... Fixes: AVD-AWS-0052
Set drop_invalid_header_fields on application load balancers.

| Ref | part | source | Details |
|---|---|---|---|
| a | control | [`aws_lb`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb) | the AWS provider v5 argument that applies the control |
| b | finding | Aqua AVD | [AVD-AWS-0052](https://avd.aquasec.com/misconfig/avd-aws-0052/) |

**DSL intuition**
It fires when tfsec raises any of this rule's findings. It matches the resource that is missing the control, then applies the exact [S24] fix block, which drops invalid request headers on the load balancer.

**DSL draft**
```
WHEN  finding in {AVD-AWS-0052}
MATCH resource missing the control
REWRITE apply the drop invalid headers on load balancer control shown in [S24]
```

### [S24] drop invalid headers on load balancer
tfsec verified: the bad example below triggers the finding and the fixed example clears it.

Bad example:
```hcl
resource "aws_lb" "a" {
  name               = "a"
  load_balancer_type = "application"
  subnets            = ["subnet-1", "subnet-2"]
}
```
Good example (fix applied, tfsec clean):
```hcl
resource "aws_lb" "a" {
  name                       = "a"
  load_balancer_type         = "application"
  subnets                    = ["subnet-1", "subnet-2"]
  drop_invalid_header_fields = true
}
```
