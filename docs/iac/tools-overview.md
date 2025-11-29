# Ferramentas de Infraestrutura como Código (IaC)

## Visão Geral

Este documento apresenta uma análise detalhada das principais ferramentas de IaC, seus casos de uso, vantagens e desvantagens, seguindo a metodologia TOGAF para seleção de tecnologia.

## Matriz de Decisão de Ferramentas

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CRITÉRIOS DE SELEÇÃO TOGAF                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  Requisitos de Negócio    →    Capabilities Necessárias                     │
│           │                              │                                   │
│           ▼                              ▼                                   │
│  Análise de Gaps          →    Seleção de Ferramenta                        │
│           │                              │                                   │
│           ▼                              ▼                                   │
│  Avaliação de Riscos      →    Implementação                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Comparativo de Ferramentas

| Ferramenta | Tipo | Multi-Cloud | Linguagem | Curva de Aprendizado |
|------------|------|-------------|-----------|---------------------|
| Terraform | Declarativo | ✅ Sim | HCL | Média |
| Ansible | Imperativo/Declarativo | ✅ Sim | YAML | Baixa |
| CloudFormation | Declarativo | ❌ AWS only | JSON/YAML | Média |
| Pulumi | Declarativo | ✅ Sim | TypeScript/Python/Go | Média-Alta |
| Bicep | Declarativo | ❌ Azure only | Bicep | Baixa-Média |
| CDK | Declarativo | ✅ Parcial | TypeScript/Python | Alta |

---

## Terraform

### Visão Geral

Terraform é uma ferramenta de código aberto desenvolvida pela HashiCorp que permite definir infraestrutura usando a linguagem declarativa HCL (HashiCorp Configuration Language).

### Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                        Terraform CLI                             │
├─────────────────────────────────────────────────────────────────┤
│                         Terraform Core                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Parser    │  │   Planner   │  │   Applier   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
├─────────────────────────────────────────────────────────────────┤
│                          Providers                               │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐                 │
│  │  AWS   │  │ Azure  │  │  GCP   │  │  K8s   │  ...            │
│  └────────┘  └────────┘  └────────┘  └────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```

### Casos de Uso

- Provisionamento multi-cloud
- Infraestrutura complexa com muitas dependências
- Equipes que precisam de planos de execução antes de aplicar

### Exemplo Básico

```hcl
# Configuração do Provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Recurso VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "main-vpc"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# Subnet Pública
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
    Type = "public"
  }
}
```

### Prós e Contras

| Prós | Contras |
|------|---------|
| Multi-cloud nativo | Curva de aprendizado do HCL |
| Grande ecossistema de providers | Gerenciamento de estado pode ser complexo |
| Plano de execução antes de aplicar | Drift detection não é automático |
| Comunidade ativa | Módulos podem ter qualidade variada |
| State locking integrado | Loops e condicionais limitados |

---

## Ansible

### Visão Geral

Ansible é uma ferramenta de automação que usa YAML para definir configurações e pode ser usado tanto para gerenciamento de configuração quanto para provisionamento de infraestrutura.

### Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                      Control Node                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Ansible Engine                        │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │    │
│  │  │ Inventory│  │ Playbooks│  │  Modules │               │    │
│  │  └──────────┘  └──────────┘  └──────────┘               │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────┬───────────────────────────────────┘
                              │ SSH/WinRM
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
    ┌─────────┐          ┌─────────┐          ┌─────────┐
    │ Host 1  │          │ Host 2  │          │ Host N  │
    └─────────┘          └─────────┘          └─────────┘
```

### Casos de Uso

- Gerenciamento de configuração de servidores
- Deployment de aplicações
- Orquestração de tarefas
- Configuração pós-provisionamento

### Exemplo Básico

```yaml
---
- name: Configurar servidor web
  hosts: webservers
  become: yes
  
  vars:
    http_port: 80
    app_user: webapp
    
  tasks:
    - name: Instalar Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes
      tags:
        - nginx
        
    - name: Criar usuário da aplicação
      user:
        name: "{{ app_user }}"
        shell: /bin/bash
        create_home: yes
        
    - name: Configurar Nginx
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
        owner: root
        group: root
        mode: '0644'
      notify: Reiniciar Nginx
      
    - name: Garantir Nginx rodando
      service:
        name: nginx
        state: started
        enabled: yes
        
  handlers:
    - name: Reiniciar Nginx
      service:
        name: nginx
        state: restarted
```

