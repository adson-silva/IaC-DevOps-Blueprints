# NIST Cybersecurity Framework - Implementação via IaC

## Visão Geral

O NIST Cybersecurity Framework (CSF) é um conjunto de diretrizes e melhores práticas para gerenciar riscos de segurança cibernética. Este documento detalha como implementar os controles NIST usando Infraestrutura como Código, alinhado com o framework TOGAF.

## Estrutura do Framework NIST

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        NIST Cybersecurity Framework                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────┐│
│  │  IDENTIFY  │  │  PROTECT   │  │   DETECT   │  │  RESPOND   │  │RECOVER ││
│  │    (ID)    │  │    (PR)    │  │    (DE)    │  │    (RS)    │  │  (RC)  ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘  └────────┘│
│       │               │               │               │              │      │
│       ▼               ▼               ▼               ▼              ▼      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────┐│
│  │ Asset Mgmt │  │ Access     │  │ Anomalies  │  │ Response   │  │Recovery││
│  │ Business   │  │ Control    │  │ Continuous │  │ Planning   │  │Planning││
│  │ Environment│  │ Data Sec   │  │ Monitoring │  │ Analysis   │  │Improve ││
│  │ Governance │  │ Protective │  │ Detection  │  │ Mitigation │  │Comms   ││
│  │ Risk Assess│  │ Technology │  │ Process    │  │ Comms      │  │        ││
│  │ Risk Mgmt  │  │ Maintenance│  │            │  │            │  │        ││
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘  └────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Mapeamento TOGAF para NIST

| Camada TOGAF | Função NIST | Controles Principais |
|--------------|-------------|---------------------|
| Business Architecture | Identify (ID) | ID.AM, ID.BE, ID.GV |
| Data Architecture | Protect (PR) | PR.DS, PR.IP |
| Application Architecture | Detect (DE) | DE.AE, DE.CM, DE.DP |
| Technology Architecture | Respond/Recover | RS.*, RC.* |

---

## 1. IDENTIFY (ID) - Identificar

### ID.AM - Gestão de Ativos

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Asset Management                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      Asset Inventory                                   │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │   Hardware  │  │  Software   │  │    Data     │  │   Users     │  │  │
│  │  │   Assets    │  │   Assets    │  │   Assets    │  │   Assets    │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    Classification & Tagging                            │  │
│  │          (Criticality, Data Classification, Owner)                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - Gestão de Ativos (AWS)

```hcl
#------------------------------------------------------------------------------
# ID.AM-1: Inventário de dispositivos físicos e sistemas
# ID.AM-2: Inventário de plataformas de software e aplicações
#------------------------------------------------------------------------------

# Política de Tags obrigatórias
resource "aws_organizations_policy" "tagging_policy" {
  name        = "nist-mandatory-tags"
  description = "NIST ID.AM - Mandatory tagging policy"
  type        = "TAG_POLICY"

  content = jsonencode({
    tags = {
      Owner = {
        tag_key = {
          @@assign = "Owner"
        }
        enforced_for = {
          @@assign = ["ec2:instance", "rds:db", "s3:bucket"]
        }
      }
      DataClassification = {
        tag_key = {
          @@assign = "DataClassification"
        }
        tag_value = {
          @@assign = ["public", "internal", "confidential", "restricted"]
        }
        enforced_for = {
          @@assign = ["s3:bucket", "rds:db", "dynamodb:table"]
        }
      }
      Criticality = {
        tag_key = {
          @@assign = "Criticality"
        }
        tag_value = {
          @@assign = ["low", "medium", "high", "critical"]
        }
        enforced_for = {
          @@assign = ["ec2:instance", "rds:db", "lambda:function"]
        }
      }
      NISTControl = {
        tag_key = {
          @@assign = "NISTControl"
        }
      }
    }
  })
}

# AWS Config para inventário automático
resource "aws_config_configuration_recorder" "main" {
  name     = "nist-config-recorder"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported = true
    include_global_resource_types = true
  }
}

# Config Rules para validação de tags
resource "aws_config_config_rule" "required_tags" {
  name = "nist-required-tags"

  source {
    owner             = "AWS"
    source_identifier = "REQUIRED_TAGS"
  }

  input_parameters = jsonencode({
    tag1Key   = "Owner"
    tag2Key   = "DataClassification"
    tag3Key   = "Criticality"
    tag4Key   = "Environment"
  })

  depends_on = [aws_config_configuration_recorder.main]
}

# SSM Inventory para software
resource "aws_ssm_resource_data_sync" "inventory" {
  name = "nist-inventory-sync"

  s3_destination {
    bucket_name = aws_s3_bucket.inventory.id
    region      = var.region
    kms_key_arn = aws_kms_key.inventory.arn
  }
}

resource "aws_ssm_association" "inventory" {
  name             = "AWS-GatherSoftwareInventory"
  association_name = "nist-software-inventory"

  schedule_expression = "rate(12 hours)"

  targets {
    key    = "InstanceIds"
    values = ["*"]
  }

  parameters = {
    applications                = "Enabled"
    awsComponents               = "Enabled"
    networkConfig               = "Enabled"
    windowsUpdates              = "Enabled"
    instanceDetailedInformation = "Enabled"
    services                    = "Enabled"
  }
}
```

