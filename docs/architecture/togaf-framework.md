# Framework TOGAF para Infraestrutura como Código

## Visão Geral

O TOGAF (The Open Group Architecture Framework) é um framework de arquitetura empresarial que fornece uma abordagem abrangente para design, planejamento, implementação e governança de arquiteturas de TI. Este documento detalha como aplicar os princípios TOGAF à Infraestrutura como Código (IaC).

## Estrutura do TOGAF

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              TOGAF Framework                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                   ADM (Architecture Development Method)              │    │
│  │                                                                       │    │
│  │     ┌─────────┐                                                      │    │
│  │     │Preliminar│                                                     │    │
│  │     └────┬────┘                                                      │    │
│  │          │                                                            │    │
│  │     ┌────▼────┐                                                      │    │
│  │     │ A:Vision│                                                      │    │
│  │     └────┬────┘                                                      │    │
│  │          │                                                            │    │
│  │ ┌────────┼────────────────────────────────────────────┐              │    │
│  │ │        ▼                                             │              │    │
│  │ │   ┌─────────┐    ┌─────────┐    ┌─────────┐        │              │    │
│  │ │   │B:Business│───▶│C:Info Sys│───▶│D:Tech   │        │              │    │
│  │ │   │   Arch  │    │   Arch   │    │  Arch   │        │              │    │
│  │ │   └─────────┘    └─────────┘    └─────────┘        │              │    │
│  │ └──────────────────────┬──────────────────────────────┘              │    │
│  │                        ▼                                              │    │
│  │  ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐        │    │
│  │  │E:Opport.│────▶│F:Migration│──▶│G:Implement│──▶│H:Change │        │    │
│  │  │Solutions│     │ Planning │    │Governance│    │  Mgmt   │        │    │
│  │  └─────────┘     └─────────┘     └─────────┘     └─────────┘        │    │
│  │                                                                       │    │
│  │     ┌───────────────────────────────────────────────────────┐        │    │
│  │     │              Requirements Management                   │        │    │
│  │     └───────────────────────────────────────────────────────┘        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Enterprise Continuum                             │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │    │
│  │  │  Foundation  │  │   Common     │  │  Industry    │               │    │
│  │  │  Architecture│  │   Systems    │  │  Architecture│               │    │
│  │  │              │  │  Architecture│  │              │               │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Camadas de Arquitetura TOGAF

### Mapeamento para IaC

| Camada TOGAF | Descrição | Artefatos IaC |
|--------------|-----------|---------------|
| **Business Architecture** | Estratégia, governança, organização | Políticas, tags, budgets |
| **Data Architecture** | Estrutura de dados, gestão, governança | Storage, databases, backup |
| **Application Architecture** | Sistemas de aplicação, interações | Compute, containers, serverless |
| **Technology Architecture** | Infraestrutura de TI, redes | VPCs, networking, security |

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TOGAF para IaC                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Business Architecture                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  • Políticas de Tagging (custo, ownership)                           │    │
│  │  • Budget Alerts e Cost Management                                   │    │
│  │  • Governance Policies (Azure Policy, AWS SCP)                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                               │
│                              ▼                                               │
│  Data Architecture                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  • Databases (RDS, SQL, CloudSQL)                                    │    │
│  │  • Storage (S3, Blob, GCS)                                           │    │
│  │  • Data Classification e Encryption                                  │    │
│  │  • Backup e DR                                                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                               │
│                              ▼                                               │
│  Application Architecture                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  • Compute (EC2, VMs, GCE)                                           │    │
│  │  • Containers (EKS, AKS, GKE)                                        │    │
│  │  • Serverless (Lambda, Functions, Cloud Functions)                   │    │
│  │  • App Services e PaaS                                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                               │
│                              ▼                                               │
│  Technology Architecture                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  • Networking (VPC, VNet, Subnets)                                   │    │
│  │  • Security (Firewalls, WAF, DDoS)                                   │    │
│  │  • Identity (IAM, RBAC, MFA)                                         │    │
│  │  • Monitoring e Logging                                              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## ADM (Architecture Development Method) para IaC

### Fase A: Visão da Arquitetura

**Objetivo**: Definir escopo, stakeholders e visão de alto nível.

```hcl
#------------------------------------------------------------------------------
# Fase A: Architecture Vision
# Define a visão de alto nível e requisitos de stakeholders
#------------------------------------------------------------------------------

# Documentação da visão em código
locals {
  architecture_vision = {
    project_name = "enterprise-platform"
    description  = "Plataforma empresarial multi-cloud com alta disponibilidade"
    
    stakeholders = {
      business = {
        concerns = ["custo", "time-to-market", "compliance"]
      }
      security = {
        concerns = ["proteção de dados", "acesso", "auditoria"]
      }
      operations = {
        concerns = ["disponibilidade", "monitoramento", "recuperação"]
      }
      development = {
        concerns = ["agilidade", "CI/CD", "ambientes"]
      }
    }
    
    principles = [
      "Infrastructure as Code para toda infraestrutura",
      "Segurança por design (security by default)",
      "Automação de operações",
      "Multi-cloud ready",
      "Compliance como código"
    ]
    
    success_criteria = {
      availability    = "99.95%"
      rto             = "4 hours"
      rpo             = "1 hour"
      deployment_time = "< 30 minutes"
    }
  }
}

output "architecture_vision" {
  value = local.architecture_vision
}
```

