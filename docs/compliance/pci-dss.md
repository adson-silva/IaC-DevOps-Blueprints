# PCI DSS - Payment Card Industry Data Security Standard

## Visão Geral

O PCI DSS (Payment Card Industry Data Security Standard) é um padrão de segurança da informação para organizações que processam, armazenam ou transmitem dados de cartões de pagamento. Este documento detalha como implementar controles PCI DSS usando Infraestrutura como Código, alinhado com o framework TOGAF.

## Estrutura PCI DSS v4.0

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PCI DSS v4.0 Requirements                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              BUILD AND MAINTAIN A SECURE NETWORK                     │    │
│  │  ┌─────────────────────────┐  ┌─────────────────────────┐           │    │
│  │  │ Req 1: Firewall Config  │  │ Req 2: Secure Defaults  │           │    │
│  │  └─────────────────────────┘  └─────────────────────────┘           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                  PROTECT CARDHOLDER DATA                             │    │
│  │  ┌─────────────────────────┐  ┌─────────────────────────┐           │    │
│  │  │ Req 3: Protect Stored   │  │ Req 4: Encrypt Trans-   │           │    │
│  │  │         Data            │  │         mission         │           │    │
│  │  └─────────────────────────┘  └─────────────────────────┘           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │           MAINTAIN A VULNERABILITY MANAGEMENT PROGRAM                │    │
│  │  ┌─────────────────────────┐  ┌─────────────────────────┐           │    │
│  │  │ Req 5: Anti-Malware     │  │ Req 6: Secure Systems   │           │    │
│  │  └─────────────────────────┘  └─────────────────────────┘           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              IMPLEMENT STRONG ACCESS CONTROL MEASURES                │    │
│  │  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐     │    │
│  │  │ Req 7: Restrict  │ │ Req 8: Identify  │ │ Req 9: Physical  │     │    │
│  │  │      Access      │ │      Access      │ │      Access      │     │    │
│  │  └──────────────────┘ └──────────────────┘ └──────────────────┘     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │           REGULARLY MONITOR AND TEST NETWORKS                        │    │
│  │  ┌─────────────────────────┐  ┌─────────────────────────┐           │    │
│  │  │ Req 10: Track Access    │  │ Req 11: Test Security   │           │    │
│  │  └─────────────────────────┘  └─────────────────────────┘           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              MAINTAIN AN INFORMATION SECURITY POLICY                 │    │
│  │  ┌─────────────────────────────────────────────────────────────────┐│    │
│  │  │         Req 12: Security Policy                                  ││    │
│  │  └─────────────────────────────────────────────────────────────────┘│    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Ambiente de Dados do Titular do Cartão (CDE)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Cardholder Data Environment (CDE)                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         Segmented Network                              │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  DMZ (Public)                                                    │  │  │
│  │  │  ┌───────────┐  ┌───────────┐                                   │  │  │
│  │  │  │    WAF    │  │    LB     │                                   │  │  │
│  │  │  └─────┬─────┘  └─────┬─────┘                                   │  │  │
│  │  └────────┼──────────────┼─────────────────────────────────────────┘  │  │
│  │           │              │                                             │  │
│  │  ┌────────┼──────────────┼─────────────────────────────────────────┐  │  │
│  │  │  App Zone│            │                                          │  │  │
│  │  │  ┌──────┴────┐  ┌─────┴─────┐  ┌───────────┐                    │  │  │
│  │  │  │ Payment   │  │  App      │  │  Token    │                    │  │  │
│  │  │  │ Gateway   │  │  Server   │  │  Service  │                    │  │  │
│  │  │  └───────────┘  └───────────┘  └───────────┘                    │  │  │
│  │  └──────────────────────┬──────────────────────────────────────────┘  │  │
│  │                         │                                              │  │
│  │  ┌──────────────────────┼──────────────────────────────────────────┐  │  │
│  │  │  Data Zone           │                                           │  │  │
│  │  │  ┌───────────────────┴───────────────────┐                      │  │  │
│  │  │  │           Encrypted Database           │                      │  │  │
│  │  │  │     (PAN, CVV não armazenado)         │                      │  │  │
│  │  │  └───────────────────────────────────────┘                      │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Requisito 1: Instalar e Manter Controles de Segurança de Rede