---

## 2. PROTECT (PR) - Proteger

### PR.AC - Controle de Acesso

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Access Control                                       │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    Identity Management                                 │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │    SSO      │  │    MFA      │  │   RBAC      │                    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                   Network Access Control                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │  Firewall   │  │    VPN      │  │   Bastion   │  │   Zero      │  │  │
│  │  │   Rules     │  │             │  │    Host     │  │   Trust     │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - Controle de Acesso

```hcl
#------------------------------------------------------------------------------
# PR.AC-1: Identidades e credenciais são gerenciadas
# PR.AC-3: Acesso remoto é gerenciado
# PR.AC-4: Permissões de acesso são gerenciadas com least privilege
#------------------------------------------------------------------------------

# IAM Password Policy
resource "aws_iam_account_password_policy" "strict" {
  minimum_password_length        = 14
  require_lowercase_characters   = true
  require_numbers                = true
  require_uppercase_characters   = true
  require_symbols                = true
  allow_users_to_change_password = true
  max_password_age               = 90
  password_reuse_prevention      = 24
}

# MFA obrigatório para acesso ao console
resource "aws_iam_policy" "require_mfa" {
  name        = "nist-require-mfa"
  description = "NIST PR.AC-1 - Require MFA for all actions"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowViewAccountInfo"
        Effect = "Allow"
        Action = [
          "iam:GetAccountPasswordPolicy",
          "iam:ListVirtualMFADevices"
        ]
        Resource = "*"
      },
      {
        Sid    = "AllowManageOwnMFA"
        Effect = "Allow"
        Action = [
          "iam:CreateVirtualMFADevice",
          "iam:DeleteVirtualMFADevice",
          "iam:EnableMFADevice",
          "iam:ListMFADevices",
          "iam:ResyncMFADevice"
        ]
        Resource = [
          "arn:aws:iam::*:mfa/$${aws:username}",
          "arn:aws:iam::*:user/$${aws:username}"
        ]
      },
      {
        Sid    = "DenyAllExceptListedIfNoMFA"
        Effect = "Deny"
        NotAction = [
          "iam:CreateVirtualMFADevice",
          "iam:EnableMFADevice",
          "iam:GetUser",
          "iam:ListMFADevices",
          "iam:ListVirtualMFADevices",
          "iam:ResyncMFADevice",
          "sts:GetSessionToken"
        ]
        Resource = "*"
        Condition = {
          BoolIfExists = {
            "aws:MultiFactorAuthPresent" = "false"
          }
        }
      }
    ]
  })
}

# Session Manager para acesso seguro (substitui bastion)
resource "aws_iam_role" "ssm_access" {
  name = "nist-ssm-access-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    NISTControl = "PR.AC-3"
  }
}

resource "aws_iam_role_policy_attachment" "ssm_managed" {
  role       = aws_iam_role.ssm_access.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# VPC Endpoints para Session Manager (sem internet)
resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    NISTControl = "PR.AC-3"
  }
}

resource "aws_vpc_endpoint" "ssmmessages" {
  vpc_id              = var.vpc_id
  service_name        = "com.amazonaws.${var.region}.ssmmessages"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    NISTControl = "PR.AC-3"
  }
}

# Security Group restritivo para endpoints
resource "aws_security_group" "vpc_endpoints" {
  name        = "nist-vpc-endpoints-sg"
  description = "NIST PR.AC - Security group for VPC endpoints"
  vpc_id      = var.vpc_id

  ingress {
    description = "HTTPS from VPC"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    NISTControl = "PR.AC-3"
  }
}
```