### Fase B: Arquitetura de Negócios

**Objetivo**: Definir estratégia, governança e organização.

```hcl
#------------------------------------------------------------------------------
# Fase B: Business Architecture
# Implementa governança, políticas e gestão de custos
#------------------------------------------------------------------------------

# Políticas de Tagging
variable "required_tags" {
  description = "Tags obrigatórias para todos os recursos"
  type        = map(string)
  default = {
    Project       = ""
    Environment   = ""
    Owner         = ""
    CostCenter    = ""
    Compliance    = ""
    DataClass     = ""
    BusinessUnit  = ""
    Application   = ""
  }
}

# Azure Policy para Governança
resource "azurerm_policy_definition" "require_tags" {
  name         = "require-mandatory-tags"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "TOGAF Phase B - Require Mandatory Tags"
  description  = "Garante que todos os recursos tenham tags obrigatórias"

  metadata = jsonencode({
    version  = "1.0.0"
    category = "Tags"
    togaf    = "Phase B - Business Architecture"
  })

  policy_rule = jsonencode({
    if = {
      allOf = [
        {
          field  = "type"
          equals = "Microsoft.Resources/subscriptions/resourceGroups"
        },
        {
          anyOf = [
            for tag in keys(var.required_tags) : {
              field  = "tags['${tag}']"
              exists = "false"
            }
          ]
        }
      ]
    }
    then = {
      effect = "deny"
    }
  })
}

# Alertas de Budget (AWS)
resource "aws_budgets_budget" "monthly" {
  name         = "togaf-monthly-budget"
  budget_type  = "COST"
  limit_amount = var.monthly_budget
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name = "TagKeyValue"
    values = [
      "user:Project$${local.architecture_vision.project_name}"
    ]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = var.budget_alert_emails
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = var.budget_alert_emails
  }
}
```

### Fase C: Arquitetura de Sistemas de Informação

**Objetivo**: Definir arquitetura de dados e aplicações.

```hcl
#------------------------------------------------------------------------------
# Fase C: Information Systems Architecture
# Define arquitetura de dados e aplicações
#------------------------------------------------------------------------------

# Módulo de Data Architecture
module "data_architecture" {
  source = "./modules/data"

  # Classificação de dados
  data_classification = {
    public = {
      encryption_required = false
      backup_retention    = 7
      access_logging      = false
    }
    internal = {
      encryption_required = true
      backup_retention    = 30
      access_logging      = true
    }
    confidential = {
      encryption_required = true
      backup_retention    = 90
      access_logging      = true
      dlp_enabled         = true
    }
    restricted = {
      encryption_required  = true
      backup_retention     = 365
      access_logging       = true
      dlp_enabled          = true
      access_review_period = 30
    }
  }

  # Estratégia de backup
  backup_strategy = {
    daily = {
      retention_days = 7
      start_window   = "03:00"
    }
    weekly = {
      retention_days = 30
      start_window   = "04:00"
      day            = "Sunday"
    }
    monthly = {
      retention_days = 365
      start_window   = "05:00"
      day            = 1
    }
  }

  # Disaster Recovery
  dr_configuration = {
    enabled            = true
    target_region      = var.dr_region
    replication_type   = "async"
    failover_threshold = 300  # seconds
  }
}

# Módulo de Application Architecture
module "application_architecture" {
  source = "./modules/application"

  # Tiers da aplicação
  application_tiers = {
    presentation = {
      type     = "serverless"
      runtime  = "nodejs18.x"
      scaling  = "auto"
      min_instances = 0
      max_instances = 100
    }
    business = {
      type     = "container"
      platform = "kubernetes"
      scaling  = "hpa"
      min_replicas = 3
      max_replicas = 20
    }
    data = {
      type     = "managed"
      platform = "rds"
      multi_az = true
      read_replicas = 2
    }
  }

  # Integrations
  integration_patterns = {
    sync  = ["rest", "grpc"]
    async = ["sqs", "eventbridge"]
  }
}
```

### Fase D: Arquitetura de Tecnologia

**Objetivo**: Definir infraestrutura técnica.

