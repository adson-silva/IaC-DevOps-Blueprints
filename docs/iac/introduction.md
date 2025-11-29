# Introdução ao IaC

A Infraestrutura como Código (IaC) automatiza o provisionamento e gerenciamento de infraestrutura. Aqui, exploramos como começar, principais ferramentas e benefícios.

---

## O que é Infraestrutura como Código?

**Infraestrutura como Código (IaC)** é uma prática que permite gerenciar e provisionar recursos de infraestrutura por meio de arquivos de configuração legíveis por máquina, em vez de processos manuais ou ferramentas interativas.

### Definição

IaC trata a infraestrutura da mesma forma que o código de aplicação:
- **Versionável**: Histórico completo de alterações
- **Revisável**: Code review antes de aplicar mudanças
- **Reproduzível**: Ambientes idênticos em qualquer momento
- **Automatizável**: Integração com pipelines CI/CD

---

## Por que adotar IaC?

### Benefícios Principais

| Benefício | Descrição |
|-----------|-----------|
| **Consistência** | Elimina configurações manuais e reduz erros humanos |
| **Velocidade** | Provisionamento em minutos em vez de dias |
| **Escalabilidade** | Fácil replicação de ambientes |
| **Documentação** | O código serve como documentação viva |
| **Auditoria** | Rastreabilidade completa de mudanças |
| **Custo** | Redução de retrabalho e otimização de recursos |

### Casos de Uso

1. **Ambientes de Desenvolvimento**
   - Criar ambientes locais idênticos ao de produção
   - Facilitar onboarding de novos desenvolvedores

2. **Disaster Recovery**
   - Reconstrução rápida de infraestrutura
   - Testes regulares de recuperação

3. **Multi-ambiente**
   - Dev, Staging, Produção com configurações consistentes
   - Promoção segura entre ambientes

4. **Compliance**
   - Políticas de segurança codificadas
   - Auditorias automatizadas

---

## Conceitos Fundamentais

### Abordagem Declarativa vs Imperativa

#### Declarativa
Define o **estado desejado** da infraestrutura. A ferramenta determina como alcançá-lo.

```hcl
# Exemplo Terraform (Declarativo)
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}
```

#### Imperativa
Define os **passos específicos** para alcançar o estado desejado.

```yaml
# Exemplo Ansible (Imperativo)
- name: Criar instância EC2
  amazon.aws.ec2_instance:
    name: "WebServer"
    image_id: ami-0c55b159cbfafe1f0
    instance_type: t2.micro
    state: present
```

### Imutabilidade

A **infraestrutura imutável** substitui servidores em vez de modificá-los:

- ✅ Criar novos recursos com a configuração atualizada
- ✅ Destruir recursos antigos após validação
- ❌ Evitar alterações diretas em recursos existentes

### Idempotência

Uma operação **idempotente** produz o mesmo resultado independente de quantas vezes é executada:

```bash
# Executar múltiplas vezes produz o mesmo resultado
terraform apply
terraform apply  # Nenhuma mudança
terraform apply  # Nenhuma mudança
```

---

## Primeiros Passos

### 1. Escolha uma Ferramenta

| Ferramenta | Tipo | Cloud | Linguagem |
|------------|------|-------|-----------|
| Terraform | Declarativo | Multi-cloud | HCL |
| Ansible | Imperativo | Multi-cloud | YAML |
| CloudFormation | Declarativo | AWS | JSON/YAML |
| Bicep | Declarativo | Azure | DSL |
| Pulumi | Declarativo | Multi-cloud | Python, Go, etc. |

> 📖 Veja mais detalhes em [Ferramentas de IaC](./tools-overview.md)

### 2. Configure o Ambiente

```bash
# Exemplo: Instalando Terraform
# Linux/macOS
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform

# Verificar instalação
terraform --version
```

### 3. Crie seu Primeiro Projeto

```bash
# Estrutura básica de projeto Terraform
mkdir meu-projeto-iac
cd meu-projeto-iac

# Criar arquivos iniciais
touch main.tf
touch variables.tf
touch outputs.tf
touch providers.tf
```

### 4. Defina a Infraestrutura

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

```hcl
# variables.tf
variable "aws_region" {
  description = "Região AWS para deploy"
  type        = string
  default     = "us-east-1"
}
```

### 5. Execute o Workflow

```bash
# Inicializar o projeto
terraform init

# Visualizar mudanças planejadas
terraform plan

# Aplicar as mudanças
terraform apply

# Destruir recursos (quando necessário)
terraform destroy
```

---

## Fluxo de Trabalho Típico

```
┌─────────────────┐
│   Escrever      │
│   Código IaC    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Validar       │
│   (lint/test)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Code Review   │
│   (Pull Request)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Plan          │
│   (dry-run)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Apply         │
│   (deploy)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Monitorar     │
│   e Iterar      │
└─────────────────┘
```

---

## Próximos Passos

- 📘 [Melhores Práticas em IaC](./best-practices.md)
- 🔧 [Visão Geral das Ferramentas](./tools-overview.md)
- 💡 [Exemplos Práticos](./examples/terraform-s3-setup.md)

---

## Referências

- [HashiCorp Learn](https://learn.hashicorp.com/terraform)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Azure Architecture Center](https://docs.microsoft.com/azure/architecture/)
- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)