### Exemplo Terraform - Segmentação de Rede

```hcl
#------------------------------------------------------------------------------
# Req 1.2: Configuração de firewall/router restritiva
# Req 1.3: Acesso à rede do CDE é restrito
# Req 1.4: Conexões entre redes confiáveis e não confiáveis
#------------------------------------------------------------------------------

# VPC para CDE (Cardholder Data Environment)
resource "aws_vpc" "cde" {
  cidr_block           = "10.100.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name          = "pci-cde-vpc"
    PCIRequirement = "1.3"
    Environment   = "cde"
  }
}

# Subnets segmentadas
resource "aws_subnet" "dmz" {
  count             = 2
  vpc_id            = aws_vpc.cde.id
  cidr_block        = "10.100.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name           = "pci-dmz-${count.index + 1}"
    PCIRequirement = "1.3"
    Tier           = "dmz"
  }
}

resource "aws_subnet" "app" {
  count             = 2
  vpc_id            = aws_vpc.cde.id
  cidr_block        = "10.100.${count.index + 11}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name           = "pci-app-${count.index + 1}"
    PCIRequirement = "1.3"
    Tier           = "application"
  }
}

resource "aws_subnet" "data" {
  count             = 2
  vpc_id            = aws_vpc.cde.id
  cidr_block        = "10.100.${count.index + 21}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name           = "pci-data-${count.index + 1}"
    PCIRequirement = "1.3"
    Tier           = "database"
  }
}

# NACLs restritivas
resource "aws_network_acl" "data" {
  vpc_id     = aws_vpc.cde.id
  subnet_ids = aws_subnet.data[*].id

  # Permitir apenas da subnet de aplicação
  ingress {
    rule_no    = 100
    action     = "allow"
    protocol   = "tcp"
    from_port  = 5432
    to_port    = 5432
    cidr_block = "10.100.11.0/24"
  }

  ingress {
    rule_no    = 101
    action     = "allow"
    protocol   = "tcp"
    from_port  = 5432
    to_port    = 5432
    cidr_block = "10.100.12.0/24"
  }

  # Deny all other ingress
  ingress {
    rule_no    = 200
    action     = "deny"
    protocol   = "-1"
    from_port  = 0
    to_port    = 0
    cidr_block = "0.0.0.0/0"
  }

  # Allow response traffic
  egress {
    rule_no    = 100
    action     = "allow"
    protocol   = "tcp"
    from_port  = 1024
    to_port    = 65535
    cidr_block = "10.100.0.0/16"
  }

  egress {
    rule_no    = 200
    action     = "deny"
    protocol   = "-1"
    from_port  = 0
    to_port    = 0
    cidr_block = "0.0.0.0/0"
  }

  tags = {
    Name           = "pci-data-nacl"
    PCIRequirement = "1.3"
  }
}

# Security Groups com documentação de regras
resource "aws_security_group" "payment_app" {
  name        = "pci-payment-app-sg"
  description = "PCI Req 1.2 - Security group for payment application"
  vpc_id      = aws_vpc.cde.id

  # Regra 1: HTTPS do Load Balancer
  ingress {
    description     = "HTTPS from ALB - Business justification: Web traffic"
    from_port       = 8443
    to_port         = 8443
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # Egress restrito
  egress {
    description     = "PostgreSQL to database tier"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.database.id]
  }

  egress {
    description = "HTTPS to tokenization service"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.tokenization_service_cidr]
  }

  tags = {
    Name           = "pci-payment-app-sg"
    PCIRequirement = "1.2"
    Documentation  = "s3://pci-docs/firewall-rules/payment-app.md"
  }
}
```

