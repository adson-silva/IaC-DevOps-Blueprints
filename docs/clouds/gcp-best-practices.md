# Google Cloud Platform - Melhores Práticas de IaC

## Visão Geral

Este documento apresenta as melhores práticas para implementação de infraestrutura no Google Cloud Platform (GCP) usando Infraestrutura como Código, alinhadas com o Google Cloud Architecture Framework e os princípios TOGAF.

## Arquitetura de Referência GCP

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              GCP Organization                                │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                              Folders                                   │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │ Production  │  │   Staging   │  │ Development │                    │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                    │  │
│  └─────────┼────────────────┼────────────────┼───────────────────────────┘  │
│            │                │                │                               │
│  ┌─────────┼────────────────┼────────────────┼───────────────────────────┐  │
│  │         ▼                ▼                ▼                            │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │  Project    │  │  Project    │  │  Project    │                    │  │
│  │  │  (prod)     │  │  (staging)  │  │   (dev)     │                    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                     Shared VPC Host Project                      │  │  │
│  │  │  ┌─────────────────────────────────────────────────────────┐    │  │  │
│  │  │  │                    Shared VPC                            │    │  │  │
│  │  │  └─────────────────────────────────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Pilares do Cloud Architecture Framework

| Pilar | Descrição | Práticas IaC |
|-------|-----------|--------------|
| **Excelência Operacional** | Monitoramento e operações | Cloud Monitoring, Cloud Logging |
| **Segurança** | Proteção de dados e sistemas | IAM, VPC Service Controls |
| **Confiabilidade** | Resiliência e recuperação | Regional/Multi-regional, Cloud DNS |
| **Eficiência de Performance** | Otimização de recursos | Committed Use Discounts, Autoscaling |
| **Otimização de Custos** | Gestão de custos | Preemptible VMs, Labels |

---

## 1. Load Balancing Regional e Global

### Arquitetura de Load Balancing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Global Load Balancer                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │   Anycast   │  │    SSL      │  │   Cloud     │                    │  │
│  │  │   IP        │──│   Proxy     │──│   Armor     │                    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │  │
│  └──────────────────────────────┬────────────────────────────────────────┘  │
│                                 │                                            │
│         ┌───────────────────────┼───────────────────────┐                   │
│         ▼                       ▼                       ▼                   │
│  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐           │
│  │  Region A   │         │  Region B   │         │  Region C   │           │
│  │  Backend    │         │  Backend    │         │  Backend    │           │
│  │  Service    │         │  Service    │         │  Service    │           │
│  └─────────────┘         └─────────────┘         └─────────────┘           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tipos de Load Balancers GCP

| Tipo | Escopo | Protocolo | Caso de Uso |
|------|--------|-----------|-------------|
| External HTTP(S) | Global | HTTP/HTTPS | Aplicações web |
| External TCP/UDP | Global | TCP/UDP | Gaming, streaming |
| Internal HTTP(S) | Regional | HTTP/HTTPS | Microservices internos |
| Internal TCP/UDP | Regional | TCP/UDP | Databases, caches |
| Network LB | Regional | TCP/UDP | Alta performance |

### Exemplo Terraform - Load Balancer Global

