# Terraform Spec Improvement ... Rules & Fixes Catalog

Two views per item:
- **[Rxx]** = generic rule (statement with ref markers, reference table, DSL draft)
- **[Sxx]** = concrete fix (purpose + exact portion added to the spec)

S-number matches R-number 1:1. Syntax rules = R01–R05 / S01–S05. Security rules = R06–R14 / S06–S14.

Rule frequency across the 458 prompt dataset (how many prompts each rule fired on):

| Rule | Title | Fired on (prompts) |
|---|---|---|
| R01 | Pin versions | 458 |
| R02 | Declare real resources | 227 |
| R03 | One routing-policy block | 7 |
| R04 | One alias or records | 22 |
| R05 | Required args, drop deprecated | 142 |
| R06 | Central KMS key | 309 |
| R07 | Encrypt CloudWatch logs | 394 |
| R08 | VPC flow logs | 347 |
| R09 | Restrict security groups | 129 |
| R10 | No public subnet IPs | 337 |
| R11 | Internal load balancer | 33 |
| R12 | Harden RDS | 80 |
| R13 | Harden S3 | 167 |
| R14 | Elastic Beanstalk service role | 50 |

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
Use only real AWS provider v5 resource types and arguments **(a)**, and declare every resource that is referenced **(b)**, so the configuration passes `terraform validate` **(c)** instead of failing on unknown types or undeclared references.

