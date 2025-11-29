# Ferramentas de IaC

Este documento apresenta ferramentas como Terraform, Ansible, CloudFormation, e Pulumi, pontuando casos de uso e melhores práticas.

---

## Índice

1. [Visão Geral](#visão-geral)
2. [Terraform](#terraform)
3. [Ansible](#ansible)
4. [AWS CloudFormation](#aws-cloudformation)
5. [Azure Bicep](#azure-bicep)
6. [Pulumi](#pulumi)
7. [Comparativo](#comparativo)
8. [Como Escolher](#como-escolher)

---

## Visão Geral

### Categorias de Ferramentas IaC

```
┌─────────────────────────────────────────────────────────────┐
│                    Ferramentas de IaC                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐    ┌─────────────────────────────┐    │
│  │ Provisionamento │    │ Gerenciamento de Configuração│    │
│  │                 │    │                             │    │
│  │ • Terraform     │    │ • Ansible                   │    │
│  │ • CloudFormation│    │ • Chef                      │    │
│  │ • Bicep         │    │ • Puppet                    │    │
│  │ • Pulumi        │    │ • SaltStack                 │    │
│  └─────────────────┘    └─────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Terraform

### O que é?

Terraform é uma ferramenta de IaC declarativa, open-source, desenvolvida pela HashiCorp. Permite provisionar infraestrutura em múltiplos provedores de nuvem usando a linguagem HCL (HashiCorp Configuration Language).

### Características

| Característica | Descrição |
|----------------|-----------|
| **Tipo** | Declarativo |
| **Linguagem** | HCL (HashiCorp Configuration Language) |
| **Estado** | Gerenciado (local ou remoto) |
| **Multi-cloud** | ✅ Sim |
| **Licença** | BSL 1.1 (anteriormente MPL 2.0) |

### Arquitetura

```
┌─────────────────────────────────────────────────┐
│                 Terraform CLI                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│  │   Core   │──▶│ Providers│──▶│ Resources│   │
│  └──────────┘   └──────────┘   └──────────┘   │
│       │                                        │
│       ▼                                        │
│  ┌──────────┐                                  │
│  │  State   │                                  │
│  └──────────┘                                  │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Exemplo Básico

```hcl
# providers.tf
terraform {
  required_version = ">= 1.0.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      ManagedBy = "Terraform"
    }
  }
}
```

```hcl
# main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "main-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count = length(var.public_subnets)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
    Tier = "Public"
  }
}
```

### Comandos Essenciais

```bash
# Inicialização
terraform init

# Formatação
terraform fmt

# Validação
terraform validate

# Planejamento
terraform plan -out=tfplan

# Aplicação
terraform apply tfplan

# Destruição
terraform destroy

# Gerenciamento de Estado
terraform state list
terraform state show aws_vpc.main
terraform import aws_vpc.main vpc-123456
```

### Quando Usar

✅ **Use Terraform quando:**
- Precisar de suporte multi-cloud
- Quiser separar provisionamento de configuração
- Precisar de gerenciamento de estado robusto
- Tiver equipes que preferem linguagem declarativa

❌ **Evite quando:**
- Precisar apenas de gerenciamento de configuração
- Tiver equipe pequena usando apenas uma cloud (considere ferramentas nativas)

---

## Ansible

### O que é?

Ansible é uma ferramenta de automação open-source para gerenciamento de configuração, provisionamento de aplicações e orquestração. Usa YAML para definir playbooks e não requer agentes.

### Características

| Característica | Descrição |
|----------------|-----------|
| **Tipo** | Imperativo/Procedural |
| **Linguagem** | YAML |
| **Agentes** | Não requer (agentless) |
| **Conexão** | SSH/WinRM |
| **Licença** | GPL 3.0 |

### Arquitetura

```
┌─────────────────────────────────────────────────┐
│              Ansible Control Node                │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│  │ Playbooks│──▶│ Inventory│──▶│ Modules  │   │
│  └──────────┘   └──────────┘   └──────────┘   │
│                       │                        │
│                       ▼                        │
│              ┌──────────────┐                  │
│              │  SSH/WinRM   │                  │
│              └──────────────┘                  │
│                       │                        │
└───────────────────────┼─────────────────────────┘
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
      ┌────────┐   ┌────────┐   ┌────────┐
      │ Host 1 │   │ Host 2 │   │ Host N │
      └────────┘   └────────┘   └────────┘
```

### Exemplo Básico

```yaml
# inventory.yml
all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.1.10
        web2:
          ansible_host: 192.168.1.11
    databases:
      hosts:
        db1:
          ansible_host: 192.168.1.20
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
```

```yaml
# playbook.yml
---
- name: Configurar servidores web
  hosts: webservers
  become: true
  
  vars:
    nginx_port: 80
    app_user: www-data
  
  tasks:
    - name: Atualizar cache do apt
      apt:
        update_cache: true
        cache_valid_time: 3600
    
    - name: Instalar Nginx
      apt:
        name: nginx
        state: present
    
    - name: Copiar configuração do Nginx
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: Reiniciar Nginx
    
    - name: Garantir que Nginx está rodando
      service:
        name: nginx
        state: started
        enabled: true
  
  handlers:
    - name: Reiniciar Nginx
      service:
        name: nginx
        state: restarted
```

```yaml
# roles/webserver/tasks/main.yml
---
- name: Instalar dependências
  apt:
    name:
      - nginx
      - python3
      - python3-pip
    state: present

- name: Criar diretório da aplicação
  file:
    path: /var/www/app
    state: directory
    owner: "{{ app_user }}"
    mode: '0755'

- name: Deploy da aplicação
  copy:
    src: app/
    dest: /var/www/app/
    owner: "{{ app_user }}"
```

### Comandos Essenciais

```bash
# Verificar sintaxe
ansible-playbook playbook.yml --syntax-check

# Dry run
ansible-playbook playbook.yml --check

# Executar playbook
ansible-playbook playbook.yml

# Limitar hosts
ansible-playbook playbook.yml --limit webservers

# Executar com verbosidade
ansible-playbook playbook.yml -vvv

# Comando ad-hoc
ansible webservers -m ping
ansible all -m shell -a "uptime"
```

### Quando Usar

✅ **Use Ansible quando:**
- Precisar configurar servidores existentes
- Quiser automação sem agentes
- Precisar de orquestração de aplicações
- Tiver ambiente híbrido (on-premise + cloud)

❌ **Evite quando:**
- Precisar provisionar infraestrutura cloud (use Terraform)
- Precisar de gerenciamento de estado complexo

---

## AWS CloudFormation

### O que é?

AWS CloudFormation é o serviço nativo da AWS para IaC. Permite modelar e provisionar recursos AWS usando templates JSON ou YAML.

### Características

| Característica | Descrição |
|----------------|-----------|
| **Tipo** | Declarativo |
| **Linguagem** | JSON/YAML |
| **Estado** | Gerenciado pela AWS |
| **Multi-cloud** | ❌ Apenas AWS |
| **Custo** | Gratuito (paga pelos recursos) |

### Exemplo Básico

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'VPC com subnets públicas e privadas'

Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod
  
  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
    Description: CIDR block for VPC

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

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Select [0, !Cidr [!Ref VpcCIDR, 4, 8]]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-public-1

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-igw

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-public-rt

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub ${EnvironmentName}-VpcId

  PublicSubnet1Id:
    Description: Public Subnet 1 ID
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub ${EnvironmentName}-PublicSubnet1Id
```

### Comandos Essenciais

```bash
# Validar template
aws cloudformation validate-template --template-body file://template.yaml

# Criar stack
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --parameters ParameterKey=EnvironmentName,ParameterValue=prod

# Atualizar stack
aws cloudformation update-stack \
  --stack-name my-stack \
  --template-body file://template.yaml

# Deletar stack
aws cloudformation delete-stack --stack-name my-stack

# Listar stacks
aws cloudformation list-stacks

# Descrever stack
aws cloudformation describe-stacks --stack-name my-stack
```

### Quando Usar

✅ **Use CloudFormation quando:**
- Trabalhar exclusivamente com AWS
- Precisar de integração profunda com serviços AWS
- Quiser evitar dependências externas
- Precisar de drift detection nativo

❌ **Evite quando:**
- Precisar de suporte multi-cloud
- A sintaxe JSON/YAML se tornar muito verbosa

---

## Azure Bicep

### O que é?

Bicep é uma linguagem específica de domínio (DSL) para deploy de recursos Azure. É uma abstração sobre ARM templates com sintaxe mais limpa e recursos avançados.

### Características

| Característica | Descrição |
|----------------|-----------|
| **Tipo** | Declarativo |
| **Linguagem** | Bicep DSL |
| **Estado** | Gerenciado pelo Azure |
| **Multi-cloud** | ❌ Apenas Azure |
| **Licença** | MIT |

### Exemplo Básico

```bicep
// main.bicep
targetScope = 'subscription'

@description('Nome do ambiente')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

@description('Localização dos recursos')
param location string = 'brazilsouth'

@description('Tags padrão para recursos')
param tags object = {
  Environment: environment
  ManagedBy: 'Bicep'
}

// Resource Group
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: 'rg-app-${environment}'
  location: location
  tags: tags
}

// Deploy de módulos no Resource Group
module network 'modules/network.bicep' = {
  scope: rg
  name: 'networkDeploy'
  params: {
    environment: environment
    location: location
    tags: tags
  }
}

module storage 'modules/storage.bicep' = {
  scope: rg
  name: 'storageDeploy'
  params: {
    environment: environment
    location: location
    tags: tags
  }
}

output vnetId string = network.outputs.vnetId
output storageAccountName string = storage.outputs.storageAccountName
```

```bicep
// modules/network.bicep
@description('Nome do ambiente')
param environment string

@description('Localização')
param location string

@description('Tags')
param tags object

var vnetName = 'vnet-${environment}'
var subnetName = 'snet-default'

resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: vnetName
  location: location
  tags: tags
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
    ]
  }
}

resource nsg 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'nsg-${environment}'
  location: location
  tags: tags
  properties: {
    securityRules: [
      {
        name: 'AllowHTTPS'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}

output vnetId string = vnet.id
output subnetId string = vnet.properties.subnets[0].id
```

### Comandos Essenciais

```bash
# Instalar Bicep CLI
az bicep install

# Compilar para ARM
az bicep build --file main.bicep

# Decompile ARM para Bicep
az bicep decompile --file template.json

# Deploy
az deployment sub create \
  --location brazilsouth \
  --template-file main.bicep \
  --parameters environment=prod

# What-if (dry run)
az deployment sub what-if \
  --location brazilsouth \
  --template-file main.bicep

# Validar
az deployment sub validate \
  --location brazilsouth \
  --template-file main.bicep
```

### Quando Usar

✅ **Use Bicep quando:**
- Trabalhar exclusivamente com Azure
- Quiser sintaxe mais limpa que ARM
- Precisar de módulos reutilizáveis
- Migrar de ARM templates

❌ **Evite quando:**
- Precisar de suporte multi-cloud
- Equipe não tiver experiência com Azure

---

## Pulumi

### O que é?

Pulumi permite definir infraestrutura usando linguagens de programação reais como Python, TypeScript, Go, e C#. Oferece a expressividade de linguagens completas com gerenciamento de estado.

### Características

| Característica | Descrição |
|----------------|-----------|
| **Tipo** | Declarativo (com linguagens imperativas) |
| **Linguagem** | Python, TypeScript, Go, C#, Java |
| **Estado** | Pulumi Cloud ou self-hosted |
| **Multi-cloud** | ✅ Sim |
| **Licença** | Apache 2.0 |

### Exemplo Básico (Python)

```python
# __main__.py
import pulumi
import pulumi_aws as aws

# Configuração
config = pulumi.Config()
environment = config.get("environment") or "dev"

# VPC
vpc = aws.ec2.Vpc(
    "main-vpc",
    cidr_block="10.0.0.0/16",
    enable_dns_hostnames=True,
    enable_dns_support=True,
    tags={
        "Name": f"vpc-{environment}",
        "Environment": environment,
        "ManagedBy": "Pulumi",
    },
)

# Internet Gateway
igw = aws.ec2.InternetGateway(
    "main-igw",
    vpc_id=vpc.id,
    tags={"Name": f"igw-{environment}"},
)

# Subnets públicas
public_subnets = []
azs = aws.get_availability_zones(state="available")

for i, az in enumerate(azs.names[:2]):
    subnet = aws.ec2.Subnet(
        f"public-subnet-{i + 1}",
        vpc_id=vpc.id,
        cidr_block=f"10.0.{i + 1}.0/24",
        availability_zone=az,
        map_public_ip_on_launch=True,
        tags={
            "Name": f"public-subnet-{i + 1}",
            "Tier": "Public",
        },
    )
    public_subnets.append(subnet)

# Route Table
public_rt = aws.ec2.RouteTable(
    "public-rt",
    vpc_id=vpc.id,
    routes=[
        aws.ec2.RouteTableRouteArgs(
            cidr_block="0.0.0.0/0",
            gateway_id=igw.id,
        ),
    ],
    tags={"Name": f"public-rt-{environment}"},
)

# Route Table Associations
for i, subnet in enumerate(public_subnets):
    aws.ec2.RouteTableAssociation(
        f"public-rta-{i + 1}",
        subnet_id=subnet.id,
        route_table_id=public_rt.id,
    )

# Outputs
pulumi.export("vpc_id", vpc.id)
pulumi.export("public_subnet_ids", [s.id for s in public_subnets])
```

### Exemplo Básico (TypeScript)

```typescript
// index.ts
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const config = new pulumi.Config();
const environment = config.get("environment") || "dev";

// VPC
const vpc = new aws.ec2.Vpc("main-vpc", {
    cidrBlock: "10.0.0.0/16",
    enableDnsHostnames: true,
    enableDnsSupport: true,
    tags: {
        Name: `vpc-${environment}`,
        Environment: environment,
        ManagedBy: "Pulumi",
    },
});

// Internet Gateway
const igw = new aws.ec2.InternetGateway("main-igw", {
    vpcId: vpc.id,
    tags: { Name: `igw-${environment}` },
});

// Public Subnets
const azs = aws.getAvailabilityZones({ state: "available" });
const publicSubnets: aws.ec2.Subnet[] = [];

azs.then(zones => {
    zones.names.slice(0, 2).forEach((az, i) => {
        const subnet = new aws.ec2.Subnet(`public-subnet-${i + 1}`, {
            vpcId: vpc.id,
            cidrBlock: `10.0.${i + 1}.0/24`,
            availabilityZone: az,
            mapPublicIpOnLaunch: true,
            tags: {
                Name: `public-subnet-${i + 1}`,
                Tier: "Public",
            },
        });
        publicSubnets.push(subnet);
    });
});

// Exports
export const vpcId = vpc.id;
export const publicSubnetIds = publicSubnets.map(s => s.id);
```

### Comandos Essenciais

```bash
# Criar novo projeto
pulumi new aws-python

# Preview (plan)
pulumi preview

# Deploy
pulumi up

# Destruir
pulumi destroy

# Stack management
pulumi stack ls
pulumi stack select dev
pulumi stack output

# Config
pulumi config set environment prod
pulumi config set --secret db_password mysecret
```

### Quando Usar

✅ **Use Pulumi quando:**
- Equipe preferir linguagens de programação completas
- Precisar de lógica complexa (loops, condicionais)
- Quiser testes unitários da infraestrutura
- Precisar de suporte multi-cloud

❌ **Evite quando:**
- Equipe preferir DSLs declarativas
- Quiser minimizar dependências de runtime

---

## Comparativo

### Tabela Comparativa

| Característica | Terraform | Ansible | CloudFormation | Bicep | Pulumi |
|----------------|-----------|---------|----------------|-------|--------|
| **Tipo** | Declarativo | Imperativo | Declarativo | Declarativo | Declarativo |
| **Linguagem** | HCL | YAML | JSON/YAML | Bicep DSL | Python, TS, Go |
| **Multi-cloud** | ✅ | ✅ | ❌ | ❌ | ✅ |
| **Estado** | Gerenciado | Não tem | AWS | Azure | Gerenciado |
| **Curva Aprendizado** | Média | Baixa | Alta | Baixa | Média |
| **Comunidade** | Grande | Grande | Média | Crescendo | Média |
| **Licença** | BSL | GPL | Proprietária | MIT | Apache |

### Quando Usar Cada Uma

```
┌─────────────────────────────────────────────────────────────┐
│                    Árvore de Decisão                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Precisa de Multi-cloud?                                    │
│  ├── Sim ──▶ Terraform ou Pulumi                           │
│  └── Não ──▶ Qual cloud?                                   │
│              ├── AWS ──▶ CloudFormation ou Terraform       │
│              ├── Azure ──▶ Bicep ou Terraform              │
│              └── GCP ──▶ Terraform                         │
│                                                             │
│  Precisa configurar servidores?                            │
│  ├── Sim ──▶ Ansible                                       │
│  └── Não ──▶ Terraform/CloudFormation/Bicep                │
│                                                             │
│  Prefere linguagens de programação?                        │
│  ├── Sim ──▶ Pulumi                                        │
│  └── Não ──▶ Terraform (HCL) ou YAML                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Como Escolher

### Critérios de Decisão

1. **Ambiente de Nuvem**
   - Multi-cloud → Terraform, Pulumi
   - AWS only → CloudFormation, Terraform
   - Azure only → Bicep, Terraform

2. **Experiência da Equipe**
   - DevOps tradicionais → Terraform, Ansible
   - Desenvolvedores → Pulumi

3. **Complexidade do Projeto**
   - Simples → CloudFormation, Bicep
   - Complexo → Terraform, Pulumi

4. **Requisitos de Configuração**
   - Apenas provisionamento → Terraform
   - Configuração de servidores → Ansible
   - Ambos → Terraform + Ansible

### Combinações Populares

| Stack | Uso |
|-------|-----|
| Terraform + Ansible | Provisionar com TF, configurar com Ansible |
| Terraform + Packer | Criar AMIs e provisionar |
| CloudFormation + CodePipeline | CI/CD nativo AWS |
| Bicep + Azure DevOps | CI/CD nativo Azure |
| Pulumi + GitHub Actions | CI/CD multi-cloud |

---

## Próximos Passos

- 📘 [Introdução ao IaC](./introduction.md)
- 📖 [Melhores Práticas](./best-practices.md)
- 💡 [Exemplos Práticos](./examples/terraform-s3-setup.md)