```hcl
#------------------------------------------------------------------------------
# External HTTP(S) Load Balancer
#------------------------------------------------------------------------------

# IP Address Global
resource "google_compute_global_address" "default" {
  name         = "${var.project_name}-global-ip"
  ip_version   = "IPV4"
  address_type = "EXTERNAL"
}

# SSL Certificate (Managed)
resource "google_compute_managed_ssl_certificate" "default" {
  name = "${var.project_name}-ssl-cert"

  managed {
    domains = [var.domain_name]
  }
}

# Backend Service
resource "google_compute_backend_service" "default" {
  name                  = "${var.project_name}-backend"
  protocol              = "HTTP"
  port_name             = "http"
  timeout_sec           = 30
  health_checks         = [google_compute_health_check.default.id]
  load_balancing_scheme = "EXTERNAL_MANAGED"
  
  enable_cdn = true
  cdn_policy {
    cache_mode                   = "CACHE_ALL_STATIC"
    default_ttl                  = 3600
    client_ttl                   = 7200
    max_ttl                      = 86400
    negative_caching             = true
    serve_while_stale            = 86400
    signed_url_cache_max_age_sec = 7200
  }

  security_policy = google_compute_security_policy.default.id

  dynamic "backend" {
    for_each = var.backend_groups
    content {
      group                 = backend.value
      balancing_mode        = "UTILIZATION"
      max_utilization       = 0.8
      capacity_scaler       = 1.0
    }
  }

  log_config {
    enable      = true
    sample_rate = 1.0
  }
}

# Health Check
resource "google_compute_health_check" "default" {
  name                = "${var.project_name}-health-check"
  check_interval_sec  = 5
  timeout_sec         = 5
  healthy_threshold   = 2
  unhealthy_threshold = 3

  http_health_check {
    port               = 80
    request_path       = "/health"
    proxy_header       = "NONE"
  }
}

# URL Map
resource "google_compute_url_map" "default" {
  name            = "${var.project_name}-url-map"
  default_service = google_compute_backend_service.default.id

  host_rule {
    hosts        = [var.domain_name]
    path_matcher = "main"
  }

  path_matcher {
    name            = "main"
    default_service = google_compute_backend_service.default.id

    path_rule {
      paths   = ["/api/*"]
      service = google_compute_backend_service.api.id
    }

    path_rule {
      paths   = ["/static/*"]
      service = google_compute_backend_bucket.static.id
    }
  }
}

# HTTPS Proxy
resource "google_compute_target_https_proxy" "default" {
  name             = "${var.project_name}-https-proxy"
  url_map          = google_compute_url_map.default.id
  ssl_certificates = [google_compute_managed_ssl_certificate.default.id]
  
  ssl_policy = google_compute_ssl_policy.default.id
}

# SSL Policy
resource "google_compute_ssl_policy" "default" {
  name            = "${var.project_name}-ssl-policy"
  profile         = "MODERN"
  min_tls_version = "TLS_1_2"
}

# Forwarding Rule
resource "google_compute_global_forwarding_rule" "https" {
  name                  = "${var.project_name}-https-rule"
  ip_protocol           = "TCP"
  port_range            = "443"
  target                = google_compute_target_https_proxy.default.id
  ip_address            = google_compute_global_address.default.id
  load_balancing_scheme = "EXTERNAL_MANAGED"
}

# HTTP to HTTPS Redirect
resource "google_compute_url_map" "http_redirect" {
  name = "${var.project_name}-http-redirect"

  default_url_redirect {
    https_redirect         = true
    redirect_response_code = "MOVED_PERMANENTLY_DEFAULT"
    strip_query            = false
  }
}

resource "google_compute_target_http_proxy" "http_redirect" {
  name    = "${var.project_name}-http-redirect-proxy"
  url_map = google_compute_url_map.http_redirect.id
}

resource "google_compute_global_forwarding_rule" "http_redirect" {
  name                  = "${var.project_name}-http-redirect-rule"
  ip_protocol           = "TCP"
  port_range            = "80"
  target                = google_compute_target_http_proxy.http_redirect.id
  ip_address            = google_compute_global_address.default.id
  load_balancing_scheme = "EXTERNAL_MANAGED"
}
```

---

## 2. Cloud Armor (WAF)

### Arquitetura de Segurança

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Cloud Armor Policy                                 │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                          Security Rules                                │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │   IP Allow  │  │  Geo Block  │  │    OWASP    │  │   Rate      │  │  │
│  │  │   List      │  │             │  │   Top 10    │  │   Limiting  │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    Backend Service Protection                          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - Cloud Armor

