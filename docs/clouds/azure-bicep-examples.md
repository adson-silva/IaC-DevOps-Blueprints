# Azure Bicep - Exemplos e Melhores Práticas

## Visão Geral

Este documento fornece exemplos detalhados e melhores práticas para implementação de infraestrutura Azure usando Bicep, alinhados com o Azure Cloud Adoption Framework e os princípios TOGAF.

## Arquitetura de Referência Azure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Azure Subscription                                │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         Resource Group                                 │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    Virtual Network                               │  │  │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │  │  │
│  │  │  │   Gateway   │  │ Application │  │   Data      │              │  │  │
│  │  │  │   Subnet    │  │   Subnet    │  │   Subnet    │              │  │  │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘              │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │     AKS     │  │  App Service│  │  Azure SQL  │  │   Storage   │  │  │
│  │  │   Cluster   │  │             │  │             │  │   Account   │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Estrutura de Projeto Bicep

```
infrastructure/
├── main.bicep                    # Template principal
├── main.bicepparam               # Arquivo de parâmetros
├── modules/                      # Módulos reutilizáveis
│   ├── networking/
│   │   ├── vnet.bicep
│   │   ├── nsg.bicep
│   │   └── bastion.bicep
│   ├── compute/
│   │   ├── aks.bicep
│   │   ├── vm.bicep
│   │   └── appservice.bicep
│   ├── data/
│   │   ├── sql.bicep
│   │   ├── cosmos.bicep
│   │   └── storage.bicep
│   └── security/
│       ├── keyvault.bicep
│       └── identity.bicep
├── environments/
│   ├── dev.bicepparam
│   ├── staging.bicepparam
│   └── prod.bicepparam
└── scripts/
    ├── deploy.sh
    └── validate.sh
```

---

## 1. Azure Kubernetes Service (AKS)

### Arquitetura AKS Enterprise

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AKS Cluster                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        Control Plane (Managed)                         │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │   API       │  │   etcd      │  │   Scheduler │                    │  │
│  │  │   Server    │  │             │  │             │                    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                          Node Pools                                    │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐                     │  │
│  │  │    System Pool      │  │    User Pool        │                     │  │
│  │  │  ┌────┐ ┌────┐      │  │  ┌────┐ ┌────┐ ┌────┐│                    │  │
│  │  │  │Node│ │Node│      │  │  │Node│ │Node│ │Node││                    │  │
│  │  │  └────┘ └────┘      │  │  └────┘ └────┘ └────┘│                    │  │
│  │  └─────────────────────┘  └─────────────────────┘                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Exemplo Bicep - AKS Completo

