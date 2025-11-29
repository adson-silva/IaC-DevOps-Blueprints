# Guia Completo AWS - Infraestrutura como Código

## Visão Geral

Este documento fornece guias detalhados para configuração de serviços AWS utilizando práticas de IaC, alinhados com o AWS Well-Architected Framework e os princípios TOGAF.

## Arquitetura de Referência AWS

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS Account                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                           VPC (10.0.0.0/16)                          │    │
│  │  ┌─────────────────────────────────────────────────────────────┐    │    │
│  │  │                    Availability Zone A                       │    │    │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │    │    │
│  │  │  │Public Subnet│  │Private Subnet│  │Database Subnet│        │    │    │
│  │  │  │ 10.0.1.0/24│  │ 10.0.11.0/24│  │ 10.0.21.0/24│          │    │    │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘          │    │    │
│  │  └─────────────────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────────────────┐    │    │
│  │  │                    Availability Zone B                       │    │    │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │    │    │
│  │  │  │Public Subnet│  │Private Subnet│  │Database Subnet│        │    │    │
│  │  │  │ 10.0.2.0/24│  │ 10.0.12.0/24│  │ 10.0.22.0/24│          │    │    │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘          │    │    │
│  │  └─────────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Pilares do Well-Architected Framework

| Pilar | Descrição | Práticas IaC |
|-------|-----------|--------------|
| **Excelência Operacional** | Executar e monitorar sistemas | CloudWatch, X-Ray, CloudFormation |
| **Segurança** | Proteger dados e sistemas | IAM, KMS, Security Groups |
| **Confiabilidade** | Recuperação de falhas | Multi-AZ, Auto Scaling, Backup |
| **Eficiência de Performance** | Uso eficiente de recursos | Right-sizing, ElastiCache |
| **Otimização de Custos** | Evitar gastos desnecessários | Reserved Instances, Spot |
| **Sustentabilidade** | Minimizar impacto ambiental | Graviton, eficiência de recursos |

---

## 1. Amazon EC2

### Visão Geral

Amazon Elastic Compute Cloud (EC2) fornece capacidade computacional escalável na nuvem AWS.

### Arquitetura Recomendada

```
┌─────────────────────────────────────────────────────────────────┐
│                        Application Load Balancer                 │
│                              (ALB)                               │
└───────────────────────────────┬─────────────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
    ┌───────────┐         ┌───────────┐         ┌───────────┐
    │   EC2     │         │   EC2     │         │   EC2     │
    │ Instance  │         │ Instance  │         │ Instance  │
    │   (AZ-A)  │         │   (AZ-B)  │         │   (AZ-C)  │
    └───────────┘         └───────────┘         └───────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                ▼
                        ┌─────────────┐
                        │    RDS      │
                        │  Multi-AZ   │
                        └─────────────┘
```

### Exemplo Terraform - EC2 com Auto Scaling

```hcl
#------------------------------------------------------------------------------
# Launch Template
#------------------------------------------------------------------------------
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type

  network_interfaces {
    associate_public_ip_address = false
    security_groups            = [aws_security_group.app.id]
  }

  iam_instance_profile {
    name = aws_iam_instance_profile.app.name
  }

  user_data = base64encode(templatefile("${path.module}/userdata.sh", {
    environment = var.environment
    app_name    = var.app_name
  }))

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2
    http_put_response_hop_limit = 1
  }

  monitoring {
    enabled = true
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name        = "${var.app_name}-${var.environment}"
      Environment = var.environment
    }
  }

  lifecycle {
    create_before_destroy = true
  }
}

#------------------------------------------------------------------------------
# Auto Scaling Group
#------------------------------------------------------------------------------
resource "aws_autoscaling_group" "app" {
  name                = "${var.app_name}-${var.environment}-asg"
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"
  
  min_size         = var.min_size
  max_size         = var.max_size
  desired_capacity = var.desired_capacity

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }

  tag {
    key                 = "Name"
    value               = "${var.app_name}-${var.environment}"
    propagate_at_launch = true
  }

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}

#------------------------------------------------------------------------------
# Auto Scaling Policies
#------------------------------------------------------------------------------
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "${var.app_name}-scale-up"
  scaling_adjustment     = 1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.app.name
}

resource "aws_autoscaling_policy" "scale_down" {
  name                   = "${var.app_name}-scale-down"
  scaling_adjustment     = -1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.app.name
}

#------------------------------------------------------------------------------
# CloudWatch Alarms
#------------------------------------------------------------------------------
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.app_name}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "Scale up when CPU > 80%"
  alarm_actions       = [aws_autoscaling_policy.scale_up.arn]
  
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }
}
```

### Best Practices EC2