```hcl
#------------------------------------------------------------------------------
# Cloud Armor Security Policy
#------------------------------------------------------------------------------

resource "google_compute_security_policy" "default" {
  name        = "${var.project_name}-security-policy"
  description = "Security policy with WAF rules"

  # Regra padrão - permitir
  rule {
    action   = "allow"
    priority = "2147483647"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    description = "Default rule - allow all"
  }

  # Bloquear países específicos
  rule {
    action   = "deny(403)"
    priority = "1000"
    match {
      expr {
        expression = "origin.region_code == 'CN' || origin.region_code == 'RU'"
      }
    }
    description = "Block specific countries"
  }

  # Rate limiting
  rule {
    action   = "rate_based_ban"
    priority = "2000"
    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
    rate_limit_options {
      conform_action = "allow"
      exceed_action  = "deny(429)"
      rate_limit_threshold {
        count        = 100
        interval_sec = 60
      }
      ban_duration_sec = 600
      enforce_on_key   = "IP"
    }
    description = "Rate limiting - 100 requests per minute"
  }

  # OWASP Top 10 - SQL Injection
  rule {
    action   = "deny(403)"
    priority = "3000"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('sqli-v33-stable')"
      }
    }
    description = "SQL Injection protection"
  }

  # OWASP Top 10 - XSS
  rule {
    action   = "deny(403)"
    priority = "3001"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('xss-v33-stable')"
      }
    }
    description = "XSS protection"
  }

  # OWASP Top 10 - Local File Inclusion
  rule {
    action   = "deny(403)"
    priority = "3002"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('lfi-v33-stable')"
      }
    }
    description = "LFI protection"
  }

  # OWASP Top 10 - Remote File Inclusion
  rule {
    action   = "deny(403)"
    priority = "3003"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('rfi-v33-stable')"
      }
    }
    description = "RFI protection"
  }

  # OWASP Top 10 - Remote Code Execution
  rule {
    action   = "deny(403)"
    priority = "3004"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('rce-v33-stable')"
      }
    }
    description = "RCE protection"
  }

  # Bot Protection
  rule {
    action   = "deny(403)"
    priority = "4000"
    match {
      expr {
        expression = "evaluatePreconfiguredExpr('cve-canary')"
      }
    }
    description = "Known CVE protection"
  }

  # Adaptive Protection
  adaptive_protection_config {
    layer_7_ddos_defense_config {
      enable = true
      rule_visibility = "STANDARD"
    }
  }
}
```

---

## 3. IAM e Organization Policies

### Hierarquia de Políticas

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Organization                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │  Organization Policies (mais restritivas)                              │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                              Folders                                   │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Folder Policies (herdam + adicionam)                            │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                             Projects                                   │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Project Policies (herdam + adicionam)                           │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - Organization Policies

```hcl
#------------------------------------------------------------------------------
# Organization Policies
#------------------------------------------------------------------------------

# Restringir criação de External IPs
resource "google_organization_policy" "disable_external_ip" {
  org_id     = var.organization_id
  constraint = "constraints/compute.vmExternalIpAccess"

  list_policy {
    deny {
      all = true
    }
  }
}

# Restringir regiões permitidas
resource "google_organization_policy" "allowed_locations" {
  org_id     = var.organization_id
  constraint = "constraints/gcp.resourceLocations"

  list_policy {
    allow {
      values = [
        "in:us-locations",
        "in:southamerica-east1-locations"
      ]
    }
  }
}

# Exigir Uniform Bucket-Level Access
resource "google_organization_policy" "uniform_bucket_access" {
  org_id     = var.organization_id
  constraint = "constraints/storage.uniformBucketLevelAccess"

  boolean_policy {
    enforced = true
  }
}

# Desabilitar criação de Service Account Keys
resource "google_organization_policy" "disable_sa_key_creation" {
  org_id     = var.organization_id
  constraint = "constraints/iam.disableServiceAccountKeyCreation"

  boolean_policy {
    enforced = true
  }
}

# Restringir VPC Peering
resource "google_organization_policy" "restrict_vpc_peering" {
  org_id     = var.organization_id
  constraint = "constraints/compute.restrictVpcPeering"

  list_policy {
    allow {
      values = [
        "under:organizations/${var.organization_id}"
      ]
    }
  }
}

#------------------------------------------------------------------------------
# IAM - Custom Roles
#------------------------------------------------------------------------------

resource "google_organization_iam_custom_role" "developer" {
  org_id      = var.organization_id
  role_id     = "developer"
  title       = "Developer Role"
  description = "Role for development team with limited permissions"
  
  permissions = [
    "compute.instances.get",
    "compute.instances.list",
    "compute.instances.start",
    "compute.instances.stop",
    "container.clusters.get",
    "container.clusters.list",
    "container.pods.get",
    "container.pods.list",
    "logging.logEntries.list",
    "monitoring.timeSeries.list",
    "storage.buckets.get",
    "storage.buckets.list",
    "storage.objects.get",
    "storage.objects.list",
  ]
}

#------------------------------------------------------------------------------
# Workload Identity
#------------------------------------------------------------------------------

resource "google_service_account" "app" {
  account_id   = "app-${var.environment}"
  display_name = "Application Service Account"
  project      = var.project_id
}

resource "google_service_account_iam_binding" "workload_identity" {
  service_account_id = google_service_account.app.name
  role               = "roles/iam.workloadIdentityUser"
  
  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[${var.namespace}/${var.k8s_service_account}]"
  ]
}

resource "google_project_iam_member" "app_permissions" {
  for_each = toset([
    "roles/storage.objectViewer",
    "roles/secretmanager.secretAccessor",
    "roles/cloudtrace.agent",
    "roles/monitoring.metricWriter",
  ])
  
  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.app.email}"
}
```