```bicep
// ============================================================================
// Módulo AKS Enterprise
// ============================================================================

@description('Nome do cluster AKS')
param clusterName string

@description('Localização do cluster')
param location string = resourceGroup().location

@description('Ambiente de deploy')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('Versão do Kubernetes')
param kubernetesVersion string = '1.28'

@description('ID da subnet para os nodes')
param subnetId string

@description('ID do Log Analytics Workspace')
param logAnalyticsWorkspaceId string

@description('CIDR para services do Kubernetes')
param serviceCidr string = '10.0.0.0/16'

@description('IP do DNS service')
param dnsServiceIP string = '10.0.0.10'

// Variáveis
var aksName = 'aks-${clusterName}-${environment}'
var nodeResourceGroup = 'MC_${resourceGroup().name}_${aksName}_${location}'

// User Assigned Managed Identity
resource aksIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: '${aksName}-identity'
  location: location
  tags: {
    Environment: environment
    ManagedBy: 'bicep'
  }
}

// AKS Cluster
resource aks 'Microsoft.ContainerService/managedClusters@2023-08-01' = {
  name: aksName
  location: location
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${aksIdentity.id}': {}
    }
  }
  sku: {
    name: 'Base'
    tier: environment == 'prod' ? 'Standard' : 'Free'
  }
  properties: {
    kubernetesVersion: kubernetesVersion
    dnsPrefix: aksName
    nodeResourceGroup: nodeResourceGroup
    
    // Configuração de rede
    networkProfile: {
      networkPlugin: 'azure'
      networkPolicy: 'calico'
      serviceCidr: serviceCidr
      dnsServiceIP: dnsServiceIP
      loadBalancerSku: 'standard'
      outboundType: 'loadBalancer'
    }
    
    // Pool de sistema
    agentPoolProfiles: [
      {
        name: 'system'
        count: environment == 'prod' ? 3 : 2
        vmSize: 'Standard_D4s_v5'
        osType: 'Linux'
        osSKU: 'AzureLinux'
        mode: 'System'
        vnetSubnetID: subnetId
        enableAutoScaling: true
        minCount: environment == 'prod' ? 3 : 1
        maxCount: environment == 'prod' ? 5 : 3
        maxPods: 50
        availabilityZones: environment == 'prod' ? ['1', '2', '3'] : []
        nodeTaints: [
          'CriticalAddonsOnly=true:NoSchedule'
        ]
        nodeLabels: {
          'nodepool-type': 'system'
          environment: environment
        }
      }
    ]
    
    // Configurações de segurança
    aadProfile: {
      managed: true
      enableAzureRBAC: true
      adminGroupObjectIDs: []
    }
    
    apiServerAccessProfile: {
      enablePrivateCluster: environment == 'prod'
      authorizedIPRanges: environment != 'prod' ? ['0.0.0.0/0'] : []
    }
    
    // Add-ons
    addonProfiles: {
      omsagent: {
        enabled: true
        config: {
          logAnalyticsWorkspaceResourceID: logAnalyticsWorkspaceId
        }
      }
      azureKeyvaultSecretsProvider: {
        enabled: true
        config: {
          enableSecretRotation: 'true'
          rotationPollInterval: '2m'
        }
      }
      azurepolicy: {
        enabled: true
      }
    }
    
    // Segurança
    enableRBAC: true
    disableLocalAccounts: environment == 'prod'
    
    // Auto-upgrade
    autoUpgradeProfile: {
      upgradeChannel: environment == 'prod' ? 'stable' : 'rapid'
    }
    
    // Maintenance window
    maintenanceWindow: {
      schedule: {
        daily: null
        weekly: {
          intervalWeeks: 1
          dayOfWeek: 'Saturday'
          startTime: '03:00'
        }
        absoluteMonthly: null
        relativeMonthly: null
      }
      notAllowedTime: [
        {
          start: '2024-12-20T00:00:00Z'
          end: '2024-12-31T23:59:59Z'
        }
      ]
    }
  }
  
  tags: {
    Environment: environment
    Application: clusterName
    ManagedBy: 'bicep'
  }
}

// Pool de usuário para workloads
resource userPool 'Microsoft.ContainerService/managedClusters/agentPools@2023-08-01' = {
  parent: aks
  name: 'user'
  properties: {
    count: environment == 'prod' ? 3 : 2
    vmSize: 'Standard_D8s_v5'
    osType: 'Linux'
    osSKU: 'AzureLinux'
    mode: 'User'
    vnetSubnetID: subnetId
    enableAutoScaling: true
    minCount: environment == 'prod' ? 2 : 1
    maxCount: environment == 'prod' ? 20 : 5
    maxPods: 50
    availabilityZones: environment == 'prod' ? ['1', '2', '3'] : []
    nodeLabels: {
      'nodepool-type': 'user'
      environment: environment
    }
  }
}

// Diagnostic Settings
resource aksDiagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: '${aksName}-diagnostics'
  scope: aks
  properties: {
    workspaceId: logAnalyticsWorkspaceId
    logs: [
      {
        categoryGroup: 'allLogs'
        enabled: true
      }
    ]
    metrics: [
      {
        category: 'AllMetrics'
        enabled: true
      }
    ]
  }
}

// Outputs
output aksId string = aks.id
output aksName string = aks.name
output kubeletIdentityObjectId string = aks.properties.identityProfile.kubeletidentity.objectId
output aksIdentityPrincipalId string = aksIdentity.properties.principalId
output nodeResourceGroup string = aks.properties.nodeResourceGroup
```

---

## 2. Azure Firewall

### Arquitetura Hub-Spoke com Firewall

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                Hub VNet                                      │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                          Azure Firewall                                │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │   Network   │  │ Application │  │    NAT      │                    │  │
│  │  │    Rules    │  │    Rules    │  │    Rules    │                    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│              │                    │                    │                     │
└──────────────┼────────────────────┼────────────────────┼─────────────────────┘
               │                    │                    │
     ┌─────────┘                    │                    └─────────┐
     ▼                              ▼                              ▼
┌─────────────┐              ┌─────────────┐              ┌─────────────┐
│  Spoke 1    │              │  Spoke 2    │              │  Spoke 3    │
│  (App A)    │              │  (App B)    │              │  (Shared)   │
└─────────────┘              └─────────────┘              └─────────────┘
```

### Exemplo Bicep - Azure Firewall

```bicep
// ============================================================================
// Módulo Azure Firewall
// ============================================================================

@description('Nome do Firewall')
param firewallName string