### Prós e Contras

| Prós | Contras |
|------|---------|
| Sintaxe YAML simples | Sem estado built-in |
| Agentless (usa SSH) | Menos eficiente para infraestrutura complexa |
| Grande biblioteca de módulos | Idempotência depende do módulo |
| Bom para configuração | Debugging pode ser difícil |
| Curva de aprendizado baixa | Velocidade em ambientes grandes |

---

## AWS CloudFormation

### Visão Geral

CloudFormation é o serviço nativo da AWS para IaC, usando templates JSON ou YAML para definir recursos AWS.

### Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                     CloudFormation Service                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Stack     │  │  StackSets  │  │   Nested    │              │
│  │             │  │             │  │   Stacks    │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
├─────────────────────────────────────────────────────────────────┤
│                        Templates                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Parameters | Resources | Outputs | Conditions | Mappings│    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Casos de Uso

- Infraestrutura 100% AWS
- Integração nativa com serviços AWS
- Governança e compliance AWS

### Exemplo Básico

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'VPC com subnets públicas e privadas'

Parameters:
  EnvironmentName:
    Type: String
    Default: production
    AllowedValues:
      - development
      - staging
      - production
    Description: Nome do ambiente

  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
    Description: CIDR block da VPC

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-vpc
        - Key: Environment
          Value: !Ref EnvironmentName

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-public-subnet

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-igw

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

Outputs:
  VpcId:
    Description: ID da VPC
    Value: !Ref VPC
    Export:
      Name: !Sub ${EnvironmentName}-VpcId
```

### Prós e Contras

| Prós | Contras |
|------|---------|
| Integração nativa AWS | Apenas AWS |
| Sem custo adicional | Sintaxe verbosa |
| Drift detection | Rollback pode ser lento |
| StackSets para multi-região | Debugging difícil |
| Change sets para preview | Limitações em loops |

---

## Pulumi

### Visão Geral

Pulumi permite definir infraestrutura usando linguagens de programação reais como TypeScript, Python, Go e C#.

### Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                       Pulumi CLI                                 │
├─────────────────────────────────────────────────────────────────┤
│                      Pulumi Engine                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Language   │  │  Resource   │  │    State    │              │
│  │    Host     │  │   Plugin    │  │   Backend   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
├─────────────────────────────────────────────────────────────────┤
│                       Providers                                  │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐                 │
│  │  AWS   │  │ Azure  │  │  GCP   │  │  K8s   │  ...            │
│  └────────┘  └────────┘  └────────┘  └────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```

### Casos de Uso

- Equipes de desenvolvimento que preferem linguagens familiares
- Lógica complexa que beneficia de programação real
- Integração com código de aplicação

### Exemplo Básico (TypeScript)

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

// Configuração
const config = new pulumi.Config();
const environment = config.get("environment") || "development";

// VPC
const vpc = new aws.ec2.Vpc("main-vpc", {
    cidrBlock: "10.0.0.0/16",
    enableDnsHostnames: true,
    enableDnsSupport: true,
    tags: {
        Name: `${environment}-vpc`,
        Environment: environment,
        ManagedBy: "pulumi",
    },
});

// Subnets públicas
const publicSubnets: aws.ec2.Subnet[] = [];
const availabilityZones = ["us-east-1a", "us-east-1b"];

availabilityZones.forEach((az, index) => {
    const subnet = new aws.ec2.Subnet(`public-subnet-${index}`, {
        vpcId: vpc.id,
        cidrBlock: `10.0.${index + 1}.0/24`,
        availabilityZone: az,
        mapPublicIpOnLaunch: true,
        tags: {
            Name: `${environment}-public-${az}`,
            Type: "public",
        },
    });
    publicSubnets.push(subnet);
});