| Ref | R02 part | source | Details |
|---|---|---|---|
| a | real types + arguments | [AWS Provider docs (Registry)](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) | authoritative v5 resource/argument schema |
| b | declare referenced resources | [Resource Blocks](https://developer.hashicorp.com/terraform/language/resources/syntax) / [References to Values](https://developer.hashicorp.com/terraform/language/expressions/references) | every `type.name` referenced must be declared |
| c | validate gate | [`terraform validate`](https://developer.hashicorp.com/terraform/cli/commands/validate) | checks config internally consistent (types, refs, args) |

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

### [R06] Central KMS key  ... Fixes: AVD-AWS-0065 (underpins 0017, 0080, 0088, 0132)
Add one `aws_kms_key` with `enable_key_rotation = true` **(a)** and reference its arn from every encryptable resource **(b)**, so encryption-at-rest findings clear with a single managed, rotated key **(c)**.

| Ref | R06 part | source | Details |
|---|---|---|---|
| a | rotated CMK | AWS Provider: [`aws_kms_key`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kms_key) | `enable_key_rotation` + `deletion_window_in_days` |
| b | reuse arn for encryption | [`aws_cloudwatch_log_group`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group) · S3 SSE · [`aws_db_instance`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance) | `kms_key_id = aws_kms_key.main.arn` on each |
| c | why rotate | Aqua AVD ... [AVD-AWS-0065](https://avd.aquasec.com/misconfig/avd-aws-0065/) | one key satisfies logs 0017, S3 0088/0132, RDS 0080 |

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

### [R07] Encrypt CloudWatch logs  ... Fixes: AVD-AWS-0017
Set `kms_key_id` **(a)** and `retention_in_days >= 365` **(b)** on every CloudWatch log group.

| Ref | R07 part | source | Details |
|---|---|---|---|
| a | log encryption | AWS Provider: [`aws_cloudwatch_log_group`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group) | `kms_key_id = aws_kms_key.main.arn` |
| b | retention | Aqua AVD ... [AVD-AWS-0017](https://avd.aquasec.com/misconfig/avd-aws-0017/) | `retention_in_days >= 365` |

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

### [R08] VPC flow logs  ... Fixes: AVD-AWS-0178
Add an `aws_flow_log` (with its IAM role) per VPC **(a)**, using the correct schema with no `resource_type` argument **(b)**.

| Ref | R08 part | source | Details |
|---|---|---|---|
| a | flow log + role | AWS Provider: [`aws_flow_log`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/flow_log), [`aws_iam_role`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | `vpc_id`, `iam_role_arn`, log to encrypted group |
| b | correct schema | Aqua AVD ... [AVD-AWS-0178](https://avd.aquasec.com/misconfig/avd-aws-0178/) | [`aws_flow_log`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/flow_log) has NO `resource_type` argument |

**DSL draft**
```
WHEN  resource aws_vpc present OR error AVD-AWS-0178
MATCH aws_vpc "<n>" {}
REWRITE add aws_flow_log { vpc_id=aws_vpc.<n>.id traffic_type="ALL" log_destination_type="cloud-watch-logs"
        log_destination=<group>.arn iam_role_arn=<role>.arn } + its IAM role
```

### [S08] network traffic logging
Enables VPC flow logs with a valid resource. **Exact portion added** ... add one `aws_flow_log` plus its IAM role for EVERY VPC, using no `resource_type` argument:
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

### [R09] Restrict security groups  ... Fixes: AVD-AWS-0099, 0104, 0107, 0124
Give every rule a `description` **(a)** and replace `0.0.0.0/0` or `::/0` (ingress AND egress) with the VPC CIDR **(b)**.

| Ref | R09 part | source | Details |
|---|---|---|---|
| a | rule description | Aqua AVD ... [AVD-AWS-0099](https://avd.aquasec.com/misconfig/avd-aws-0099/)/0124 | every ingress/egress rule needs `description` |
| b | restrict CIDR | Aqua AVD ... [AVD-AWS-0107](https://avd.aquasec.com/misconfig/avd-aws-0107/) (ingress), 0104 (egress) | no public CIDR; use `aws_vpc.main.cidr_block` |

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
| a | no auto public IP | AWS Provider: [`aws_subnet`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet) · Aqua AVD ... [AVD-AWS-0164](https://avd.aquasec.com/misconfig/avd-aws-0164/) | `map_public_ip_on_launch = false` |

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
| a | not public | AWS Provider: [`aws_lb`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb)/[`aws_elb`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elb) · Aqua AVD ... [AVD-AWS-0053](https://avd.aquasec.com/misconfig/avd-aws-0053/) | `internal = true` |
| b | access logging | AWS Provider: [`aws_lb`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb) | `access_logs { bucket=... enabled=true }` |

**DSL draft**
```
WHEN  resource aws_elb/aws_lb present OR error AVD-AWS-0053
MATCH aws_elb { ...public... }
REWRITE aws_lb { internal=true access_logs{ bucket=... enabled=true } }
```

### [S11] load balancer exposure
The balancer is not public and records access logs. **Exact portion added**: set `internal = true`; enable access logging on `aws_elb`/`aws_lb`.

---

### [R12] Harden RDS  ... Fixes: AVD-AWS-0077, 0080, 0133, 0176, 0177 (also 0079, 0082)
Set `storage_encrypted`+`kms_key_id`, `iam_database_authentication_enabled`, `deletion_protection`, `performance_insights_enabled`, `backup_retention_period >= 7`, `multi_az` **(a)**; remove hardcoded passwords **(b)**.

| Ref | R12 part | source | Details |
|---|---|---|---|
| a | six hardening args | AWS Provider: [`aws_db_instance`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance) · Aqua AVD ... 0080/0077/0176/0177/0133 | encryption, IAM auth, deletion protection, PI, backups, multi-AZ |
| b | no plaintext password | Aqua AVD | never hardcode `password` |

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

### [R13] Harden S3  ... Fixes: AVD-AWS-0086, 0087, 0088, 0089, 0090, 0091, 0092, 0093, 0094, 0131, 0132
Add SEPARATE resources: public access block (all four true) **(a)**, server-side encryption with the KMS key **(b)**, versioning Enabled **(c)**, bucket logging **(d)**.

| Ref | R13 part | source | Details |
|---|---|---|---|
| a | public access block | [`aws_s3_bucket_public_access_block`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_public_access_block) · [AVD-AWS-0086](https://avd.aquasec.com/misconfig/avd-aws-0086/)/0087/0091/0093/0094 | all four settings `true` |
| b | encryption | [`aws_s3_bucket_server_side_encryption_configuration`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_server_side_encryption_configuration) · [AVD-AWS-0088](https://avd.aquasec.com/misconfig/avd-aws-0088/)/0132/0131 | KMS CMK, `sse_algorithm = "aws:kms"` |
| c | versioning | [`aws_s3_bucket_versioning`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_versioning) · [AVD-AWS-0090](https://avd.aquasec.com/misconfig/avd-aws-0090/) | `status = "Enabled"` |
| d | logging | [`aws_s3_bucket_logging`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_logging) · [AVD-AWS-0089](https://avd.aquasec.com/misconfig/avd-aws-0089/) | log to a dedicated bucket |

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

### [R14] Elastic Beanstalk service role  ... Fixes: missing-required-argument (validate)
Give the `appversion_lifecycle` block a `service_role` (IAM role arn) **(a)**, or omit the block **(b)**.

| Ref | R14 part | source | Details |
|---|---|---|---|
| a | service_role | AWS Provider: [`aws_elastic_beanstalk_application`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elastic_beanstalk_application) | `appversion_lifecycle` requires `service_role` |
| b | omit block | AWS Provider | drop `appversion_lifecycle` if no role |

**DSL draft**
```
WHEN  aws_elastic_beanstalk_application has appversion_lifecycle without service_role
MATCH appversion_lifecycle { max_age_in_days=... }
REWRITE appversion_lifecycle { service_role=aws_iam_role.eb_service.arn max_age_in_days=... }  (or remove the block)
```

### [S14] platform service role
Clears the missing-required-argument error. **Exact portion added**: include `service_role = aws_iam_role.eb_service.arn` in `appversion_lifecycle` (and declare that role), or omit the block entirely.