### PR.DS - Segurança de Dados

```hcl
#------------------------------------------------------------------------------
# PR.DS-1: Dados em repouso são protegidos
# PR.DS-2: Dados em trânsito são protegidos
# PR.DS-5: Proteção contra vazamento de dados
#------------------------------------------------------------------------------

# S3 com criptografia e controles de acesso
resource "aws_s3_bucket" "protected" {
  bucket = "${var.project}-nist-protected"

  tags = {
    NISTControl        = "PR.DS-1"
    DataClassification = "confidential"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "protected" {
  bucket = aws_s3_bucket.protected.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.data.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_versioning" "protected" {
  bucket = aws_s3_bucket.protected.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "protected" {
  bucket = aws_s3_bucket.protected.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Macie para detecção de dados sensíveis
resource "aws_macie2_account" "main" {
  finding_publishing_frequency = "FIFTEEN_MINUTES"
  status                       = "ENABLED"
}

resource "aws_macie2_classification_job" "sensitive_data" {
  name                       = "nist-sensitive-data-scan"
  job_type                   = "SCHEDULED"
  initial_run                = true

  s3_job_definition {
    bucket_definitions {
      account_id = data.aws_caller_identity.current.account_id
      buckets    = [aws_s3_bucket.protected.id]
    }
  }

  schedule_frequency {
    daily_schedule = true
  }

  sampling_percentage = 100

  tags = {
    NISTControl = "PR.DS-5"
  }
}

# ALB com TLS 1.2+ obrigatório
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }

  tags = {
    NISTControl = "PR.DS-2"
  }
}
```

---

## 3. DETECT (DE) - Detectar

### DE.CM - Monitoramento Contínuo

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Continuous Monitoring                                  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                       Data Sources                                     │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │  │
│  │  │CloudTrail│ │VPC Flow  │ │GuardDuty │ │   WAF    │ │  Config  │    │  │
│  │  │   Logs   │ │  Logs    │ │ Findings │ │   Logs   │ │  Events  │    │  │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘    │  │
│  └───────┼────────────┼────────────┼────────────┼────────────┼──────────┘  │
│          └────────────┴────────────┴────────────┴────────────┘             │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      Security Hub / SIEM                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │ Aggregation │  │  Analysis   │  │  Alerting   │                    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - Detecção