@description('Localização')
param location string = resourceGroup().location

@description('Ambiente')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('ID da subnet para o Firewall')
param firewallSubnetId string

@description('ID do Log Analytics Workspace')
param logAnalyticsWorkspaceId string

@description('Zonas de disponibilidade')
param availabilityZones array = environment == 'prod' ? ['1', '2', '3'] : []

// Variáveis
var fwName = 'fw-${firewallName}-${environment}'
var fwPolicyName = 'fwpol-${firewallName}-${environment}'

// IP Público
resource publicIP 'Microsoft.Network/publicIPAddresses@2023-05-01' = {
  name: '${fwName}-pip'
  location: location
  sku: {
    name: 'Standard'
    tier: 'Regional'
  }
  zones: availabilityZones
  properties: {
    publicIPAllocationMethod: 'Static'
    publicIPAddressVersion: 'IPv4'
    dnsSettings: {
      domainNameLabel: fwName
    }
  }
  tags: {
    Environment: environment
    ManagedBy: 'bicep'
  }
}

// Firewall Policy
resource firewallPolicy 'Microsoft.Network/firewallPolicies@2023-05-01' = {
  name: fwPolicyName
  location: location
  properties: {
    sku: {
      tier: environment == 'prod' ? 'Premium' : 'Standard'
    }
    threatIntelMode: 'Alert'
    threatIntelWhitelist: {
      fqdns: []
      ipAddresses: []
    }
    dnsSettings: {
      enableProxy: true
    }
    intrusionDetection: environment == 'prod' ? {
      mode: 'Alert'
      configuration: {
        signatureOverrides: []
        bypassTrafficSettings: []
      }
    } : null
  }
  tags: {
    Environment: environment
    ManagedBy: 'bicep'
  }
}

