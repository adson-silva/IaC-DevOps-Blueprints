# Exemplo Terraform: Configurando Amazon S3

## Visão Geral

Este guia demonstra como criar e configurar um bucket S3 na AWS usando Terraform, seguindo as melhores práticas de segurança e o framework TOGAF para arquitetura empresarial.

## Arquitetura da Solução

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            AWS Account                                       │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                           S3 Bucket                                    │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │  Versioning │  │  Encryption │  │   Logging   │                    │  │
│  │  │   Enabled   │  │   AES-256   │  │   Enabled   │                    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │  Lifecycle  │  │   Public    │  │  Replication│                    │  │
│  │  │   Rules     │  │ Access Block│  │   (opcional)│                    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                           IAM Policies                                 │  │
│  │  ┌─────────────────────┐  ┌─────────────────────────────────────────┐ │  │
│  │  │   Bucket Policy     │  │     IAM Role for Access                 │ │  │
│  │  └─────────────────────┘  └─────────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Pré-requisitos

- Terraform >= 1.0.0
- AWS CLI configurado
- Credenciais AWS com permissões S3 e IAM

## Estrutura do Projeto

```
terraform-s3/
├── main.tf              # Recursos principais
├── variables.tf         # Variáveis de entrada
├── outputs.tf           # Valores de saída
├── versions.tf          # Versões de providers
├── locals.tf            # Variáveis locais
├── terraform.tfvars     # Valores das variáveis
└── README.md            # Documentação
```

## Implementação

### versions.tf

```hcl
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Backend remoto para estado (recomendado para produção)
  # backend "s3" {
  #   bucket         = "my-terraform-state"
  #   key            = "s3-bucket/terraform.tfstate"
  #   region         = "us-east-1"
  #   encrypt        = true
  #   dynamodb_table = "terraform-locks"
  # }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
      Owner       = var.owner
    }
  }
}
```

### variables.tf

```hcl
variable "aws_region" {
  description = "Região AWS para deploy dos recursos"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Nome do projeto"
  type        = string
  default     = "my-project"
}

variable "environment" {
  description = "Ambiente (dev, staging, prod)"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment deve ser dev, staging ou prod."
  }
}

variable "owner" {
  description = "Owner/responsável pelo recurso"
  type        = string
  default     = "platform-team"
}

variable "bucket_name" {
  description = "Nome do bucket S3"
  type        = string
}

variable "enable_versioning" {
  description = "Habilitar versionamento do bucket"
  type        = bool
  default     = true
}

variable "enable_logging" {
  description = "Habilitar logging de acesso"
  type        = bool
  default     = true
}

variable "lifecycle_rules" {
  description = "Regras de lifecycle para o bucket"
  type = list(object({
    id                       = string
    enabled                  = bool
    prefix                   = string
    transition_days          = number
    transition_storage_class = string
    expiration_days          = number
  }))
  default = []
}

variable "allowed_principals" {
  description = "ARNs de principals com acesso ao bucket"
  type        = list(string)
  default     = []
}
```

### locals.tf

```hcl
locals {
  bucket_name = "${var.project_name}-${var.environment}-${var.bucket_name}"
  
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.owner
    CreatedAt   = timestamp()
  }

  # Regras de lifecycle padrão se não fornecidas
  default_lifecycle_rules = [
    {
      id                       = "transition-to-ia"
      enabled                  = true
      prefix                   = ""
      transition_days          = 30
      transition_storage_class = "STANDARD_IA"
      expiration_days          = 0
    },
    {
      id                       = "transition-to-glacier"
      enabled                  = true
      prefix                   = "archive/"
      transition_days          = 90
      transition_storage_class = "GLACIER"
      expiration_days          = 365
    }
  ]

  lifecycle_rules = length(var.lifecycle_rules) > 0 ? var.lifecycle_rules : local.default_lifecycle_rules
}
```

### main.tf