```hcl
#------------------------------------------------------------------------------
# Fase D: Technology Architecture
# Define infraestrutura de rede, segurança e operações
#------------------------------------------------------------------------------

# Módulo de Network Architecture
module "network_architecture" {
  source = "./modules/network"

  # Design de rede
  network_design = {
    topology = "hub-spoke"
    
    hub = {
      cidr = "10.0.0.0/16"
      subnets = {
        gateway    = "10.0.1.0/24"
        firewall   = "10.0.2.0/24"
        management = "10.0.3.0/24"
        shared     = "10.0.4.0/24"
      }
    }
    
    spokes = {
      production = {
        cidr = "10.1.0.0/16"
        subnets = {
          web  = ["10.1.1.0/24", "10.1.2.0/24", "10.1.3.0/24"]
          app  = ["10.1.11.0/24", "10.1.12.0/24", "10.1.13.0/24"]
          data = ["10.1.21.0/24", "10.1.22.0/24", "10.1.23.0/24"]
        }
      }
      staging = {
        cidr = "10.2.0.0/16"
        subnets = {
          web  = ["10.2.1.0/24", "10.2.2.0/24"]
          app  = ["10.2.11.0/24", "10.2.12.0/24"]
          data = ["10.2.21.0/24", "10.2.22.0/24"]
        }
      }
    }
  }

  # Segurança de rede
  security_zones = {
    dmz = {
      ingress = ["internet"]
      egress  = ["app"]
    }
    app = {
      ingress = ["dmz", "mgmt"]
      egress  = ["data", "internet"]
    }
    data = {
      ingress = ["app"]
      egress  = []
    }
    mgmt = {
      ingress = ["vpn"]
      egress  = ["all"]
    }
  }
}

# Módulo de Security Architecture
module "security_architecture" {
  source = "./modules/security"

  # Camadas de segurança
  security_layers = {
    perimeter = {
      components = ["waf", "ddos_protection", "cloud_armor"]
      enabled    = true
    }
    network = {
      components = ["firewall", "nacls", "security_groups"]
      enabled    = true
    }
    identity = {
      components = ["iam", "mfa", "sso", "pim"]
      enabled    = true
    }
    data = {
      components = ["encryption", "key_management", "dlp"]
      enabled    = true
    }
    application = {
      components = ["secrets_management", "api_security", "code_scanning"]
      enabled    = true
    }
  }

  # Zero Trust
  zero_trust_config = {
    enabled                    = true
    verify_explicitly          = true
    least_privilege_access     = true
    assume_breach              = true
    micro_segmentation         = true
  }
}

# Módulo de Operations Architecture
module "operations_architecture" {
  source = "./modules/operations"

  # Observabilidade
  observability = {
    metrics = {
      provider    = "cloudwatch"
      retention   = 90
      granularity = "1m"
    }
    logging = {
      provider  = "cloudwatch_logs"
      retention = 365
      format    = "json"
    }
    tracing = {
      provider    = "xray"
      sample_rate = 0.1
    }
    dashboards = {
      provider = "grafana"
      enabled  = true
    }
  }

  # Alerting
  alerting = {
    channels = {
      critical = ["pagerduty", "slack"]
      warning  = ["slack", "email"]
      info     = ["email"]
    }
    
    sla_targets = {
      availability = 99.95
      latency_p99  = 500  # ms
      error_rate   = 0.1  # %
    }
  }
}
```

### Fase E-H: Implementação e Governança

```hcl
#------------------------------------------------------------------------------
# Fases E-H: Opportunities, Migration, Implementation, Change Management
# Implementa CI/CD, deployment e gestão de mudanças
#------------------------------------------------------------------------------

# Módulo de Deployment Architecture
module "deployment_architecture" {
  source = "./modules/deployment"

  # Estratégia de deployment
  deployment_strategy = {
    type = "blue-green"
    
    canary = {
      enabled     = true
      percentage  = 10
      duration    = "15m"
      metrics     = ["error_rate", "latency"]
      thresholds  = {
        error_rate = 1  # %
        latency    = 500  # ms
      }
    }
    
    rollback = {
      automatic  = true
      timeout    = "10m"
    }
  }

  # GitOps
  gitops = {
    enabled     = true
    provider    = "argocd"
    repository  = var.gitops_repo
    branch      = "main"
    path        = "environments/${var.environment}"
    sync_policy = "automated"
  }

  # Environments
  environments = {
    development = {
      auto_deploy  = true
      approval     = false
      retention    = 7
    }
    staging = {
      auto_deploy  = true
      approval     = false
      retention    = 14
    }
    production = {
      auto_deploy  = false
      approval     = true
      approvers    = var.production_approvers
      retention    = 90
      change_window = {
        days       = ["Tuesday", "Wednesday", "Thursday"]
        start_hour = 10
        end_hour   = 16
        timezone   = "America/Sao_Paulo"
      }
    }
  }
}

# Requirements Management
resource "local_file" "requirements_traceability" {
  filename = "${path.module}/outputs/requirements_traceability.json"
  content = jsonencode({
    project = local.architecture_vision.project_name
    version = var.architecture_version
    date    = timestamp()
    
    requirements = {
      business = {
        BR001 = {
          description = "Alta disponibilidade (99.95%)"
          implementation = "Multi-AZ deployment, auto-scaling"
          status = "implemented"
          tests = ["availability_test", "failover_test"]
        }
        BR002 = {
          description = "Compliance com regulamentações"
          implementation = "Encryption, logging, access control"
          status = "implemented"
          tests = ["compliance_scan", "security_audit"]
        }
      }
      
      technical = {
        TR001 = {
          description = "Infraestrutura como código"
          implementation = "Terraform modules"
          status = "implemented"
          tests = ["terraform_validate", "plan_review"]
        }
        TR002 = {
          description = "CI/CD automatizado"
          implementation = "GitHub Actions + ArgoCD"
          status = "implemented"
          tests = ["pipeline_test", "deployment_test"]
        }
      }
    }
  })
}
```