| Categoria | Prática | Justificativa |
|-----------|---------|---------------|
| **Segurança** | Usar IMDSv2 | Previne SSRF attacks |
| **Segurança** | Security Groups restritivos | Least privilege |
| **Disponibilidade** | Multi-AZ deployment | Alta disponibilidade |
| **Custo** | Right-sizing | Otimização de custos |
| **Custo** | Spot para workloads tolerantes | Até 90% de economia |
| **Performance** | Graviton instances | Melhor custo-benefício |

---

## 2. Amazon S3

### Hierarquia de Armazenamento

```
┌─────────────────────────────────────────────────────────────────┐
│                      S3 Standard                                 │
│          Acesso frequente, baixa latência                        │
│                        $0.023/GB                                 │
├─────────────────────────────────────────────────────────────────┤
│                    S3 Standard-IA                                │
│          Acesso infrequente, recuperação rápida                  │
│                        $0.0125/GB                                │
├─────────────────────────────────────────────────────────────────┤
│                    S3 Glacier Instant                            │
│          Arquivo com acesso em milissegundos                     │
│                        $0.004/GB                                 │
├─────────────────────────────────────────────────────────────────┤
│                  S3 Glacier Flexible                             │
│          Arquivo com recuperação em minutos/horas                │
│                        $0.0036/GB                                │
├─────────────────────────────────────────────────────────────────┤
│                   S3 Glacier Deep Archive                        │
│          Arquivo de longo prazo (7-10 anos)                      │
│                        $0.00099/GB                               │
└─────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - S3 com Replicação

```hcl
#------------------------------------------------------------------------------
# S3 Bucket com Replicação Cross-Region
#------------------------------------------------------------------------------
resource "aws_s3_bucket" "primary" {
  bucket = "${var.project}-${var.environment}-primary"

  tags = {
    Name        = "${var.project}-primary"
    Environment = var.environment
    Replication = "source"
  }
}

resource "aws_s3_bucket" "replica" {
  provider = aws.replica
  bucket   = "${var.project}-${var.environment}-replica"

  tags = {
    Name        = "${var.project}-replica"
    Environment = var.environment
    Replication = "destination"
  }
}

resource "aws_s3_bucket_versioning" "primary" {
  bucket = aws_s3_bucket.primary.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_versioning" "replica" {
  provider = aws.replica
  bucket   = aws_s3_bucket.replica.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_replication_configuration" "primary" {
  depends_on = [aws_s3_bucket_versioning.primary]

  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.primary.id

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.replica.arn
      storage_class = "STANDARD_IA"
      
      encryption_configuration {
        replica_kms_key_id = aws_kms_key.replica.arn
      }
    }

    source_selection_criteria {
      sse_kms_encrypted_objects {
        status = "Enabled"
      }
    }
  }
}
```

---

## 3. AWS IAM

### Modelo de Permissões

```
┌─────────────────────────────────────────────────────────────────┐
│                        IAM Policies                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Managed   │  │   Inline    │  │  Resource   │              │
│  │  Policies   │  │  Policies   │  │ -based      │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
│         ▼                ▼                ▼                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     Evaluation                           │    │
│  │     Explicit Deny > Explicit Allow > Implicit Deny       │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - IAM Role com Least Privilege

```hcl
#------------------------------------------------------------------------------
# IAM Role para Aplicação
#------------------------------------------------------------------------------
resource "aws_iam_role" "app" {
  name = "${var.app_name}-${var.environment}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Condition = {
          StringEquals = {
            "aws:SourceAccount" = data.aws_caller_identity.current.account_id
          }
        }
      }
    ]
  })

  tags = {
    Name        = "${var.app_name}-role"
    Environment = var.environment
  }
}

#------------------------------------------------------------------------------
# Policy de Acesso a S3 (Least Privilege)
#------------------------------------------------------------------------------
resource "aws_iam_role_policy" "s3_access" {
  name = "s3-access"
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ListBucket"
        Effect = "Allow"
        Action = [
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.app_data.arn
        ]
        Condition = {
          StringLike = {
            "s3:prefix" = ["${var.app_name}/*"]
          }
        }
      },
      {
        Sid    = "ReadWriteObjects"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = [
          "${aws_s3_bucket.app_data.arn}/${var.app_name}/*"
        ]
      }
    ]
  })
}

#------------------------------------------------------------------------------
# Policy de Acesso a Secrets Manager
#------------------------------------------------------------------------------
resource "aws_iam_role_policy" "secrets_access" {
  name = "secrets-access"
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "GetSecrets"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          "arn:aws:secretsmanager:${var.region}:${data.aws_caller_identity.current.account_id}:secret:${var.app_name}/*"
        ]
        Condition = {
          StringEquals = {
            "aws:ResourceTag/Environment" = var.environment
          }
        }
      }
    ]
  })
}
```

---

## 4. Amazon VPC

### Design de Rede Segmentada