---

## Requisito 3: Proteger Dados Armazenados do Titular do Cartão

### Tokenização e Criptografia

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Tokenization Flow                                    │
│                                                                              │
│  ┌─────────────┐     ┌─────────────────┐     ┌─────────────────────────┐   │
│  │   Client    │────▶│ Payment Gateway │────▶│   Token Service (HSM)   │   │
│  │             │     │                 │     │                         │   │
│  └─────────────┘     └─────────────────┘     └─────────────────────────┘   │
│         │                    │                         │                    │
│         │                    │                         │                    │
│         ▼                    ▼                         ▼                    │
│  ┌─────────────┐     ┌─────────────────┐     ┌─────────────────────────┐   │
│  │    PAN:     │     │    Token:       │     │   Vault (encrypted):    │   │
│  │ 4111...1111 │     │ tok_abc123xyz   │     │   PAN ↔ Token mapping   │   │
│  └─────────────┘     └─────────────────┘     └─────────────────────────┘   │
│                                                                              │
│  ╔═══════════════════════════════════════════════════════════════════════╗  │
│  ║  NUNCA ARMAZENE:                                                       ║  │
│  ║  • CVV/CVC (Magnetic stripe data)                                     ║  │
│  ║  • PIN/PIN Block                                                       ║  │
│  ║  • Full track data                                                     ║  │
│  ╚═══════════════════════════════════════════════════════════════════════╝  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - Criptografia de Dados

```hcl
#------------------------------------------------------------------------------
# Req 3.4: Render PAN illegible anywhere it is stored
# Req 3.5: Procedures to protect keys used to secure stored CHD
# Req 3.6: Key management procedures
#------------------------------------------------------------------------------

# KMS Key para criptografia de dados do titular do cartão
resource "aws_kms_key" "cardholder_data" {
  description              = "PCI Req 3.5 - KMS key for cardholder data encryption"
  deletion_window_in_days  = 30
  enable_key_rotation      = true
  multi_region             = false
  
  # Política de acesso restritiva
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "RootAccess"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "KeyAdministrators"
        Effect = "Allow"
        Principal = {
          AWS = var.key_admin_role_arns
        }
        Action = [
          "kms:Create*",
          "kms:Describe*",
          "kms:Enable*",
          "kms:List*",
          "kms:Put*",
          "kms:Update*",
          "kms:Revoke*",
          "kms:Disable*",
          "kms:Get*",
          "kms:Delete*",
          "kms:ScheduleKeyDeletion",
          "kms:CancelKeyDeletion"
        ]
        Resource = "*"
      },
      {
        Sid    = "KeyUsers"
        Effect = "Allow"
        Principal = {
          AWS = var.payment_app_role_arns
        }
        Action = [
          "kms:Encrypt",
          "kms:Decrypt",
          "kms:ReEncrypt*",
          "kms:GenerateDataKey*",
          "kms:DescribeKey"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "kms:ViaService" = "rds.${var.region}.amazonaws.com"
          }
        }
      },
      {
        Sid    = "DenyDirectDecrypt"
        Effect = "Deny"
        Principal = "*"
        Action = [
          "kms:Decrypt"
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "kms:ViaService" = [
              "rds.${var.region}.amazonaws.com",
              "secretsmanager.${var.region}.amazonaws.com"
            ]
          }
          ArnNotEquals = {
            "aws:PrincipalArn" = var.break_glass_role_arn
          }
        }
      }
    ]
  })

  tags = {
    Name           = "pci-cardholder-data-key"
    PCIRequirement = "3.5"
    DataType       = "cardholder-data"
  }
}

# RDS com criptografia
resource "aws_db_instance" "cardholder" {
  identifier           = "pci-cardholder-db"
  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = "db.r6g.large"
  allocated_storage    = 100
  max_allocated_storage = 500
  
  # Criptografia obrigatória
  storage_encrypted = true
  kms_key_id        = aws_kms_key.cardholder_data.arn
  
  # Rede isolada
  db_subnet_group_name   = aws_db_subnet_group.cardholder.name
  vpc_security_group_ids = [aws_security_group.database.id]
  publicly_accessible    = false
  
  # Backup
  backup_retention_period = 35
  backup_window           = "03:00-04:00"
  maintenance_window      = "sat:04:00-sat:05:00"
  
  # Logging
  enabled_cloudwatch_logs_exports = [
    "postgresql",
    "upgrade"
  ]
  
  # Performance Insights
  performance_insights_enabled    = true
  performance_insights_kms_key_id = aws_kms_key.cardholder_data.arn
  
  # Delete protection
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "pci-cardholder-final-${formatdate("YYYY-MM-DD-hh-mm", timestamp())}"

  tags = {
    Name           = "pci-cardholder-db"
    PCIRequirement = "3.4"
    DataType       = "cardholder-data"
  }
}

# Secrets Manager para credenciais do banco
resource "aws_secretsmanager_secret" "db_credentials" {
  name        = "pci/cardholder-db/credentials"
  description = "PCI Req 3.5 - Database credentials"
  kms_key_id  = aws_kms_key.cardholder_data.arn

  tags = {
    PCIRequirement = "3.5"
  }
}

resource "aws_secretsmanager_secret_rotation" "db_credentials" {
  secret_id           = aws_secretsmanager_secret.db_credentials.id
  rotation_lambda_arn = aws_lambda_function.rotate_db_credentials.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

---

## Requisito 4: Proteger Dados do Titular do Cartão com Criptografia Forte

```hcl
#------------------------------------------------------------------------------
# Req 4.1: Use strong cryptography and security protocols to safeguard 
#          sensitive cardholder data during transmission over open, public networks
# Req 4.2: Never send unprotected PANs by end-user messaging technologies
#------------------------------------------------------------------------------

