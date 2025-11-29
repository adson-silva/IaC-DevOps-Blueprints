# Melhores Práticas em IaC

Dicas para organizar repositórios, gerenciar estados do Terraform e integrar automação com pipelines CI/CD.

---

## Índice

1. [Organização de Repositório](#organização-de-repositório)
2. [Gerenciamento de Estado](#gerenciamento-de-estado)
3. [Modularização](#modularização)
4. [Segurança](#segurança)
5. [Integração CI/CD](#integração-cicd)
6. [Documentação](#documentação)
7. [Testes](#testes)
8. [Convenções de Nomenclatura](#convenções-de-nomenclatura)

---

## Organização de Repositório

### Estrutura Recomendada

```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── compute/
│   │   └── ...
│   └── database/
│       └── ...
├── scripts/
│   ├── deploy.sh
│   └── destroy.sh
├── .github/
│   └── workflows/
│       └── terraform.yml
├── .gitignore
├── .terraform-version
└── README.md
```

### Separação de Ambientes

| Estratégia | Descrição | Quando Usar |
|------------|-----------|-------------|
| **Workspaces** | Mesmo código, estados separados | Ambientes similares |
| **Diretórios** | Diretórios separados por ambiente | Ambientes com diferenças significativas |
| **Repositórios** | Repositórios separados | Equipes independentes |

### Arquivo .gitignore

```gitignore
# Terraform
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl

# Variáveis sensíveis
*.tfvars
!*.tfvars.example

# Credenciais
.env
*.pem
*.key

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db
```

---

## Gerenciamento de Estado

### Estado Remoto

**Nunca armazene o estado localmente em produção.**

#### AWS S3 + DynamoDB

```hcl
terraform {
  backend "s3" {
    bucket         = "minha-empresa-terraform-state"
    key            = "prod/infrastructure/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

```hcl
# Criar recursos de backend (execute uma vez)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "minha-empresa-terraform-state"
  
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

#### Azure Blob Storage

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### Boas Práticas de Estado

| Prática | Descrição |
|---------|-----------|
| ✅ Criptografia | Sempre criptografe o estado em repouso |
| ✅ Versionamento | Habilite versionamento para rollback |
| ✅ Lock | Use locks para evitar mudanças concorrentes |
| ✅ Backup | Configure backups automáticos |
| ❌ Commit | Nunca commite arquivos de estado |

---

## Modularização

### Criando Módulos Reutilizáveis

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support
  
  tags = merge(
    var.tags,
    {
      Name = var.name
    }
  )
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.availability_zones[count.index]
  
  map_public_ip_on_launch = true
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-public-${count.index + 1}"
      Tier = "Public"
    }
  )
}
```

```hcl
# modules/vpc/variables.tf
variable "name" {
  description = "Nome da VPC"
  type        = string
}

variable "cidr_block" {
  description = "Bloco CIDR da VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnets" {
  description = "Lista de CIDRs para subnets públicas"
  type        = list(string)
  default     = []
}

variable "availability_zones" {
  description = "Lista de AZs para as subnets"
  type        = list(string)
}

variable "enable_dns_hostnames" {
  description = "Habilitar DNS hostnames"
  type        = bool
  default     = true
}

variable "enable_dns_support" {
  description = "Habilitar DNS support"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Tags para recursos"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "ID da VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs das subnets públicas"
  value       = aws_subnet.public[*].id
}
```

### Usando Módulos

```hcl
module "vpc" {
  source = "../../modules/vpc"
  
  name               = "producao"
  cidr_block         = "10.0.0.0/16"
  public_subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
  availability_zones = ["us-east-1a", "us-east-1b"]
  
  tags = {
    Environment = "production"
    Project     = "main-app"
  }
}
```

---

## Segurança

### Gerenciamento de Segredos

#### ❌ Não Faça

```hcl
# NUNCA hardcode credenciais
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

#### ✅ Use Variáveis de Ambiente

```bash
export AWS_ACCESS_KEY_ID="sua-access-key"
export AWS_SECRET_ACCESS_KEY="sua-secret-key"
```

#### ✅ Use Gerenciadores de Segredos

```hcl
# AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  # ...
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

```hcl
# Azure Key Vault
data "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  key_vault_id = azurerm_key_vault.main.id
}
```

### Políticas de Segurança como Código

```hcl
# Sentinel Policy (Terraform Enterprise)
policy "require-encryption" {
  enforcement_level = "hard-mandatory"
}

# OPA (Open Policy Agent)
deny[msg] {
  resource := input.planned_values.root_module.resources[_]
  resource.type == "aws_s3_bucket"
  not resource.values.server_side_encryption_configuration
  msg := sprintf("S3 bucket '%s' deve ter criptografia habilitada", [resource.name])
}
```

---

## Integração CI/CD

### GitHub Actions

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  TF_VERSION: "1.5.0"

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Format
        run: terraform fmt -check -recursive
      
      - name: Terraform Init
        run: terraform init -backend=false
      
      - name: Terraform Validate
        run: terraform validate

  plan:
    needs: validate
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Plan
        run: terraform plan -no-color
        continue-on-error: true
      
      - name: Comment on PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

  apply:
    needs: plan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Apply
        run: terraform apply -auto-approve
```

### Azure DevOps

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  terraformVersion: '1.5.0'

stages:
  - stage: Validate
    jobs:
      - job: ValidateTerraform
        steps:
          - task: TerraformInstaller@0
            inputs:
              terraformVersion: $(terraformVersion)
          
          - script: |
              terraform fmt -check -recursive
              terraform init -backend=false
              terraform validate
            displayName: 'Validate Terraform'

  - stage: Plan
    dependsOn: Validate
    condition: and(succeeded(), eq(variables['Build.Reason'], 'PullRequest'))
    jobs:
      - job: PlanTerraform
        steps:
          - task: TerraformInstaller@0
            inputs:
              terraformVersion: $(terraformVersion)
          
          - task: TerraformTaskV4@4
            inputs:
              command: 'init'
              backendServiceArm: 'Azure-Connection'
          
          - task: TerraformTaskV4@4
            inputs:
              command: 'plan'

  - stage: Apply
    dependsOn: Plan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployTerraform
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: TerraformTaskV4@4
                  inputs:
                    command: 'apply'
                    commandOptions: '-auto-approve'
```

---

## Documentação

### README de Módulo

```markdown
# Módulo VPC

Este módulo cria uma VPC com subnets públicas e privadas.

## Uso

\`\`\`hcl
module "vpc" {
  source = "./modules/vpc"
  
  name       = "producao"
  cidr_block = "10.0.0.0/16"
}
\`\`\`

## Inputs

| Nome | Descrição | Tipo | Default | Obrigatório |
|------|-----------|------|---------|-------------|
| name | Nome da VPC | string | - | sim |
| cidr_block | CIDR da VPC | string | "10.0.0.0/16" | não |

## Outputs

| Nome | Descrição |
|------|-----------|
| vpc_id | ID da VPC criada |
| public_subnet_ids | IDs das subnets públicas |

## Exemplos

### VPC Básica

\`\`\`hcl
module "vpc" {
  source = "./modules/vpc"
  name   = "dev"
}
\`\`\`

### VPC com Múltiplas Subnets

\`\`\`hcl
module "vpc" {
  source         = "./modules/vpc"
  name           = "prod"
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
}
\`\`\`
```

### terraform-docs

```bash
# Instalar terraform-docs
brew install terraform-docs

# Gerar documentação
terraform-docs markdown table ./modules/vpc > ./modules/vpc/README.md
```

---

## Testes

### Terratest

```go
// test/vpc_test.go
package test

import (
    "testing"
    
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
    t.Parallel()
    
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "name":       "test-vpc",
            "cidr_block": "10.0.0.0/16",
        },
    })
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)
}
```

### Validação de Políticas

```bash
# Conftest (OPA)
conftest test main.tf

# Checkov
checkov -d .

# TFLint
tflint --init
tflint
```

---

## Convenções de Nomenclatura

### Recursos

| Tipo | Padrão | Exemplo |
|------|--------|---------|
| Resource Group | `rg-{projeto}-{ambiente}` | `rg-webapp-prod` |
| VPC/VNet | `vpc-{projeto}-{ambiente}` | `vpc-main-dev` |
| Subnet | `snet-{tipo}-{ambiente}` | `snet-public-prod` |
| EC2/VM | `vm-{função}-{número}` | `vm-web-01` |
| S3/Blob | `st{empresa}{projeto}{ambiente}` | `stacmewebprod` |
| Load Balancer | `lb-{projeto}-{ambiente}` | `lb-api-prod` |

### Variáveis

```hcl
# Use snake_case para variáveis
variable "instance_type" {}
variable "vpc_cidr_block" {}
variable "enable_monitoring" {}

# Prefixos por tipo
variable "enable_dns" {}    # Booleanos: enable_, is_, has_
variable "list_of_ips" {}   # Listas: list_of_, allowed_
variable "map_of_tags" {}   # Maps: map_of_
```

### Tags Obrigatórias

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
    Owner       = var.owner_email
    CostCenter  = var.cost_center
    CreatedAt   = timestamp()
  }
}
```

---

## Checklist de Revisão

- [ ] Código formatado (`terraform fmt`)
- [ ] Validação passou (`terraform validate`)
- [ ] Linting sem erros (`tflint`)
- [ ] Segredos gerenciados corretamente
- [ ] Estado remoto configurado
- [ ] Documentação atualizada
- [ ] Testes passando
- [ ] Tags aplicadas
- [ ] Recursos nomeados corretamente
- [ ] Políticas de segurança validadas

---

## Próximos Passos

- 📘 [Introdução ao IaC](./introduction.md)
- 🔧 [Visão Geral das Ferramentas](./tools-overview.md)
- 💡 [Exemplos Práticos](./examples/terraform-s3-setup.md)