```
┌─────────────────────────────────────────────────────────────────┐
│                         VPC (10.0.0.0/16)                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Public Tier                                             │    │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐           │    │
│  │  │ 10.0.1.0/24│ │ 10.0.2.0/24│ │ 10.0.3.0/24│           │    │
│  │  │   (NAT)    │ │   (NAT)    │ │   (NAT)    │           │    │
│  │  └────────────┘ └────────────┘ └────────────┘           │    │
│  └────────────────────────────┬────────────────────────────┘    │
│                               │                                  │
│  ┌────────────────────────────┼────────────────────────────┐    │
│  │  Private Tier (App)        │                             │    │
│  │  ┌────────────┐ ┌──────────┴─┐ ┌────────────┐           │    │
│  │  │10.0.11.0/24│ │10.0.12.0/24│ │10.0.13.0/24│           │    │
│  │  │   (App)    │ │   (App)    │ │   (App)    │           │    │
│  │  └────────────┘ └────────────┘ └────────────┘           │    │
│  └────────────────────────────┬────────────────────────────┘    │
│                               │                                  │
│  ┌────────────────────────────┼────────────────────────────┐    │
│  │  Private Tier (Data)       │                             │    │
│  │  ┌────────────┐ ┌──────────┴─┐ ┌────────────┐           │    │
│  │  │10.0.21.0/24│ │10.0.22.0/24│ │10.0.23.0/24│           │    │
│  │  │   (RDS)    │ │   (RDS)    │ │   (RDS)    │           │    │
│  │  └────────────┘ └────────────┘ └────────────┘           │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - VPC Completa

```hcl
#------------------------------------------------------------------------------
# VPC Module
#------------------------------------------------------------------------------
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project}-${var.environment}"
  cidr = var.vpc_cidr

  azs              = ["${var.region}a", "${var.region}b", "${var.region}c"]
  public_subnets   = var.public_subnets
  private_subnets  = var.private_subnets
  database_subnets = var.database_subnets

  enable_nat_gateway     = true
  single_nat_gateway     = var.environment != "prod"
  one_nat_gateway_per_az = var.environment == "prod"

  enable_dns_hostnames = true
  enable_dns_support   = true

  enable_flow_log                      = true
  create_flow_log_cloudwatch_log_group = true
  create_flow_log_cloudwatch_iam_role  = true
  flow_log_max_aggregation_interval    = 60

  tags = {
    Environment = var.environment
    Project     = var.project
  }

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
    Tier                     = "public"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
    Tier                              = "private"
  }

  database_subnet_tags = {
    Tier = "database"
  }
}

#------------------------------------------------------------------------------
# VPC Endpoints
#------------------------------------------------------------------------------
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = module.vpc.vpc_id
  service_name = "com.amazonaws.${var.region}.s3"
  
  route_table_ids = concat(
    module.vpc.private_route_table_ids,
    module.vpc.database_route_table_ids
  )

  tags = {
    Name = "${var.project}-s3-endpoint"
  }
}

resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.region}.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.project}-ssm-endpoint"
  }
}
```

---

## 5. Segurança e Compliance

### Checklist de Segurança AWS

| Categoria | Controle | Ferramenta IaC |
|-----------|----------|----------------|
| Identity | MFA para root | AWS Config Rule |
| Identity | IAM least privilege | Terraform IAM |
| Network | VPC Flow Logs | Terraform |
| Network | Security Groups | Terraform |
| Data | S3 encryption | Terraform |
| Data | RDS encryption | Terraform |
| Logging | CloudTrail | Terraform |
| Monitoring | GuardDuty | Terraform |
| Compliance | Config Rules | Terraform |
| Secrets | Secrets Manager | Terraform |

### Exemplo - Configuração de Segurança

```hcl
#------------------------------------------------------------------------------
# AWS Config Rules
#------------------------------------------------------------------------------
resource "aws_config_config_rule" "s3_bucket_ssl_requests_only" {
  name = "s3-bucket-ssl-requests-only"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SSL_REQUESTS_ONLY"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_config_rule" "encrypted_volumes" {
  name = "encrypted-volumes"

  source {
    owner             = "AWS"
    source_identifier = "ENCRYPTED_VOLUMES"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

#------------------------------------------------------------------------------
# GuardDuty
#------------------------------------------------------------------------------
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
}

#------------------------------------------------------------------------------
# CloudTrail
#------------------------------------------------------------------------------
resource "aws_cloudtrail" "main" {
  name                          = "${var.project}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_logging                = true
  
  kms_key_id = aws_kms_key.cloudtrail.arn

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3"]
    }
  }

  tags = {
    Name        = "${var.project}-cloudtrail"
    Environment = var.environment
  }
}
```

---

## Referências

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS IaC Best Practices](https://docs.aws.amazon.com/whitepapers/latest/introduction-devops-aws/infrastructure-as-code.html)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [AWS CloudFormation](https://docs.aws.amazon.com/cloudformation/)
- [TOGAF Standard](https://www.opengroup.org/togaf)