---

## 4. GKE (Google Kubernetes Engine)

### Arquitetura GKE Enterprise

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              GKE Cluster                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                   Control Plane (Managed by Google)                    │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                    │  │
│  │  │   API       │  │   etcd      │  │   Cloud     │                    │  │
│  │  │   Server    │  │             │  │   Controller│                    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                    │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                           Node Pools                                   │  │
│  │  ┌─────────────────────────┐  ┌─────────────────────────┐             │  │
│  │  │     System Pool         │  │     Application Pool    │             │  │
│  │  │  (n2-standard-4)        │  │    (n2-standard-8)      │             │  │
│  │  │  ┌────┐ ┌────┐ ┌────┐   │  │  ┌────┐ ┌────┐ ┌────┐   │             │  │
│  │  │  │Node│ │Node│ │Node│   │  │  │Node│ │Node│ │Node│   │             │  │
│  │  │  └────┘ └────┘ └────┘   │  │  └────┘ └────┘ └────┘   │             │  │
│  │  └─────────────────────────┘  └─────────────────────────┘             │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Exemplo Terraform - GKE

```hcl
#------------------------------------------------------------------------------
# GKE Cluster
#------------------------------------------------------------------------------

resource "google_container_cluster" "primary" {
  name     = "${var.project_name}-${var.environment}"
  location = var.region
  
  # Usar release channel para updates automáticos
  release_channel {
    channel = var.environment == "prod" ? "STABLE" : "REGULAR"
  }

  # Remover node pool default
  remove_default_node_pool = true
  initial_node_count       = 1

  # Configuração de rede
  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.gke.name

  # IP Allocation para pods e services
  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  # Private cluster
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = var.environment == "prod"
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  # Master authorized networks
  master_authorized_networks_config {
    dynamic "cidr_blocks" {
      for_each = var.authorized_networks
      content {
        cidr_block   = cidr_blocks.value.cidr
        display_name = cidr_blocks.value.name
      }
    }
  }

  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Network policy
  network_policy {
    enabled  = true
    provider = "CALICO"
  }

  # Addons
  addons_config {
    http_load_balancing {
      disabled = false
    }
    horizontal_pod_autoscaling {
      disabled = false
    }
    network_policy_config {
      disabled = false
    }
    gce_persistent_disk_csi_driver_config {
      enabled = true
    }
    gcs_fuse_csi_driver_config {
      enabled = true
    }
  }

  # Binary Authorization
  binary_authorization {
    evaluation_mode = var.environment == "prod" ? "PROJECT_SINGLETON_POLICY_ENFORCE" : "DISABLED"
  }

  # Shielded nodes
  node_config {
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
  }

  # Logging e Monitoring
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }

  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS"]
    managed_prometheus {
      enabled = true
    }
  }

  # Maintenance window
  maintenance_policy {
    recurring_window {
      start_time = "2024-01-01T03:00:00Z"
      end_time   = "2024-01-01T07:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA"
    }
  }

  # Resource labels
  resource_labels = {
    environment = var.environment
    project     = var.project_name
    managed_by  = "terraform"
  }
}

#------------------------------------------------------------------------------
# Node Pool - Sistema
#------------------------------------------------------------------------------

resource "google_container_node_pool" "system" {
  name       = "system"
  cluster    = google_container_cluster.primary.name
  location   = var.region
  
  node_count = var.environment == "prod" ? 3 : 1

  autoscaling {
    min_node_count = var.environment == "prod" ? 3 : 1
    max_node_count = var.environment == "prod" ? 5 : 3
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  node_config {
    machine_type = "n2-standard-4"
    disk_size_gb = 100
    disk_type    = "pd-ssd"

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    service_account = google_service_account.gke_nodes.email

    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    # Taints para sistema
    taint {
      key    = "CriticalAddonsOnly"
      value  = "true"
      effect = "NO_SCHEDULE"
    }

    labels = {
      "node-type" = "system"
      environment = var.environment
    }

    tags = ["gke-node", "gke-${var.project_name}-${var.environment}"]
  }
}

#------------------------------------------------------------------------------
# Node Pool - Aplicação
#------------------------------------------------------------------------------

resource "google_container_node_pool" "application" {
  name     = "application"
  cluster  = google_container_cluster.primary.name
  location = var.region

  autoscaling {
    min_node_count = var.environment == "prod" ? 2 : 1
    max_node_count = var.environment == "prod" ? 20 : 5
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  node_config {
    machine_type = "n2-standard-8"
    disk_size_gb = 200
    disk_type    = "pd-ssd"

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    service_account = google_service_account.gke_nodes.email

    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    labels = {
      "node-type" = "application"
      environment = var.environment
    }

    tags = ["gke-node", "gke-${var.project_name}-${var.environment}"]
  }
}
```

