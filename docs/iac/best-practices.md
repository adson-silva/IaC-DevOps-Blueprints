# Melhores Práticas em Infraestrutura como Código (IaC)

## Visão Geral

Este documento apresenta as melhores práticas para implementação de IaC, alinhadas com o framework TOGAF e padrões da indústria. Seguir estas práticas garante infraestrutura consistente, segura e escalável.

## Arquitetura de Referência TOGAF para IaC

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CAMADA DE GOVERNANÇA                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │   Políticas     │  │   Compliance    │  │   Auditoria     │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
├─────────────────────────────────────────────────────────────────────────────┤
│                        CAMADA DE AUTOMAÇÃO                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │    CI/CD        │  │   Validação     │  │   Deployment    │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
├─────────────────────────────────────────────────────────────────────────────┤
│                        CAMADA DE CÓDIGO                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │    Módulos      │  │   Templates     │  │   Configuração  │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1. Organização de Repositórios

### Estrutura Recomendada

```
infrastructure/
├── modules/                    # Módulos reutilizáveis
│   ├── networking/
│   │   ├── vpc/
│   │   ├── subnets/
│   │   └── security-groups/
│   ├── compute/
│   │   ├── ec2/
│   │   ├── aks/
│   │   └── gke/
│   └── storage/
│       ├── s3/
│       ├── blob/
│       └── gcs/
├── environments/               # Configurações por ambiente
│   ├── dev/
│   ├── staging/
│   └── prod/
├── policies/                   # Políticas de segurança
│   ├── opa/
│   └── sentinel/
├── tests/                      # Testes de infraestrutura
│   ├── unit/
│   ├── integration/
│   └── compliance/
└── docs/                       # Documentação
    ├── architecture/
    └── runbooks/
```

### Convenções de Nomenclatura

| Recurso | Padrão | Exemplo |
|---------|--------|---------|
| Resource Groups | `rg-<projeto>-<ambiente>-<região>` | `rg-webapp-prod-eastus` |
| Virtual Networks | `vnet-<projeto>-<ambiente>-<número>` | `vnet-webapp-prod-001` |
| Storage Accounts | `st<projeto><ambiente><número>` | `stwebappprod001` |
| VMs | `vm-<função>-<ambiente>-<número>` | `vm-web-prod-001` |
| Buckets S3 | `<org>-<projeto>-<ambiente>-<tipo>` | `acme-webapp-prod-assets` |

## 2. Gerenciamento de Estado

### Princípios Fundamentais

- **Estado Remoto**: Sempre use backends remotos (S3, Azure Blob, GCS)
- **State Locking**: Habilite locking para evitar conflitos
- **Criptografia**: Criptografe o estado em repouso
- **Separação**: Um estado por ambiente/componente

### Exemplo de Configuração (Terraform)

```hcl
terraform {
  backend "s3" {
    bucket         = "acme-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Anti-padrões a Evitar

| Anti-padrão | Problema | Solução |
|-------------|----------|---------|
| Estado local | Conflitos, perda de dados | Usar backend remoto |
| Estado monolítico | Mudanças lentas, alto risco | Dividir por componente |
| Sem locking | Corrupção de estado | Habilitar state locking |
| Estado não criptografado | Exposição de secrets | Habilitar criptografia |

## 3. Modularização

### Benefícios dos Módulos

- **Reutilização**: Evita duplicação de código
- **Consistência**: Padrões aplicados automaticamente
- **Manutenibilidade**: Mudanças centralizadas
- **Testabilidade**: Módulos podem ser testados isoladamente

### Estrutura de um Módulo

```
modules/
└── vpc/
    ├── main.tf           # Recursos principais
    ├── variables.tf      # Variáveis de entrada
    ├── outputs.tf        # Valores de saída
    ├── versions.tf       # Versões de providers
    ├── README.md         # Documentação
    └── examples/         # Exemplos de uso
        ├── simple/
        └── complete/
```

### Versionamento de Módulos

```hcl
module "vpc" {
  source  = "git::https://github.com/org/modules.git//vpc?ref=v1.2.0"
  
  name        = "main-vpc"
  cidr_block  = "10.0.0.0/16"
  environment = "production"
}
```

## 4. Integração CI/CD

### Pipeline Recomendado

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│   Lint     │───▶│   Test     │───▶│   Plan     │───▶│   Apply    │
└────────────┘    └────────────┘    └────────────┘    └────────────┘
      │                 │                 │                 │
      ▼                 ▼                 ▼                 ▼
  terraform        terratest         terraform         terraform
    fmt           OPA/Sentinel        plan              apply
  tflint           checkov           (review)       (auto/manual)
```

### Etapas do Pipeline