// Rule Collection Group - Network Rules
resource networkRuleGroup 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-05-01' = {
  parent: firewallPolicy
  name: 'NetworkRuleCollectionGroup'
  properties: {
    priority: 100
    ruleCollections: [
      {
        name: 'AllowAzureServices'
        priority: 100
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        action: {
          type: 'Allow'
        }
        rules: [
          {
            name: 'AllowAzureMonitor'
            ruleType: 'NetworkRule'
            sourceAddresses: ['10.0.0.0/8']
            destinationAddresses: ['AzureMonitor']
            destinationPorts: ['443']
            ipProtocols: ['TCP']
          }
          {
            name: 'AllowAzureKeyVault'
            ruleType: 'NetworkRule'
            sourceAddresses: ['10.0.0.0/8']
            destinationAddresses: ['AzureKeyVault']
            destinationPorts: ['443']
            ipProtocols: ['TCP']
          }
          {
            name: 'AllowAzureContainerRegistry'
            ruleType: 'NetworkRule'
            sourceAddresses: ['10.0.0.0/8']
            destinationAddresses: ['AzureContainerRegistry']
            destinationPorts: ['443']
            ipProtocols: ['TCP']
          }
        ]
      }
      {
        name: 'DenyInternetOutbound'
        priority: 1000
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        action: {
          type: 'Deny'
        }
        rules: [
          {
            name: 'DenyAllOutbound'
            ruleType: 'NetworkRule'
            sourceAddresses: ['10.0.0.0/8']
            destinationAddresses: ['*']
            destinationPorts: ['*']
            ipProtocols: ['Any']
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
  dependsOn: [networkRuleGroup]
  properties: {
    priority: 200
    ruleCollections: [
      {
        name: 'AllowAKS'
        priority: 100
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        action: {
          type: 'Allow'
        }
        rules: [
          {
            name: 'AllowAKSRequired'
            ruleType: 'ApplicationRule'
            sourceAddresses: ['10.0.0.0/8']
            protocols: [
              {
                protocolType: 'Https'
                port: 443
              }
            ]
            targetFqdns: [
              '*.hcp.${location}.azmk8s.io'
              'mcr.microsoft.com'
              '*.data.mcr.microsoft.com'
              'management.azure.com'
              'login.microsoftonline.com'
              '*.blob.core.windows.net'
              '*.cdn.mscr.io'
              '*.azurecr.io'
            ]
          }
          {
            name: 'AllowDocker'
            ruleType: 'ApplicationRule'
            sourceAddresses: ['10.0.0.0/8']
            protocols: [
              {
                protocolType: 'Https'
                port: 443
              }
            ]
            targetFqdns: [
              'docker.io'
              '*.docker.io'
              '*.docker.com'
            ]
          }
        ]
      }
      {
        name: 'AllowUpdates'
        priority: 200
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        action: {
          type: 'Allow'
        }
        rules: [
          {
            name: 'AllowUbuntuUpdates'
            ruleType: 'ApplicationRule'
            sourceAddresses: ['10.0.0.0/8']
            protocols: [
              {
                protocolType: 'Https'
                port: 443
              }
              {
                protocolType: 'Http'
                port: 80
              }
            ]
            targetFqdns: [
              'security.ubuntu.com'
              'azure.archive.ubuntu.com'
              'archive.ubuntu.com'
            ]
          }
        ]
      }
    ]
  }
}

// Azure Firewall
resource firewall 'Microsoft.Network/azureFirewalls@2023-05-01' = {
  name: fwName
  location: location
  zones: availabilityZones
  properties: {
    sku: {
      name: 'AZFW_VNet'
      tier: environment == 'prod' ? 'Premium' : 'Standard'
    }
    firewallPolicy: {
      id: firewallPolicy.id
    }
    ipConfigurations: [
      {
        name: 'fw-ipconfig'
        properties: {
          subnet: {
            id: firewallSubnetId
          }
          publicIPAddress: {
            id: publicIP.id
          }
        }
      }
    ]
  }
  tags: {
    Environment: environment
    ManagedBy: 'bicep'
  }
}

// Diagnostic Settings
resource firewallDiagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: '${fwName}-diagnostics'
  scope: firewall
  properties: {
    workspaceId: logAnalyticsWorkspaceId
    logs: [
      {
        categoryGroup: 'allLogs'
        enabled: true
      }
    ]
    metrics: [
      {
        category: 'AllMetrics'
        enabled: true
      }
    ]
  }
}

// Outputs
output firewallId string = firewall.id
output firewallPrivateIp string = firewall.properties.ipConfigurations[0].properties.privateIPAddress
output firewallPublicIp string = publicIP.properties.ipAddress
```

---

## 3. Convenções de Nomenclatura Azure

### Padrão de Nomenclatura

| Recurso | Padrão | Exemplo |
|---------|--------|---------|
| Resource Group | `rg-<projeto>-<ambiente>-<região>` | `rg-webapp-prod-eastus2` |
| Virtual Network | `vnet-<projeto>-<ambiente>-<número>` | `vnet-webapp-prod-001` |
| Subnet | `snet-<função>-<ambiente>` | `snet-app-prod` |
| Storage Account | `st<projeto><ambiente><número>` | `stwebappprod001` |
| AKS | `aks-<projeto>-<ambiente>` | `aks-webapp-prod` |
| Key Vault | `kv-<projeto>-<ambiente>-<número>` | `kv-webapp-prod-001` |
| App Service | `app-<projeto>-<ambiente>-<número>` | `app-webapp-prod-001` |
| Azure Firewall | `fw-<projeto>-<ambiente>` | `fw-hub-prod` |
| NSG | `nsg-<subnet>-<ambiente>` | `nsg-app-prod` |

### Função de Nomenclatura Bicep

```bicep
// ============================================================================
// Funções de Nomenclatura
// ============================================================================

@description('Nome do projeto')
param projectName string

@description('Ambiente')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('Região abreviada')
param regionCode string = 'eastus2'

// Função para gerar nomes padronizados
var naming = {
  resourceGroup: 'rg-${projectName}-${environment}-${regionCode}'
  virtualNetwork: 'vnet-${projectName}-${environment}-001'
  subnet: 'snet-${projectName}-${environment}'
  storageAccount: 'st${replace(projectName, '-', '')}${environment}001'
  keyVault: 'kv-${projectName}-${environment}-001'
  aksCluster: 'aks-${projectName}-${environment}'
  appService: 'app-${projectName}-${environment}-001'
  logAnalytics: 'log-${projectName}-${environment}'
  appInsights: 'appi-${projectName}-${environment}'
}

output names object = naming
```

---

## 4. Segurança e Compliance

### Checklist de Segurança Azure

| Categoria | Controle | Recurso Bicep |
|-----------|----------|---------------|
| Identity | Managed Identity | `Microsoft.ManagedIdentity` |
| Network | NSG com deny all | `Microsoft.Network/networkSecurityGroups` |
| Network | Private Endpoints | `Microsoft.Network/privateEndpoints` |
| Data | Encryption at rest | `Microsoft.Storage/storageAccounts` |
| Secrets | Key Vault | `Microsoft.KeyVault/vaults` |
| Logging | Diagnostic Settings | `Microsoft.Insights/diagnosticSettings` |
| Policy | Azure Policy | `Microsoft.Authorization/policyAssignments` |
| Monitoring | Azure Monitor | `Microsoft.Insights/components` |

### Exemplo - Key Vault Seguro

```bicep
// ============================================================================
// Key Vault com Configurações de Segurança
// ============================================================================

@description('Nome do Key Vault')
param keyVaultName string

@description('Localização')
param location string = resourceGroup().location

@description('Ambiente')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('ID do subnet para Private Endpoint')
param subnetId string

@description('IDs dos objetos com acesso ao Key Vault')
param accessPolicies array = []

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: keyVaultName
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    
    // Configurações de segurança
    enabledForDeployment: false
    enabledForDiskEncryption: false
    enabledForTemplateDeployment: false
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true
    enableRbacAuthorization: true
    
    // Configurações de rede
    publicNetworkAccess: environment == 'prod' ? 'Disabled' : 'Enabled'
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
      ipRules: []
      virtualNetworkRules: environment != 'prod' ? [
        {
          id: subnetId
          ignoreMissingVnetServiceEndpoint: false
        }
      ] : []
    }
  }
  tags: {
    Environment: environment
    ManagedBy: 'bicep'
  }
}