---

## 5. Convenções de Nomenclatura GCP

### Padrão de Nomenclatura

| Recurso | Padrão | Exemplo |
|---------|--------|---------|
| Project | `<org>-<projeto>-<ambiente>` | `acme-webapp-prod` |
| VPC | `vpc-<projeto>-<ambiente>` | `vpc-webapp-prod` |
| Subnet | `snet-<função>-<região>` | `snet-app-us-east1` |
| GKE | `gke-<projeto>-<ambiente>` | `gke-webapp-prod` |
| Cloud SQL | `sql-<projeto>-<ambiente>-<tipo>` | `sql-webapp-prod-mysql` |
| GCS Bucket | `<org>-<projeto>-<ambiente>-<função>` | `acme-webapp-prod-assets` |
| Service Account | `sa-<função>-<ambiente>` | `sa-app-prod` |
| Firewall | `fw-<ação>-<protocolo>-<descrição>` | `fw-allow-https-ingress` |

---

## 6. CI/CD com Cloud Build

### Exemplo cloudbuild.yaml

```yaml
# cloudbuild.yaml
timeout: 3600s

substitutions:
  _ENVIRONMENT: 'dev'
  _REGION: 'us-east1'
  _PROJECT_ID: ${PROJECT_ID}

steps:
  # Validar Terraform
  - id: 'tf-validate'
    name: 'hashicorp/terraform:1.6'
    dir: 'infrastructure'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        terraform init -backend=false
        terraform validate
        terraform fmt -check -recursive

  # Security scan com tfsec
  - id: 'security-scan'
    name: 'aquasec/tfsec:latest'
    dir: 'infrastructure'
    args:
      - '--format'
      - 'json'
      - '--out'
      - '/workspace/tfsec-results.json'
      - '.'
    
  # Init Terraform
  - id: 'tf-init'
    name: 'hashicorp/terraform:1.6'
    dir: 'infrastructure'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        terraform init \
          -backend-config="bucket=${_PROJECT_ID}-tfstate" \
          -backend-config="prefix=${_ENVIRONMENT}"

  # Plan Terraform
  - id: 'tf-plan'
    name: 'hashicorp/terraform:1.6'
    dir: 'infrastructure'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        terraform plan \
          -var="project_id=${_PROJECT_ID}" \
          -var="environment=${_ENVIRONMENT}" \
          -var="region=${_REGION}" \
          -out=tfplan

  # Apply (apenas em main branch)
  - id: 'tf-apply'
    name: 'hashicorp/terraform:1.6'
    dir: 'infrastructure'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        if [ "${BRANCH_NAME}" = "main" ]; then
          terraform apply -auto-approve tfplan
        else
          echo "Skipping apply on non-main branch"
        fi

# Artifacts
artifacts:
  objects:
    location: 'gs://${_PROJECT_ID}-tfstate/plans/${BUILD_ID}/'
    paths:
      - 'infrastructure/tfplan'
      - 'tfsec-results.json'

# Options
options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'
```

---

## Referências

- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
- [GCP Best Practices](https://cloud.google.com/docs/enterprise/best-practices-for-enterprise-organizations)
- [Terraform Google Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [GKE Best Practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)
- [TOGAF Standard](https://www.opengroup.org/togaf)