1. **Lint**: Validação de formatação e sintaxe
2. **Security Scan**: Análise de vulnerabilidades (tfsec, checkov)
3. **Unit Tests**: Testes de módulos isolados
4. **Plan**: Geração e revisão do plano de execução
5. **Approval**: Aprovação humana para ambientes críticos
6. **Apply**: Aplicação das mudanças
7. **Integration Tests**: Validação pós-deployment

## 5. Segurança

### Gestão de Secrets

```
┌─────────────────────────────────────────────────────────────────┐
│                    NUNCA FAÇA ISSO                              │
│  ✗ Secrets hardcoded no código                                  │
│  ✗ Secrets em variáveis de ambiente não criptografadas          │
│  ✗ Secrets commitados no Git                                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    SEMPRE FAÇA ISSO                             │
│  ✓ Use vault solutions (HashiCorp Vault, AWS Secrets Manager)   │
│  ✓ Use managed identities/service accounts                       │
│  ✓ Rotacione secrets regularmente                                │
│  ✓ Use git-secrets ou similar para prevenir commits acidentais  │
└─────────────────────────────────────────────────────────────────┘
```

### Políticas como Código

Implemente políticas usando OPA/Rego ou Sentinel:

```rego
# Exemplo OPA: Negar recursos sem tags
deny[msg] {
    resource := input.resource_changes[_]
    not resource.change.after.tags
    msg := sprintf("Recurso '%s' deve ter tags", [resource.address])
}

deny[msg] {
    resource := input.resource_changes[_]
    not resource.change.after.tags.Environment
    msg := sprintf("Recurso '%s' deve ter tag 'Environment'", [resource.address])
}
```

## 6. Testes

### Pirâmide de Testes IaC

```
                    /\
                   /  \
                  / E2E\          # Testes end-to-end (poucos)
                 /──────\
                /        \
               /Integration\       # Testes de integração
              /────────────\
             /              \
            /   Unit Tests   \     # Testes unitários (muitos)
           /──────────────────\
          /                    \
         /   Static Analysis    \  # Análise estática (base)
        /────────────────────────\
```

### Tipos de Testes

| Tipo | Ferramenta | Objetivo |
|------|------------|----------|
| Lint | tflint, terraform fmt | Sintaxe e formatação |
| Security | tfsec, checkov, trivy | Vulnerabilidades |
| Unit | Terratest, pytest | Lógica de módulos |
| Contract | conftest, OPA | Políticas e compliance |
| Integration | Terratest | Recursos reais |
| E2E | InSpec, ServerSpec | Validação completa |

## 7. Documentação

### Requisitos Mínimos

Cada módulo deve incluir:

- **README.md**: Descrição, uso, exemplos
- **CHANGELOG.md**: Histórico de mudanças
- **Diagramas**: Arquitetura visual
- **Inputs/Outputs**: Documentação automática (terraform-docs)

### Exemplo de README de Módulo

```markdown
# Módulo VPC

## Descrição
Cria uma VPC com subnets públicas e privadas.

## Uso
module "vpc" {
  source = "./modules/vpc"
  name   = "main"
  cidr   = "10.0.0.0/16"
}

## Inputs
| Nome | Descrição | Tipo | Default | Obrigatório |
|------|-----------|------|---------|-------------|
| name | Nome da VPC | string | - | sim |
| cidr | CIDR block | string | - | sim |

## Outputs
| Nome | Descrição |
|------|-----------|
| vpc_id | ID da VPC criada |
```

## 8. GitOps e Controle de Versão

### Estratégia de Branches

```
main (protected)
  │
  ├── develop
  │     │
  │     ├── feature/add-vpc-module
  │     ├── feature/update-security-groups
  │     └── bugfix/fix-subnet-cidr
  │
  └── release/v1.0.0
```

### Fluxo de Aprovação

1. **Feature Branch**: Desenvolvimento isolado
2. **Pull Request**: Revisão de código
3. **Automated Checks**: Lint, tests, plan
4. **Peer Review**: Mínimo 2 aprovadores para prod
5. **Merge**: Aplicação automática ou manual

## Checklist de Qualidade

- [ ] Código formatado e lintado
- [ ] Testes passando
- [ ] Documentação atualizada
- [ ] Secrets não expostos
- [ ] Tags obrigatórias aplicadas
- [ ] Estado remoto configurado
- [ ] Backup de estado habilitado
- [ ] Políticas de compliance verificadas
- [ ] Revisão de código aprovada

## Referências

- [TOGAF Standard](https://www.opengroup.org/togaf)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Azure Cloud Adoption Framework](https://docs.microsoft.com/azure/cloud-adoption-framework/)