// Private Endpoint (produção)
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = if (environment == 'prod') {
  name: '${keyVaultName}-pe'
  location: location
  properties: {
    subnet: {
      id: subnetId
    }
    privateLinkServiceConnections: [
      {
        name: '${keyVaultName}-connection'
        properties: {
          privateLinkServiceId: keyVault.id
          groupIds: ['vault']
        }
      }
    ]
  }
  tags: {
    Environment: environment
    ManagedBy: 'bicep'
  }
}

// Outputs
output keyVaultId string = keyVault.id
output keyVaultUri string = keyVault.properties.vaultUri
output keyVaultName string = keyVault.name
```

---

## 5. CI/CD com Bicep

### Pipeline Azure DevOps

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/**

parameters:
  - name: environment
    displayName: 'Target Environment'
    type: string
    default: 'dev'
    values:
      - dev
      - staging
      - prod

variables:
  azureServiceConnection: 'Azure-ServiceConnection'
  resourceGroupName: 'rg-myapp-${{ parameters.environment }}'
  location: 'eastus2'

stages:
  - stage: Validate
    displayName: 'Validate Bicep'
    jobs:
      - job: ValidateBicep
        displayName: 'Lint and Validate'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: AzureCLI@2
            displayName: 'Build Bicep'
            inputs:
              azureSubscription: $(azureServiceConnection)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep build --file infrastructure/main.bicep
                
          - task: AzureCLI@2
            displayName: 'Validate Deployment'
            inputs:
              azureSubscription: $(azureServiceConnection)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az deployment group validate \
                  --resource-group $(resourceGroupName) \
                  --template-file infrastructure/main.bicep \
                  --parameters infrastructure/environments/${{ parameters.environment }}.bicepparam

  - stage: WhatIf
    displayName: 'What-If Analysis'
    dependsOn: Validate
    jobs:
      - job: WhatIf
        displayName: 'Preview Changes'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: AzureCLI@2
            displayName: 'What-If'
            inputs:
              azureSubscription: $(azureServiceConnection)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az deployment group what-if \
                  --resource-group $(resourceGroupName) \
                  --template-file infrastructure/main.bicep \
                  --parameters infrastructure/environments/${{ parameters.environment }}.bicepparam

  - stage: Deploy
    displayName: 'Deploy Infrastructure'
    dependsOn: WhatIf
    condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
    jobs:
      - deployment: DeployInfra
        displayName: 'Deploy to ${{ parameters.environment }}'
        environment: ${{ parameters.environment }}
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self
                - task: AzureCLI@2
                  displayName: 'Deploy Bicep'
                  inputs:
                    azureSubscription: $(azureServiceConnection)
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment group create \
                        --resource-group $(resourceGroupName) \
                        --template-file infrastructure/main.bicep \
                        --parameters infrastructure/environments/${{ parameters.environment }}.bicepparam \
                        --name "deploy-$(Build.BuildNumber)"
```

---

## Referências

- [Azure Bicep Documentation](https://docs.microsoft.com/azure/azure-resource-manager/bicep/)
- [Azure Cloud Adoption Framework](https://docs.microsoft.com/azure/cloud-adoption-framework/)
- [Azure Well-Architected Framework](https://docs.microsoft.com/azure/architecture/framework/)
- [Azure Naming Conventions](https://docs.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [TOGAF Standard](https://www.opengroup.org/togaf)