```hcl
#------------------------------------------------------------------------------
# DE.CM-1: A rede é monitorada para detectar eventos de segurança
# DE.CM-3: Atividades de pessoal são monitoradas
# DE.CM-7: Monitoramento de pessoal, conexões e dispositivos não autorizados
#------------------------------------------------------------------------------

# GuardDuty para detecção de ameaças
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs {
      enable = true
    }
    kubernetes {
      audit_logs {
        enable = true
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          enable = true
        }
      }
    }
  }

  tags = {
    NISTControl = "DE.CM-1"
  }
}

# Security Hub para agregação
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "nist" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/nist-800-53/v/5.0.0"
  depends_on    = [aws_securityhub_account.main]
}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/cis-aws-foundations-benchmark/v/1.4.0"
  depends_on    = [aws_securityhub_account.main]
}

# CloudWatch Alarms para eventos críticos
resource "aws_cloudwatch_log_metric_filter" "unauthorized_api" {
  name           = "nist-unauthorized-api-calls"
  pattern        = "{ ($.errorCode = \"*UnauthorizedAccess*\") || ($.errorCode = \"AccessDenied*\") }"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name      = "UnauthorizedAPICalls"
    namespace = "NIST/Security"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "unauthorized_api" {
  alarm_name          = "nist-unauthorized-api-calls"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "UnauthorizedAPICalls"
  namespace           = "NIST/Security"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "NIST DE.CM-1 - Unauthorized API calls detected"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]

  tags = {
    NISTControl = "DE.CM-1"
  }
}

# EventBridge para eventos de segurança
resource "aws_cloudwatch_event_rule" "security_events" {
  name        = "nist-security-events"
  description = "NIST DE.CM - Capture security events"

  event_pattern = jsonencode({
    source = ["aws.guardduty", "aws.securityhub", "aws.config"]
    detail-type = [
      "GuardDuty Finding",
      "Security Hub Findings - Imported",
      "Config Rules Compliance Change"
    ]
  })

  tags = {
    NISTControl = "DE.CM-1"
  }
}

resource "aws_cloudwatch_event_target" "security_events" {
  rule      = aws_cloudwatch_event_rule.security_events.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.security_alerts.arn
}

# Inspector para vulnerability scanning
resource "aws_inspector2_enabler" "main" {
  account_ids    = [data.aws_caller_identity.current.account_id]
  resource_types = ["EC2", "ECR", "LAMBDA"]
}
```

---

## 4. RESPOND (RS) - Responder

### RS.RP - Planejamento de Resposta

```hcl
#------------------------------------------------------------------------------
# RS.RP-1: Plano de resposta é executado durante ou após um evento
# RS.AN-1: Notificações de sistemas de detecção são investigadas
# RS.MI-1: Incidentes são contidos
#------------------------------------------------------------------------------

# SNS Topic para alertas
resource "aws_sns_topic" "security_alerts" {
  name              = "nist-security-alerts"
  kms_master_key_id = aws_kms_key.sns.id

  tags = {
    NISTControl = "RS.CO-2"
  }
}

resource "aws_sns_topic_subscription" "security_team" {
  topic_arn = aws_sns_topic.security_alerts.arn
  protocol  = "email"
  endpoint  = var.security_team_email
}

# Lambda para resposta automática
resource "aws_lambda_function" "incident_response" {
  function_name = "nist-incident-response"
  role          = aws_iam_role.incident_response.arn
  handler       = "index.handler"
  runtime       = "python3.11"
  timeout       = 300

  filename         = data.archive_file.incident_response.output_path
  source_code_hash = data.archive_file.incident_response.output_base64sha256

  environment {
    variables = {
      SNS_TOPIC_ARN = aws_sns_topic.security_alerts.arn
      QUARANTINE_SG = aws_security_group.quarantine.id
    }
  }

  tags = {
    NISTControl = "RS.MI-1"
  }
}

# Security Group para quarentena
resource "aws_security_group" "quarantine" {
  name        = "nist-quarantine-sg"
  description = "NIST RS.MI-1 - Quarantine security group"
  vpc_id      = var.vpc_id

  # Sem regras de ingress ou egress = isolamento total
  
  tags = {
    NISTControl = "RS.MI-1"
    Purpose     = "incident-response-quarantine"
  }
}

# EventBridge para trigger de resposta
resource "aws_cloudwatch_event_rule" "high_severity_findings" {
  name        = "nist-high-severity-findings"
  description = "NIST RS.AN-1 - Trigger on high severity findings"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      severity = [{ numeric = [">=", 7] }]
    }
  })

  tags = {
    NISTControl = "RS.AN-1"
  }
}

resource "aws_cloudwatch_event_target" "incident_response" {
  rule      = aws_cloudwatch_event_rule.high_severity_findings.name
  target_id = "TriggerIncidentResponse"
  arn       = aws_lambda_function.incident_response.arn
}

resource "aws_lambda_permission" "eventbridge" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.incident_response.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.high_severity_findings.arn
}
```