---

## Princípios de Arquitetura TOGAF para IaC

### Princípios Fundamentais

| Princípio | Descrição | Implementação IaC |
|-----------|-----------|-------------------|
| **Reusabilidade** | Maximizar reutilização de componentes | Módulos Terraform versionados |
| **Interoperabilidade** | Sistemas devem trabalhar juntos | APIs padronizadas, formatos comuns |
| **Escalabilidade** | Suportar crescimento | Auto-scaling, infrastructure patterns |
| **Segurança** | Proteção por design | Security policies, encryption |
| **Simplicidade** | Evitar complexidade desnecessária | Módulos simples, composição |

### Padrões de Design

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TOGAF Design Patterns para IaC                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Layered Architecture                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Infrastructure Layer (VPC, Subnets, Security Groups)               │    │
│  │  Platform Layer (Kubernetes, Databases, Cache)                       │    │
│  │  Application Layer (Services, Functions, APIs)                       │    │
│  │  Presentation Layer (CDN, Load Balancer, WAF)                        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Composition Pattern                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Root Module                                                         │    │
│  │    ├── module "networking"                                           │    │
│  │    ├── module "security"                                             │    │
│  │    ├── module "compute"                                              │    │
│  │    ├── module "data"                                                 │    │
│  │    └── module "observability"                                        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  Environment Pattern                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  environments/                                                       │    │
│  │    ├── dev/                                                          │    │
│  │    │     └── main.tf (uses shared modules with dev vars)            │    │
│  │    ├── staging/                                                      │    │
│  │    │     └── main.tf (uses shared modules with staging vars)        │    │
│  │    └── prod/                                                         │    │
│  │          └── main.tf (uses shared modules with prod vars)           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Governança de Arquitetura

### Estrutura de Governança

```hcl
#------------------------------------------------------------------------------
# Architecture Governance
# Implementa governança de arquitetura com policies e controles
#------------------------------------------------------------------------------

# Architecture Decision Records (ADRs)
resource "local_file" "adr_template" {
  filename = "${path.module}/docs/adr/template.md"
  content  = <<-EOT
    # ADR {NUMBER}: {TÍTULO}

    ## Status
    {Proposta | Aceita | Depreciada | Substituída}

    ## Contexto
    Descreva o contexto e o problema que estamos enfrentando.

    ## Decisão
    Descreva a decisão tomada.

    ## Consequências
    Descreva as consequências positivas e negativas da decisão.

    ## Compliance
    - TOGAF Phase: {A|B|C|D|E|F|G|H}
    - Principles: {Lista de princípios relacionados}
    - Standards: {Lista de padrões relacionados}
  EOT
}

# Compliance Checks
resource "null_resource" "architecture_compliance" {
  triggers = {
    always_run = timestamp()
  }

  provisioner "local-exec" {
    command = <<-EOT
      # Verificar compliance com princípios de arquitetura
      echo "Verificando compliance de arquitetura..."
      
      # Check 1: Todas as tags obrigatórias
      terraform state list | xargs -I {} terraform state show {} | grep -c "tags"
      
      # Check 2: Encryption enabled
      terraform plan -json | jq '.resource_changes[] | select(.type | contains("storage", "database")) | .change.after.encryption'
      
      # Check 3: Logging enabled
      terraform plan -json | jq '.resource_changes[] | select(.type | contains("logging", "trail", "flow_log")) | length'
    EOT
  }
}
```

---

## Referências

- [TOGAF Standard, Version 9.2](https://www.opengroup.org/togaf)
- [TOGAF Series Guide: Using TOGAF to Define and Govern SOA](https://www.opengroup.org/library/g112/)
- [ArchiMate Specification](https://www.opengroup.org/archimate-home)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Azure Cloud Adoption Framework](https://docs.microsoft.com/azure/cloud-adoption-framework/)
- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
