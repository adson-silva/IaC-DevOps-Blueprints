# Exemplo Terraform: Configurando S3

Aprenda como criar um bucket S3 na AWS usando Terraform.

---

## Índice

1. [Visão Geral](#visão-geral)
2. [Pré-requisitos](#pré-requisitos)
3. [Estrutura do Projeto](#estrutura-do-projeto)
4. [Implementação](#implementação)
5. [Configurações Avançadas](#configurações-avançadas)
6. [Deploy](#deploy)
7. [Limpeza](#limpeza)

---

## Visão Geral

Este exemplo demonstra como criar um bucket S3 com as seguintes características:

- ✅ Criptografia server-side (SSE-S3)
- ✅ Versionamento habilitado
- ✅ Bloqueio de acesso público
- ✅ Lifecycle policies
- ✅ Logging de acesso
- ✅ Tags para organização

---

## Pré-requisitos

### Ferramentas Necessárias

- [Terraform](https://www.terraform.io/downloads.html) >= 1.0.0
- [AWS CLI](https://aws.amazon.com/cli/) configurado
- Conta AWS com permissões para S3

### Configurar AWS CLI

```bash
# Configurar credenciais
aws configure

# Verificar configuração
aws sts get-caller-identity
```

---

## Estrutura do Projeto

```
s3-terraform/
├── main.tf           # Recursos principais
├── variables.tf      # Definição de variáveis
├── outputs.tf        # Outputs do módulo
├── providers.tf      # Configuração de providers
├── terraform.tfvars  # Valores das variáveis
└── README.md         # Documentação
```

---

## Implementação

### providers.tf

```hcl
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  # Backend para estado remoto (recomendado para produção)
  # backend "s3" {
  #   bucket         = "meu-terraform-state"
  #   key            = "s3-example/terraform.tfstate"
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
      ManagedBy   = "Terraform"
    }
  }
}
```

### variables.tf

```hcl
variable "aws_region" {
  description = "Região AWS para criar os recursos"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Nome do projeto"
  type        = string
  default     = "s3-example"
}

variable "environment" {
  description = "Ambiente de deploy"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "O ambiente deve ser dev, staging ou prod."
  }
}

variable "bucket_name" {
  description = "Nome do bucket S3 (deve ser globalmente único)"
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
  description = "Regras de lifecycle para objetos"
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

variable "allowed_origins" {
  description = "Origens permitidas para CORS"
  type        = list(string)
  default     = []
}

variable "tags" {
  description = "Tags adicionais para recursos"
  type        = map(string)
  default     = {}
}
```

### main.tf

```hcl
# Bucket principal
resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name

  tags = merge(
    var.tags,
    {
      Name = var.bucket_name
    }
  )
}

# Bloqueio de acesso público
resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Criptografia server-side
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
    bucket_key_enabled = true
  }
}

# Versionamento
resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id

  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Suspended"
  }
}

# Configuração de ownership
resource "aws_s3_bucket_ownership_controls" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

# Lifecycle rules
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  count = length(var.lifecycle_rules) > 0 ? 1 : 0

  bucket = aws_s3_bucket.main.id

  dynamic "rule" {
    for_each = var.lifecycle_rules

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

      expiration {
        days = rule.value.expiration_days
      }
    }
  }
}

# Bucket de logging (se habilitado)
resource "aws_s3_bucket" "logs" {
  count = var.enable_logging ? 1 : 0

  bucket = "${var.bucket_name}-logs"

  tags = merge(
    var.tags,
    {
      Name    = "${var.bucket_name}-logs"
      Purpose = "S3 Access Logs"
    }
  )
}

resource "aws_s3_bucket_public_access_block" "logs" {
  count = var.enable_logging ? 1 : 0

  bucket = aws_s3_bucket.logs[0].id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "logs" {
  count = var.enable_logging ? 1 : 0

  bucket = aws_s3_bucket.logs[0].id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Política do bucket de logs para permitir logging
resource "aws_s3_bucket_policy" "logs" {
  count = var.enable_logging ? 1 : 0

  bucket = aws_s3_bucket.logs[0].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3ServerAccessLogsPolicy"
        Effect = "Allow"
        Principal = {
          Service = "logging.s3.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.logs[0].arn}/*"
        Condition = {
          ArnLike = {
            "aws:SourceArn" = aws_s3_bucket.main.arn
          }
          StringEquals = {
            "aws:SourceAccount" = data.aws_caller_identity.current.account_id
          }
        }
      }
    ]
  })
}

# Configurar logging no bucket principal
resource "aws_s3_bucket_logging" "main" {
  count = var.enable_logging ? 1 : 0

  bucket = aws_s3_bucket.main.id

  target_bucket = aws_s3_bucket.logs[0].id
  target_prefix = "logs/"
}

# CORS configuration (se configurado)
resource "aws_s3_bucket_cors_configuration" "main" {
  count = length(var.allowed_origins) > 0 ? 1 : 0

  bucket = aws_s3_bucket.main.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "HEAD", "PUT", "POST", "DELETE"]
    allowed_origins = var.allowed_origins
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}

# Data source para account ID
data "aws_caller_identity" "current" {}
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

### terraform.tfvars

```hcl
# Configurações do projeto
aws_region   = "us-east-1"
project_name = "meu-projeto"
environment  = "dev"

# Configurações do bucket
bucket_name       = "meu-bucket-exemplo-12345"  # Deve ser único globalmente
enable_versioning = true
enable_logging    = true

# Lifecycle rules (opcional)
lifecycle_rules = [
  {
    id                       = "move-to-glacier"
    enabled                  = true
    prefix                   = "arquivos/"
    transition_days          = 90
    transition_storage_class = "GLACIER"
    expiration_days          = 365
  },
  {
    id                       = "cleanup-temp"
    enabled                  = true
    prefix                   = "temp/"
    transition_days          = 30
    transition_storage_class = "STANDARD_IA"
    expiration_days          = 90
  }
]

# CORS (opcional - para aplicações web)
allowed_origins = [
  "https://meusite.com",
  "https://app.meusite.com"
]

# Tags adicionais
tags = {
  Owner      = "time-devops"
  CostCenter = "12345"
}
```

---

## Configurações Avançadas

### Política de Bucket Personalizada

```hcl
# Política para permitir acesso de uma role específica
resource "aws_s3_bucket_policy" "main" {
  bucket = aws_s3_bucket.main.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowSpecificRole"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::123456789012:role/MyAppRole"
        }
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.main.arn,
          "${aws_s3_bucket.main.arn}/*"
        ]
      },
      {
        Sid       = "DenyInsecureConnections"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.main.arn,
          "${aws_s3_bucket.main.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}
```

### Replicação Cross-Region

```hcl
# Bucket de destino em outra região
provider "aws" {
  alias  = "replica"
  region = "eu-west-1"
}

resource "aws_s3_bucket" "replica" {
  provider = aws.replica
  bucket   = "${var.bucket_name}-replica"
}

resource "aws_s3_bucket_versioning" "replica" {
  provider = aws.replica
  bucket   = aws_s3_bucket.replica.id

  versioning_configuration {
    status = "Enabled"
  }
}

# IAM Role para replicação
resource "aws_iam_role" "replication" {
  name = "s3-replication-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "s3.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "replication" {
  name = "s3-replication-policy"
  role = aws_iam_role.replication.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "s3:GetReplicationConfiguration",
          "s3:ListBucket"
        ]
        Effect   = "Allow"
        Resource = aws_s3_bucket.main.arn
      },
      {
        Action = [
          "s3:GetObjectVersionForReplication",
          "s3:GetObjectVersionAcl",
          "s3:GetObjectVersionTagging"
        ]
        Effect   = "Allow"
        Resource = "${aws_s3_bucket.main.arn}/*"
      },
      {
        Action = [
          "s3:ReplicateObject",
          "s3:ReplicateDelete",
          "s3:ReplicateTags"
        ]
        Effect   = "Allow"
        Resource = "${aws_s3_bucket.replica.arn}/*"
      }
    ]
  })
}

# Configurar replicação
resource "aws_s3_bucket_replication_configuration" "main" {
  bucket = aws_s3_bucket.main.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.replica.arn
      storage_class = "STANDARD"
    }
  }

  depends_on = [aws_s3_bucket_versioning.main]
}
```

### Notificações de Eventos

```hcl
# Notificar SNS quando objetos são criados
resource "aws_s3_bucket_notification" "main" {
  bucket = aws_s3_bucket.main.id

  topic {
    topic_arn     = aws_sns_topic.s3_events.arn
    events        = ["s3:ObjectCreated:*"]
    filter_prefix = "uploads/"
  }

  lambda_function {
    lambda_function_arn = aws_lambda_function.processor.arn
    events              = ["s3:ObjectCreated:*"]
    filter_suffix       = ".csv"
  }
}
```

---

## Deploy

### Inicializar Projeto

```bash
# Navegar para o diretório do projeto
cd s3-terraform

# Inicializar Terraform
terraform init
```

### Visualizar Mudanças

```bash
# Gerar plano de execução
terraform plan -out=tfplan

# Revisar o plano
terraform show tfplan
```

### Aplicar Mudanças

```bash
# Aplicar o plano
terraform apply tfplan

# Ou aplicar diretamente (com confirmação)
terraform apply
```

### Verificar Recursos

```bash
# Listar recursos criados
terraform state list

# Ver detalhes de um recurso
terraform state show aws_s3_bucket.main

# Ver outputs
terraform output
```

---

## Limpeza

### Destruir Recursos

```bash
# Gerar plano de destruição
terraform plan -destroy

# Destruir recursos
terraform destroy
```

### Esvaziar Bucket Antes de Destruir

Se o bucket contiver objetos, você precisará esvaziá-lo antes de destruir:

```bash
# Usando AWS CLI
aws s3 rm s3://meu-bucket-exemplo-12345 --recursive

# Se tiver versionamento, remover versões também
aws s3api list-object-versions \
  --bucket meu-bucket-exemplo-12345 \
  --query 'Versions[].{Key:Key,VersionId:VersionId}' \
  --output json | jq -r '.[] | "\(.Key) \(.VersionId)"' | \
  while read key version; do
    aws s3api delete-object \
      --bucket meu-bucket-exemplo-12345 \
      --key "$key" \
      --version-id "$version"
  done
```

---

## Próximos Passos

- 📘 [Introdução ao IaC](../introduction.md)
- 📖 [Melhores Práticas](../best-practices.md)
- ☁️ [Guias AWS](../../clouds/aws-guides.md)