# ALB com TLS 1.2+
resource "aws_lb" "payment" {
  name               = "pci-payment-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.dmz[*].id

  drop_invalid_header_fields = true
  enable_deletion_protection = true

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "pci-payment-alb"
    enabled = true
  }

  tags = {
    Name           = "pci-payment-alb"
    PCIRequirement = "4.1"
  }
}

# Listener HTTPS com política TLS restritiva
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.payment.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.payment.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.payment.arn
  }

  tags = {
    PCIRequirement = "4.1"
  }
}

# Redirect HTTP to HTTPS
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.payment.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# WAF para proteção adicional
resource "aws_wafv2_web_acl" "payment" {
  name        = "pci-payment-waf"
  description = "PCI WAF rules for payment processing"
  scope       = "REGIONAL"

  default_action {
    allow {}
  }

  # Regra 1: Rate limiting
  rule {
    name     = "RateLimit"
    priority = 1

    override_action {
      none {}
    }

    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
      sampled_requests_enabled   = true
    }
  }

  # Regra 2: SQL Injection
  rule {
    name     = "SQLInjection"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLInjection"
      sampled_requests_enabled   = true
    }
  }

  # Regra 3: Known Bad Inputs
  rule {
    name     = "KnownBadInputs"
    priority = 3

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "KnownBadInputs"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "pci-payment-waf"
    sampled_requests_enabled   = true
  }

  tags = {
    Name           = "pci-payment-waf"
    PCIRequirement = "4.1"
  }
}

resource "aws_wafv2_web_acl_association" "payment" {
  resource_arn = aws_lb.payment.arn
  web_acl_arn  = aws_wafv2_web_acl.payment.arn
}
```

---

## Requisito 10: Registrar e Monitorar Todo o Acesso

```hcl
#------------------------------------------------------------------------------
# Req 10.1: Implement audit trails to link all access to system components
# Req 10.2: Implement automated audit trails for all system components
# Req 10.3: Record audit trail entries for all system components
# Req 10.4: Synchronize all critical system clocks
# Req 10.5: Secure audit trails so they cannot be altered
#------------------------------------------------------------------------------