---

## 5. RECOVER (RC) - Recuperar

### RC.RP - Planejamento de Recuperação

```hcl
#------------------------------------------------------------------------------
# RC.RP-1: Plano de recuperação é executado durante ou após um evento
# RC.IM-1: Planos de recuperação incorporam lições aprendidas
#------------------------------------------------------------------------------

# Backup automatizado
resource "aws_backup_vault" "main" {
  name        = "nist-backup-vault"
  kms_key_arn = aws_kms_key.backup.arn

  tags = {
    NISTControl = "RC.RP-1"
  }
}

resource "aws_backup_plan" "main" {
  name = "nist-backup-plan"

  rule {
    rule_name         = "daily-backup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 5 ? * * *)"

    lifecycle {
      delete_after = 35
    }

    copy_action {
      destination_vault_arn = aws_backup_vault.dr.arn
      lifecycle {
        delete_after = 90
      }
    }
  }

  rule {
    rule_name         = "weekly-backup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 5 ? * SAT *)"

    lifecycle {
      cold_storage_after = 30
      delete_after       = 365
    }
  }

  tags = {
    NISTControl = "RC.RP-1"
  }
}

resource "aws_backup_selection" "main" {
  name         = "nist-backup-selection"
  plan_id      = aws_backup_plan.main.id
  iam_role_arn = aws_iam_role.backup.arn

  selection_tag {
    type  = "STRINGEQUALS"
    key   = "Backup"
    value = "true"
  }
}

# DR Vault em outra região
resource "aws_backup_vault" "dr" {
  provider    = aws.dr
  name        = "nist-backup-vault-dr"
  kms_key_arn = aws_kms_key.backup_dr.arn

  tags = {
    NISTControl = "RC.RP-1"
    Purpose     = "disaster-recovery"
  }
}
```

---

## Matriz de Conformidade NIST

| Função | Categoria | Controle | Implementação | Status |
|--------|-----------|----------|---------------|--------|
| ID | ID.AM-1 | Inventário físico | Config, SSM | ✅ |
| ID | ID.AM-2 | Inventário software | SSM Inventory | ✅ |
| ID | ID.RA-1 | Vulnerabilidades | Inspector | ✅ |
| PR | PR.AC-1 | Identidades | IAM, MFA | ✅ |
| PR | PR.AC-3 | Acesso remoto | Session Manager | ✅ |
| PR | PR.DS-1 | Dados em repouso | KMS, encryption | ✅ |
| PR | PR.DS-2 | Dados em trânsito | TLS 1.2+ | ✅ |
| DE | DE.CM-1 | Monitoramento rede | GuardDuty, VPC Logs | ✅ |
| DE | DE.CM-7 | Não autorizados | GuardDuty | ✅ |
| RS | RS.RP-1 | Plano resposta | Lambda, EventBridge | ✅ |
| RS | RS.MI-1 | Contenção | Quarantine SG | ✅ |
| RC | RC.RP-1 | Plano recuperação | AWS Backup | ✅ |

---

## Referências

- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [NIST SP 800-53](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [AWS Security Hub NIST Standard](https://docs.aws.amazon.com/securityhub/latest/userguide/nist-standard.html)
- [TOGAF Standard](https://www.opengroup.org/togaf)