// Export
export const vpcId = vpc.id;
export const subnetIds = publicSubnets.map(s => s.id);
```

### Prós e Contras

| Prós | Contras |
|------|---------|
| Linguagens de programação reais | Curva de aprendizado alta |
| Loops, condicionais, funções | Dependência de runtime |
| Tipagem forte | Menos documentação que Terraform |
| Testes unitários nativos | Estado na cloud Pulumi por padrão |
| IDE support completo | Comunidade menor |

---

## Azure Bicep

### Visão Geral

Bicep é uma DSL (Domain Specific Language) da Microsoft para deploy de recursos Azure, sendo uma abstração sobre ARM templates.

### Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                         Bicep Files                              │
│                          (.bicep)                                │
└────────────────────────────────┬────────────────────────────────┘
                                 │ Compilation
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        ARM Templates                             │
│                          (.json)                                 │
└────────────────────────────────┬────────────────────────────────┘
                                 │ Deployment
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Azure Resource Manager                         │
└─────────────────────────────────────────────────────────────────┘
```

### Casos de Uso

- Infraestrutura 100% Azure
- Migração de ARM templates
- Equipes que usam VS Code com extensões Azure

### Exemplo Básico

```bicep
// Parâmetros
@description('Nome do ambiente')
@allowed([
  'dev'
  'staging'
  'prod'
])
param environment string = 'dev'

@description('Localização dos recursos')
param location string = resourceGroup().location

// Variáveis
var vnetName = 'vnet-${environment}-${location}'
var subnetName = 'snet-${environment}-main'

// Virtual Network
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: vnetName
  location: location
  tags: {
    Environment: environment
    ManagedBy: 'bicep'
  }
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.0.1.0/24'
          privateEndpointNetworkPolicies: 'Disabled'
        }
      }
    ]
  }
}

// Storage Account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'st${environment}${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    networkAcls: {
      defaultAction: 'Deny'
      virtualNetworkRules: [
        {
          id: vnet.properties.subnets[0].id
          action: 'Allow'
        }
      ]
    }
  }
}

// Outputs
output vnetId string = vnet.id
output storageAccountName string = storageAccount.name
```

### Prós e Contras

| Prós | Contras |
|------|---------|
| Sintaxe limpa vs ARM | Apenas Azure |
| Compilação para ARM | Novo, menos maturidade |
| VS Code support excelente | Documentação em evolução |
| Módulos reutilizáveis | Ferramentas de teste limitadas |
| Tipagem forte | Menos exemplos da comunidade |

---

## Matriz de Seleção por Cenário

| Cenário | Ferramenta Recomendada | Justificativa |
|---------|----------------------|---------------|
| Multi-cloud enterprise | Terraform | Maior compatibilidade e maturidade |
| Apenas AWS | CloudFormation ou Terraform | CFN para integração nativa, TF para flexibilidade |
| Apenas Azure | Bicep ou Terraform | Bicep para simplidade, TF para multi-cloud futuro |
| Apenas GCP | Terraform | Melhor suporte que Deployment Manager |
| Equipe dev-first | Pulumi | Linguagens familiares |
| Config management | Ansible | Especializado em configuração |
| Kubernetes-native | Terraform + Helm | Combinação complementar |

## Fluxo de Decisão

```
                         ┌────────────────────┐
                         │ Multi-cloud?       │
                         └─────────┬──────────┘
                          ┌────────┴────────┐
                        Sim│                │Não
                          ▼                 ▼
                 ┌──────────────┐   ┌────────────────┐
                 │  Terraform   │   │ Qual cloud?    │
                 │   ou Pulumi  │   └───────┬────────┘
                 └──────────────┘     ┌─────┼─────┐
                                    AWS│  Azure│ GCP│
                                      ▼      ▼    ▼
                              ┌──────────┐ ┌────┐┌───────────┐
                              │CFN ou TF │ │Bicep││ TF        │
                              └──────────┘ └────┘└───────────┘
```

## Referências

- [Terraform Documentation](https://www.terraform.io/docs)
- [Ansible Documentation](https://docs.ansible.com/)
- [AWS CloudFormation](https://aws.amazon.com/cloudformation/)
- [Pulumi Documentation](https://www.pulumi.com/docs/)
- [Azure Bicep](https://docs.microsoft.com/azure/azure-resource-manager/bicep/)