# CloudTrail com integridade habilitada
resource "aws_cloudtrail" "pci" {
  name                          = "pci-audit-trail"
  s3_bucket_name                = aws_s3_bucket.audit_logs.id
  s3_key_prefix                 = "cloudtrail"
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_logging                = true
  enable_log_file_validation    = true  # Req 10.5
  kms_key_id                    = aws_kms_key.audit_logs.arn

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::pci-*"]
    }

    data_resource {
      type   = "AWS::Lambda::Function"
      values = ["arn:aws:lambda"]
    }
  }

  insight_selector {
    insight_type = "ApiCallRateInsight"
  }

  insight_selector {
    insight_type = "ApiErrorRateInsight"
  }

  tags = {
    Name           = "pci-audit-trail"
    PCIRequirement = "10.1"
  }
}

# S3 Bucket para logs com Object Lock (WORM)
resource "aws_s3_bucket" "audit_logs" {
  bucket = "pci-audit-logs-${data.aws_caller_identity.current.account_id}"
  
  object_lock_enabled = true

  tags = {
    Name           = "pci-audit-logs"
    PCIRequirement = "10.5"
  }
}

resource "aws_s3_bucket_object_lock_configuration" "audit_logs" {
  bucket = aws_s3_bucket.audit_logs.id

  rule {
    default_retention {
      mode = "COMPLIANCE"
      years = 1
    }
  }
}