```hcl
#------------------------------------------------------------------------------
# S3 Bucket Principal
#------------------------------------------------------------------------------
resource "aws_s3_bucket" "main" {
  bucket = local.bucket_name

  tags = merge(local.common_tags, {
    Name = local.bucket_name
  })
}

#------------------------------------------------------------------------------
# Configuração de Versionamento
#------------------------------------------------------------------------------
resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id

  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Disabled"
  }
}

#------------------------------------------------------------------------------
# Criptografia Server-Side
#------------------------------------------------------------------------------
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
    bucket_key_enabled = true
  }
}

#------------------------------------------------------------------------------
# Bloqueio de Acesso Público
#------------------------------------------------------------------------------
resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

#------------------------------------------------------------------------------
# Bucket de Logging (opcional)
#------------------------------------------------------------------------------
resource "aws_s3_bucket" "logs" {
  count  = var.enable_logging ? 1 : 0
  bucket = "${local.bucket_name}-logs"

  tags = merge(local.common_tags, {
    Name    = "${local.bucket_name}-logs"
    Purpose = "access-logs"
  })
}

resource "aws_s3_bucket_versioning" "logs" {
  count  = var.enable_logging ? 1 : 0
  bucket = aws_s3_bucket.logs[0].id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "logs" {
  count  = var.enable_logging ? 1 : 0
  bucket = aws_s3_bucket.logs[0].id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "logs" {
  count  = var.enable_logging ? 1 : 0
  bucket = aws_s3_bucket.logs[0].id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

#------------------------------------------------------------------------------
# Configuração de Logging
#------------------------------------------------------------------------------
resource "aws_s3_bucket_logging" "main" {
  count  = var.enable_logging ? 1 : 0
  bucket = aws_s3_bucket.main.id

  target_bucket = aws_s3_bucket.logs[0].id
  target_prefix = "logs/"
}

#------------------------------------------------------------------------------
# Regras de Lifecycle
#------------------------------------------------------------------------------
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  dynamic "rule" {
    for_each = local.lifecycle_rules
    content {
      id     = rule.value.id
      status = rule.value.enabled ? "Enabled" : "Disabled"

      filter {
        prefix = rule.value.prefix
      }

      transition {
        days          = rule.value.transition_days
        storage_class = rule.value.transition_storage_class
      }

      dynamic "expiration" {
        for_each = rule.value.expiration_days > 0 ? [1] : []
        content {
          days = rule.value.expiration_days
        }
      }
    }
  }
}

#------------------------------------------------------------------------------
# Bucket Policy
#------------------------------------------------------------------------------
resource "aws_s3_bucket_policy" "main" {
  count  = length(var.allowed_principals) > 0 ? 1 : 0
  bucket = aws_s3_bucket.main.id
  policy = data.aws_iam_policy_document.bucket_policy[0].json
}

data "aws_iam_policy_document" "bucket_policy" {
  count = length(var.allowed_principals) > 0 ? 1 : 0

  statement {
    sid    = "AllowPrincipalsAccess"
    effect = "Allow"

    principals {
      type        = "AWS"
      identifiers = var.allowed_principals
    }

    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:ListBucket",
    ]

    resources = [
      aws_s3_bucket.main.arn,
      "${aws_s3_bucket.main.arn}/*",
    ]
  }

  statement {
    sid    = "DenyInsecureTransport"
    effect = "Deny"

    principals {
      type        = "*"
      identifiers = ["*"]
    }

    actions = ["s3:*"]

    resources = [
      aws_s3_bucket.main.arn,
      "${aws_s3_bucket.main.arn}/*",
    ]

    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}
```

### outputs.tf

