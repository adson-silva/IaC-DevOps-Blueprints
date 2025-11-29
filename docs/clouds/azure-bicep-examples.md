# Azure Bicep Examples

Exemplos e templates para recursos como AKS, e Firewall no Azure usando Bicep.

---

## Índice

1. [Introdução ao Bicep](#introdução-ao-bicep)
2. [Configuração do Ambiente](#configuração-do-ambiente)
3. [Networking](#networking)
4. [Azure Kubernetes Service (AKS)](#azure-kubernetes-service-aks)
5. [Azure Firewall](#azure-firewall)
6. [Storage Account](#storage-account)
7. [Azure SQL Database](#azure-sql-database)
8. [Módulos e Boas Práticas](#módulos-e-boas-práticas)

---

## Introdução ao Bicep

### O que é Bicep?

Bicep é uma linguagem específica de domínio (DSL) para deploy de recursos Azure. É uma camada de abstração sobre ARM templates com sintaxe mais limpa e legível.

### Bicep vs ARM Templates

| Característica | Bicep | ARM (JSON) |
|----------------|-------|------------|
| Sintaxe | Limpa e concisa | Verbosa |
| Módulos | Nativo | Linked templates |
| Intellisense | Excelente | Limitado |
| Type safety | Sim | Não |
| Curva de aprendizado | Baixa | Alta |

### Estrutura Básica

```bicep
// Escopo do deploy
targetScope = 'resourceGroup'

// Parâmetros
@description('Nome do ambiente')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

// Variáveis
var resourcePrefix = 'app-${environment}'

// Recursos
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: '${resourcePrefix}storage'
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}

// Outputs
output storageAccountId string = storageAccount.id
```

---

## Configuração do Ambiente

### Instalar Azure CLI e Bicep

```bash
# Instalar Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Instalar extensão Bicep
az bicep install

# Verificar versão
az bicep version

# Atualizar Bicep
az bicep upgrade
```

### Extensão VS Code

```bash
# Extensão oficial do Bicep
code --install-extension ms-azuretools.vscode-bicep
```

### Comandos Principais

```bash
# Compilar Bicep para ARM
az bicep build --file main.bicep

# Decompile ARM para Bicep
az bicep decompile --file template.json

# Deploy no Resource Group
az deployment group create \
  --resource-group rg-myapp-dev \
  --template-file main.bicep \
  --parameters environment=dev

# Deploy na Subscription
az deployment sub create \
  --location brazilsouth \
  --template-file main.bicep

# What-if (preview das mudanças)
az deployment group what-if \
  --resource-group rg-myapp-dev \
  --template-file main.bicep
```

---

## Networking

### Virtual Network Completa

```bicep
// networking.bicep
@description('Nome do ambiente')
param environment string = 'dev'

@description('Localização dos recursos')
param location string = resourceGroup().location

@description('Address space da VNet')
param vnetAddressPrefix string = '10.0.0.0/16'

var vnetName = 'vnet-${environment}'

// Virtual Network
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        vnetAddressPrefix
      ]
    }
    subnets: [
      {
        name: 'snet-web'
        properties: {
          addressPrefix: '10.0.1.0/24'
          networkSecurityGroup: {
            id: nsgWeb.id
          }
          serviceEndpoints: [
            {
              service: 'Microsoft.Storage'
            }
            {
              service: 'Microsoft.Sql'
            }
          ]
        }
      }
      {
        name: 'snet-app'
        properties: {
          addressPrefix: '10.0.2.0/24'
          networkSecurityGroup: {
            id: nsgApp.id
          }
        }
      }
      {
        name: 'snet-db'
        properties: {
          addressPrefix: '10.0.3.0/24'
          networkSecurityGroup: {
            id: nsgDb.id
          }
          delegations: []
        }
      }
      {
        name: 'AzureBastionSubnet'
        properties: {
          addressPrefix: '10.0.255.0/27'
        }
      }
    ]
  }
}

// NSG Web Tier
resource nsgWeb 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'nsg-${environment}-web'
  location: location
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
      {
        name: 'AllowHTTP'
        properties: {
          priority: 110
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '80'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}

// NSG App Tier
resource nsgApp 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'nsg-${environment}-app'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowFromWebTier'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '8080'
          sourceAddressPrefix: '10.0.1.0/24'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}

// NSG Database Tier
resource nsgDb 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'nsg-${environment}-db'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowFromAppTier'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '1433'
          sourceAddressPrefix: '10.0.2.0/24'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}

// Outputs
output vnetId string = vnet.id
output webSubnetId string = vnet.properties.subnets[0].id
output appSubnetId string = vnet.properties.subnets[1].id
output dbSubnetId string = vnet.properties.subnets[2].id
```

---

## Azure Kubernetes Service (AKS)

### Cluster AKS Completo

```bicep
// aks.bicep
@description('Nome do cluster AKS')
param clusterName string = 'aks-app'

@description('Localização')
param location string = resourceGroup().location

@description('Versão do Kubernetes')
param kubernetesVersion string = '1.28'

@description('Tamanho dos nodes')
param nodeVmSize string = 'Standard_D2s_v3'

@description('Número mínimo de nodes')
@minValue(1)
@maxValue(100)
param nodeCount int = 3

@description('ID da subnet para o AKS')
param subnetId string

@description('ID da identidade gerenciada')
param identityId string

// Log Analytics Workspace
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'log-${clusterName}'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

// Container Registry
resource acr 'Microsoft.ContainerRegistry/registries@2023-07-01' = {
  name: replace('acr${clusterName}', '-', '')
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    adminUserEnabled: false
  }
}

// AKS Cluster
resource aksCluster 'Microsoft.ContainerService/managedClusters@2023-08-01' = {
  name: clusterName
  location: location
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${identityId}': {}
    }
  }
  properties: {
    dnsPrefix: clusterName
    kubernetesVersion: kubernetesVersion
    
    // Network Profile
    networkProfile: {
      networkPlugin: 'azure'
      networkPolicy: 'azure'
      loadBalancerSku: 'standard'
      serviceCidr: '10.100.0.0/16'
      dnsServiceIP: '10.100.0.10'
    }
    
    // Default Node Pool
    agentPoolProfiles: [
      {
        name: 'system'
        count: nodeCount
        vmSize: nodeVmSize
        osType: 'Linux'
        mode: 'System'
        vnetSubnetID: subnetId
        enableAutoScaling: true
        minCount: 2
        maxCount: 5
        availabilityZones: [
          '1'
          '2'
          '3'
        ]
        nodeTaints: []
        nodeLabels: {
          'nodepool-type': 'system'
        }
      }
    ]
    
    // Add-ons
    addonProfiles: {
      omsagent: {
        enabled: true
        config: {
          logAnalyticsWorkspaceResourceID: logAnalytics.id
        }
      }
      azurepolicy: {
        enabled: true
      }
      azureKeyvaultSecretsProvider: {
        enabled: true
        config: {
          enableSecretRotation: 'true'
          rotationPollInterval: '2m'
        }
      }
    }
    
    // Security
    enableRBAC: true
    aadProfile: {
      managed: true
      enableAzureRBAC: true
    }
    
    // Maintenance
    autoUpgradeProfile: {
      upgradeChannel: 'patch'
    }
  }
}

// User Node Pool
resource userNodePool 'Microsoft.ContainerService/managedClusters/agentPools@2023-08-01' = {
  parent: aksCluster
  name: 'user'
  properties: {
    count: 2
    vmSize: 'Standard_D4s_v3'
    osType: 'Linux'
    mode: 'User'
    vnetSubnetID: subnetId
    enableAutoScaling: true
    minCount: 2
    maxCount: 10
    availabilityZones: [
      '1'
      '2'
      '3'
    ]
    nodeLabels: {
      'nodepool-type': 'user'
    }
  }
}

// Role assignment para ACR
resource acrPullRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(aksCluster.id, acr.id, 'acrpull')
  scope: acr
  properties: {
    principalId: aksCluster.properties.identityProfile.kubeletidentity.objectId
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '7f951dda-4ed3-4680-a7ca-43fe172d538d')
    principalType: 'ServicePrincipal'
  }
}

// Outputs
output aksClusterId string = aksCluster.id
output aksClusterFqdn string = aksCluster.properties.fqdn
output acrLoginServer string = acr.properties.loginServer
output kubeletIdentity string = aksCluster.properties.identityProfile.kubeletidentity.objectId
```

---

## Azure Firewall

### Firewall com Policy

```bicep
// firewall.bicep
@description('Nome do Firewall')
param firewallName string = 'afw-hub'

@description('Localização')
param location string = resourceGroup().location

@description('ID da subnet do Firewall')
param firewallSubnetId string

@description('SKU do Firewall')
@allowed(['Standard', 'Premium'])
param firewallSku string = 'Standard'

// Public IP para o Firewall
resource firewallPip 'Microsoft.Network/publicIPAddresses@2023-05-01' = {
  name: 'pip-${firewallName}'
  location: location
  sku: {
    name: 'Standard'
    tier: 'Regional'
  }
  properties: {
    publicIPAllocationMethod: 'Static'
    publicIPAddressVersion: 'IPv4'
  }
  zones: [
    '1'
    '2'
    '3'
  ]
}

// Firewall Policy
resource firewallPolicy 'Microsoft.Network/firewallPolicies@2023-05-01' = {
  name: '${firewallName}-policy'
  location: location
  properties: {
    sku: {
      tier: firewallSku
    }
    threatIntelMode: 'Alert'
    threatIntelWhitelist: {
      fqdns: []
      ipAddresses: []
    }
    dnsSettings: {
      enableProxy: true
    }
  }
}

// Rule Collection Group - Network Rules
resource networkRuleGroup 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-05-01' = {
  parent: firewallPolicy
  name: 'NetworkRuleCollectionGroup'
  properties: {
    priority: 200
    ruleCollections: [
      {
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        name: 'AllowInfrastructure'
        priority: 100
        action: {
          type: 'Allow'
        }
        rules: [
          {
            ruleType: 'NetworkRule'
            name: 'AllowAzureMonitor'
            ipProtocols: [
              'TCP'
            ]
            sourceAddresses: [
              '10.0.0.0/16'
            ]
            destinationAddresses: [
              'AzureMonitor'
            ]
            destinationPorts: [
              '443'
            ]
          }
          {
            ruleType: 'NetworkRule'
            name: 'AllowAzureStorage'
            ipProtocols: [
              'TCP'
            ]
            sourceAddresses: [
              '10.0.0.0/16'
            ]
            destinationAddresses: [
              'Storage'
            ]
            destinationPorts: [
              '443'
            ]
          }
        ]
      }
    ]
  }
}

// Rule Collection Group - Application Rules
resource applicationRuleGroup 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-05-01' = {
  parent: firewallPolicy
  name: 'ApplicationRuleCollectionGroup'
  dependsOn: [
    networkRuleGroup
  ]
  properties: {
    priority: 300
    ruleCollections: [
      {
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        name: 'AllowWebTraffic'
        priority: 100
        action: {
          type: 'Allow'
        }
        rules: [
          {
            ruleType: 'ApplicationRule'
            name: 'AllowMicrosoft'
            protocols: [
              {
                protocolType: 'Https'
                port: 443
              }
            ]
            sourceAddresses: [
              '10.0.0.0/16'
            ]
            targetFqdns: [
              '*.microsoft.com'
              '*.azure.com'
              '*.windows.net'
            ]
          }
          {
            ruleType: 'ApplicationRule'
            name: 'AllowGitHub'
            protocols: [
              {
                protocolType: 'Https'
                port: 443
              }
            ]
            sourceAddresses: [
              '10.0.0.0/16'
            ]
            targetFqdns: [
              'github.com'
              '*.github.com'
              '*.githubusercontent.com'
            ]
          }
        ]
      }
      {
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        name: 'AllowAKS'
        priority: 200
        action: {
          type: 'Allow'
        }
        rules: [
          {
            ruleType: 'ApplicationRule'
            name: 'AllowAKSRequired'
            protocols: [
              {
                protocolType: 'Https'
                port: 443
              }
            ]
            sourceAddresses: [
              '10.0.0.0/16'
            ]
            targetFqdns: [
              '*.hcp.brazilsouth.azmk8s.io'
              'mcr.microsoft.com'
              '*.data.mcr.microsoft.com'
              'management.azure.com'
              'login.microsoftonline.com'
              'packages.microsoft.com'
              'acs-mirror.azureedge.net'
            ]
          }
        ]
      }
    ]
  }
}

// Azure Firewall
resource firewall 'Microsoft.Network/azureFirewalls@2023-05-01' = {
  name: firewallName
  location: location
  zones: [
    '1'
    '2'
    '3'
  ]
  properties: {
    sku: {
      name: 'AZFW_VNet'
      tier: firewallSku
    }
    firewallPolicy: {
      id: firewallPolicy.id
    }
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: firewallSubnetId
          }
          publicIPAddress: {
            id: firewallPip.id
          }
        }
      }
    ]
  }
}

// Outputs
output firewallPrivateIp string = firewall.properties.ipConfigurations[0].properties.privateIPAddress
output firewallPublicIp string = firewallPip.properties.ipAddress
output firewallId string = firewall.id
```

---

## Storage Account

### Storage Account Seguro

```bicep
// storage.bicep
@description('Nome da Storage Account')
@minLength(3)
@maxLength(24)
param storageAccountName string

@description('Localização')
param location string = resourceGroup().location

@description('SKU da Storage Account')
@allowed(['Standard_LRS', 'Standard_GRS', 'Standard_ZRS', 'Premium_LRS'])
param storageSku string = 'Standard_ZRS'

@description('ID da subnet para Private Endpoint')
param subnetId string

@description('IPs permitidos')
param allowedIpRanges array = []

// Storage Account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    allowBlobPublicAccess: false
    allowSharedKeyAccess: true
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    
    // Network Rules
    networkAcls: {
      bypass: 'AzureServices'
      defaultAction: 'Deny'
      ipRules: [for ip in allowedIpRanges: {
        value: ip
        action: 'Allow'
      }]
      virtualNetworkRules: [
        {
          id: subnetId
          action: 'Allow'
        }
      ]
    }
    
    // Encryption
    encryption: {
      keySource: 'Microsoft.Storage'
      services: {
        blob: {
          enabled: true
          keyType: 'Account'
        }
        file: {
          enabled: true
          keyType: 'Account'
        }
      }
    }
  }
}

// Blob Services
resource blobServices 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    containerDeleteRetentionPolicy: {
      enabled: true
      days: 7
    }
    deleteRetentionPolicy: {
      enabled: true
      days: 7
    }
    isVersioningEnabled: true
    changeFeed: {
      enabled: true
    }
  }
}

// Containers
resource dataContainer 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  parent: blobServices
  name: 'data'
  properties: {
    publicAccess: 'None'
  }
}

resource logsContainer 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  parent: blobServices
  name: 'logs'
  properties: {
    publicAccess: 'None'
  }
}

// Private Endpoint
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'pe-${storageAccountName}'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: 'pe-${storageAccountName}-blob'
        properties: {
          privateLinkServiceId: storageAccount.id
          groupIds: [
            'blob'
          ]
        }
      }
    ]
  }
}

// Outputs
output storageAccountId string = storageAccount.id
output storageAccountName string = storageAccount.name
output blobEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

---

## Azure SQL Database

### SQL Database com Elastic Pool

```bicep
// sql-database.bicep
@description('Nome do servidor SQL')
param sqlServerName string

@description('Localização')
param location string = resourceGroup().location

@description('Username do admin')
@secure()
param adminUsername string

@description('Password do admin')
@secure()
param adminPassword string

@description('ID da subnet para Private Endpoint')
param subnetId string

// SQL Server
resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: sqlServerName
  location: location
  properties: {
    administratorLogin: adminUsername
    administratorLoginPassword: adminPassword
    version: '12.0'
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'
  }
}

// Elastic Pool
resource elasticPool 'Microsoft.Sql/servers/elasticPools@2023-05-01-preview' = {
  parent: sqlServer
  name: 'ep-${sqlServerName}'
  location: location
  sku: {
    name: 'GP_Gen5'
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 2
  }
  properties: {
    perDatabaseSettings: {
      minCapacity: 0
      maxCapacity: 2
    }
    maxSizeBytes: 34359738368 // 32GB
    zoneRedundant: true
  }
}

// Database
resource database 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: 'appdb'
  location: location
  sku: {
    name: 'ElasticPool'
    tier: 'GeneralPurpose'
  }
  properties: {
    elasticPoolId: elasticPool.id
    collation: 'SQL_Latin1_General_CP1_CI_AS'
    maxSizeBytes: 10737418240 // 10GB
    zoneRedundant: true
  }
}

// Auditing
resource auditingSettings 'Microsoft.Sql/servers/auditingSettings@2023-05-01-preview' = {
  parent: sqlServer
  name: 'default'
  properties: {
    state: 'Enabled'
    isAzureMonitorTargetEnabled: true
    retentionDays: 90
  }
}

// Vulnerability Assessment
resource vulnerabilityAssessment 'Microsoft.Sql/servers/vulnerabilityAssessments@2023-05-01-preview' = {
  parent: sqlServer
  name: 'default'
  properties: {
    recurringScans: {
      isEnabled: true
      emailSubscriptionAdmins: true
    }
  }
}

// Private Endpoint
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'pe-${sqlServerName}'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: 'pe-${sqlServerName}-sql'
        properties: {
          privateLinkServiceId: sqlServer.id
          groupIds: [
            'sqlServer'
          ]
        }
      }
    ]
  }
}

// Outputs
output sqlServerFqdn string = sqlServer.properties.fullyQualifiedDomainName
output databaseName string = database.name
```

---

## Módulos e Boas Práticas

### Estrutura de Projeto

```
infrastructure/
├── main.bicep
├── parameters/
│   ├── dev.bicepparam
│   ├── staging.bicepparam
│   └── prod.bicepparam
├── modules/
│   ├── networking/
│   │   ├── vnet.bicep
│   │   └── nsg.bicep
│   ├── compute/
│   │   ├── aks.bicep
│   │   └── vm.bicep
│   ├── storage/
│   │   └── storage.bicep
│   └── security/
│       ├── keyvault.bicep
│       └── firewall.bicep
└── .azure-pipelines/
    └── deploy.yml
```

### Main.bicep com Módulos

```bicep
// main.bicep
targetScope = 'subscription'

@description('Nome do ambiente')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('Localização')
param location string = 'brazilsouth'

// Resource Group
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: 'rg-app-${environment}'
  location: location
  tags: {
    Environment: environment
    ManagedBy: 'Bicep'
  }
}

// Networking
module networking 'modules/networking/vnet.bicep' = {
  scope: rg
  name: 'networkingDeploy'
  params: {
    environment: environment
    location: location
  }
}

// Storage
module storage 'modules/storage/storage.bicep' = {
  scope: rg
  name: 'storageDeploy'
  params: {
    storageAccountName: 'stapp${environment}${uniqueString(rg.id)}'
    location: location
    subnetId: networking.outputs.webSubnetId
  }
}

// AKS
module aks 'modules/compute/aks.bicep' = {
  scope: rg
  name: 'aksDeploy'
  params: {
    clusterName: 'aks-app-${environment}'
    location: location
    subnetId: networking.outputs.appSubnetId
    identityId: identity.outputs.identityId
  }
}
```

### Pipeline de Deploy

```yaml
# .azure-pipelines/deploy.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/**

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: azure-credentials

stages:
  - stage: Validate
    jobs:
      - job: ValidateBicep
        steps:
          - task: AzureCLI@2
            displayName: 'Validate Bicep'
            inputs:
              azureSubscription: 'Azure-Connection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep build --file infrastructure/main.bicep

  - stage: DeployDev
    dependsOn: Validate
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployDev
        environment: 'development'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: 'Deploy to Dev'
                  inputs:
                    azureSubscription: 'Azure-Connection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment sub create \
                        --location brazilsouth \
                        --template-file infrastructure/main.bicep \
                        --parameters infrastructure/parameters/dev.bicepparam
```

---

## Próximos Passos

- 📘 [Introdução ao IaC](../iac/introduction.md)
- 📖 [Melhores Práticas](../iac/best-practices.md)
- ☁️ [Guias AWS](./aws-guides.md)
- 🔒 [Conformidade ISO 27001](../compliance/iso-27001.md)