resource "aws_s3_bucket_versioning" "audit_logs" {
  bucket = aws_s3_bucket.audit_logs.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "audit_logs" {
  bucket = aws_s3_bucket.audit_logs.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.audit_logs.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

# Lifecycle para retenção mínima de 1 ano (PCI Req 10.7)
resource "aws_s3_bucket_lifecycle_configuration" "audit_logs" {
  bucket = aws_s3_bucket.audit_logs.id

  rule {
    id     = "audit-log-retention"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    # Não deletar antes de 1 ano
    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}

# CloudWatch Log Group para logs de aplicação
resource "aws_cloudwatch_log_group" "payment_app" {
  name              = "/pci/payment-app"
  retention_in_days = 365  # Req 10.7 - 1 ano mínimo
  kms_key_id        = aws_kms_key.audit_logs.arn

  tags = {
    Name           = "pci-payment-app-logs"
    PCIRequirement = "10.2"
  }
}

# Alertas para eventos críticos
resource "aws_cloudwatch_metric_alarm" "failed_login_attempts" {
  alarm_name          = "pci-failed-login-attempts"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "FailedLoginAttempts"
  namespace           = "PCI/Security"
  period              = 300
  statistic           = "Sum"
  threshold           = 5
  alarm_description   = "PCI Req 10 - Multiple failed login attempts detected"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]

  tags = {
    PCIRequirement = "10.6"
  }
}

# NTP para sincronização de tempo (Req 10.4)
resource "aws_vpc_dhcp_options" "cde" {
  domain_name_servers = ["AmazonProvidedDNS"]
  ntp_servers         = ["169.254.169.123"]  # Amazon Time Sync Service

  tags = {
    Name           = "pci-dhcp-options"
    PCIRequirement = "10.4"
  }
}

resource "aws_vpc_dhcp_options_association" "cde" {
  vpc_id          = aws_vpc.cde.id
  dhcp_options_id = aws_vpc_dhcp_options.cde.id
}
```

---

## Requisito 11: Testar Segurança de Sistemas e Redes Regularmente

```hcl
#------------------------------------------------------------------------------
# Req 11.2: Run internal and external network vulnerability scans
# Req 11.3: Implement a methodology for penetration testing
# Req 11.4: Detect and respond to intrusions and changes
#------------------------------------------------------------------------------

# Inspector para vulnerability scanning
resource "aws_inspector2_enabler" "cde" {
  account_ids    = [data.aws_caller_identity.current.account_id]
  resource_types = ["EC2", "ECR", "LAMBDA"]
}

# Config Rules para compliance contínuo
resource "aws_config_config_rule" "pci_checks" {
  for_each = toset([
    "ec2-instance-managed-by-systems-manager",
    "encrypted-volumes",
    "rds-storage-encrypted",
    "s3-bucket-ssl-requests-only",
    "vpc-flow-logs-enabled",
    "cloudtrail-enabled"
  ])

  name = "pci-${each.key}"

  source {
    owner             = "AWS"
    source_identifier = upper(replace(each.key, "-", "_"))
  }

  tags = {
    PCIRequirement = "11.2"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

# GuardDuty para detecção de intrusão
resource "aws_guardduty_detector" "cde" {
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
    Name           = "pci-guardduty"
    PCIRequirement = "11.4"
  }
}

# Security Hub com PCI DSS standard
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "pci" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/pci-dss/v/3.2.1"
  depends_on    = [aws_securityhub_account.main]
}

# FIM (File Integrity Monitoring) via SSM
resource "aws_ssm_document" "fim" {
  name          = "pci-file-integrity-monitor"
  document_type = "Command"

  content = jsonencode({
    schemaVersion = "2.2"
    description   = "PCI Req 11.5 - File Integrity Monitoring"
    parameters    = {}
    mainSteps = [
      {
        action = "aws:runShellScript"
        name   = "checkFileIntegrity"
        inputs = {
          runCommand = [
            "#!/bin/bash",
            "aide --check",
            "if [ $? -ne 0 ]; then",
            "  aws sns publish --topic-arn ${aws_sns_topic.security_alerts.arn} --message 'PCI Alert: File integrity check failed'",
            "fi"
          ]
        }
      }
    ]
  })

  tags = {
    PCIRequirement = "11.5"
  }
}

resource "aws_ssm_association" "fim" {
  name                = aws_ssm_document.fim.name
  association_name    = "pci-fim-daily"
  schedule_expression = "cron(0 2 * * ? *)"

  targets {
    key    = "tag:PCIScope"
    values = ["true"]
  }
}
```

---

## Matriz de Conformidade PCI DSS

| Requisito | Descrição | Implementação IaC | Status |
|-----------|-----------|-------------------|--------|
| 1.2 | Firewall configuration | Security Groups, NACLs | ✅ |
| 1.3 | CDE network segmentation | VPC, Subnets isoladas | ✅ |
| 3.4 | Render PAN illegible | KMS, encryption | ✅ |
| 3.5 | Key protection | KMS policies | ✅ |
| 4.1 | Strong cryptography | TLS 1.2+, WAF | ✅ |
| 6.4 | Change detection | Config, GuardDuty | ✅ |
| 8.3 | MFA | IAM MFA policies | ✅ |
| 10.1 | Audit trails | CloudTrail | ✅ |
| 10.5 | Secure audit trails | Object Lock, KMS | ✅ |
| 10.7 | Retention 1 year | Lifecycle policies | ✅ |
| 11.2 | Vulnerability scans | Inspector | ✅ |
| 11.4 | Intrusion detection | GuardDuty | ✅ |
| 11.5 | File integrity | SSM, AIDE | ✅ |

---

## Referências

- [PCI DSS v4.0](https://www.pcisecuritystandards.org/document_library/)
- [AWS PCI DSS Compliance](https://aws.amazon.com/compliance/pci-dss-level-1-faqs/)
- [Azure PCI DSS Blueprint](https://docs.microsoft.com/azure/governance/blueprints/samples/pci-dss-3.2.1/)
- [GCP PCI DSS Compliance](https://cloud.google.com/security/compliance/pci-dss)
- [TOGAF Standard](https://www.opengroup.org/togaf)