```hcl
output "bucket_id" {
  description = "ID do bucket S3"
  value       = aws_s3_bucket.main.id
}

output "bucket_arn" {
  description = "ARN do bucket S3"
  value       = aws_s3_bucket.main.arn
}

output "bucket_domain_name" {
  description = "Domain name do bucket"
  value       = aws_s3_bucket.main.bucket_domain_name
}

output "bucket_regional_domain_name" {
  description = "Regional domain name do bucket"
  value       = aws_s3_bucket.main.bucket_regional_domain_name
}

output "logs_bucket_id" {
  description = "ID do bucket de logs (se habilitado)"
  value       = var.enable_logging ? aws_s3_bucket.logs[0].id : null
}

output "logs_bucket_arn" {
  description = "ARN do bucket de logs (se habilitado)"
  value       = var.enable_logging ? aws_s3_bucket.logs[0].arn : null
}
```

### terraform.tfvars (exemplo)

```hcl
aws_region    = "us-east-1"
project_name  = "acme"
environment   = "prod"
owner         = "platform-team"
bucket_name   = "application-assets"

enable_versioning = true
enable_logging    = true

lifecycle_rules = [
  {
    id                       = "archive-old-files"
    enabled                  = true
    prefix                   = ""
    transition_days          = 30
    transition_storage_class = "STANDARD_IA"
    expiration_days          = 0
  },
  {
    id                       = "delete-old-logs"
    enabled                  = true
    prefix                   = "logs/"
    transition_days          = 7
    transition_storage_class = "GLACIER"
    expiration_days          = 90
  }
]

allowed_principals = [
  "arn:aws:iam::123456789012:role/application-role"
]
```

## Uso

### Inicialização

```bash
# Inicializar o Terraform
terraform init

# Validar a configuração
terraform validate

# Formatar o código
terraform fmt
```

### Planejamento e Aplicação

```bash
# Ver o plano de execução
terraform plan

# Aplicar as mudanças
terraform apply

# Aplicar com auto-approve (cuidado em produção)
terraform apply -auto-approve
```

### Destruição

```bash
# Ver o que será destruído
terraform plan -destroy

# Destruir os recursos
terraform destroy
```

## Segurança

### Checklist de Segurança

| Item | Status | Descrição |
|------|--------|-----------|
| Criptografia | ✅ | AES-256 server-side encryption |
| Acesso público | ✅ | Bloqueado por padrão |
| Versionamento | ✅ | Habilitado para recuperação |
| Logging | ✅ | Access logs habilitados |
| Transport | ✅ | HTTPS obrigatório (deny insecure) |
| IAM | ✅ | Least privilege via bucket policy |

### Políticas Recomendadas

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEGURANÇA S3                                  │
├─────────────────────────────────────────────────────────────────┤
│  ✓ Bloquear acesso público                                      │
│  ✓ Habilitar criptografia em repouso                            │
│  ✓ Forçar HTTPS para transporte                                 │
│  ✓ Habilitar versionamento                                      │
│  ✓ Configurar lifecycle rules                                    │
│  ✓ Habilitar access logging                                      │
│  ✓ Usar IAM policies específicas                                │
│  ✓ Habilitar MFA delete (produção)                              │
│  ✓ Configurar replicação cross-region (DR)                      │
└─────────────────────────────────────────────────────────────────┘
```

## Testes

### Validação com Terratest

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestS3BucketCreation(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
            "bucket_name":  "test-bucket",
            "environment":  "dev",
            "project_name": "test",
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    bucketID := terraform.Output(t, terraformOptions, "bucket_id")
    assert.Contains(t, bucketID, "test-bucket")
}
```

### Validação de Segurança

```bash
# Usando tfsec
tfsec .

# Usando checkov
checkov -d .

# Usando trivy
trivy config .
```

## Troubleshooting

| Problema | Causa | Solução |
|----------|-------|---------|
| Bucket name already exists | Nome globalmente duplicado | Usar nome único (incluir account ID) |
| Access Denied | Permissões IAM insuficientes | Verificar políticas IAM |
| Versioning not working | Configuração incorreta | Verificar `aws_s3_bucket_versioning` |
| Objects not transitioning | Lifecycle rules incorretas | Verificar filtros e dias |

## Referências

- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [TOGAF Documentation](https://www.opengroup.org/togaf)