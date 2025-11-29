# GCP Best Practices

Melhores práticas e dicas para balanceadores regionais e políticas no Google Cloud.

---

## Índice

1. [Visão Geral do GCP](#visão-geral-do-gcp)
2. [Organização e Hierarquia](#organização-e-hierarquia)
3. [Networking](#networking)
4. [Load Balancing](#load-balancing)
5. [Compute Engine](#compute-engine)
6. [Google Kubernetes Engine (GKE)](#google-kubernetes-engine-gke)
7. [Cloud Storage](#cloud-storage)
8. [IAM e Segurança](#iam-e-segurança)
9. [Terraform para GCP](#terraform-para-gcp)

---

## Visão Geral do GCP

### Regiões e Zonas

```
┌─────────────────────────────────────────────────────────────┐
│                      GCP Global                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────┐   ┌─────────────────────┐        │
│   │ Region: us-central1 │   │ Region: southamerica│        │
│   │                     │   │ -east1 (São Paulo)  │        │
│   │ ┌────┐ ┌────┐ ┌────┐│   │ ┌────┐ ┌────┐ ┌────┐│        │
│   │ │ -a │ │ -b │ │ -c ││   │ │ -a │ │ -b │ │ -c ││        │
│   │ └────┘ └────┘ └────┘│   │ └────┘ └────┘ └────┘│        │
│   └─────────────────────┘   └─────────────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Serviços Principais

| Categoria | Serviço | Equivalente AWS |
|-----------|---------|-----------------|
| **Compute** | Compute Engine | EC2 |
| **Compute** | GKE | EKS |
| **Compute** | Cloud Functions | Lambda |
| **Storage** | Cloud Storage | S3 |
| **Database** | Cloud SQL | RDS |
| **Database** | Firestore | DynamoDB |
| **Network** | VPC | VPC |
| **Security** | IAM | IAM |

---

## Organização e Hierarquia

### Estrutura Organizacional

```
┌─────────────────────────────────────────────────────────────┐
│                     Organization                             │
│                   (empresa.com.br)                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌───────────────┐   ┌───────────────┐                    │
│   │  Folder: Prod │   │  Folder: Dev  │                    │
│   └───────┬───────┘   └───────┬───────┘                    │
│           │                   │                            │
│   ┌───────┴───────┐   ┌───────┴───────┐                    │
│   │Project: app-  │   │Project: app-  │                    │
│   │   prod        │   │   dev         │                    │
│   │               │   │               │                    │
│   │ • VMs         │   │ • VMs         │                    │
│   │ • Buckets     │   │ • Buckets     │                    │
│   │ • Databases   │   │ • Databases   │                    │
│   └───────────────┘   └───────────────┘                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Terraform: Organização

```hcl
# Organization Policy
resource "google_organization_policy" "require_os_login" {
  org_id     = var.org_id
  constraint = "compute.requireOsLogin"
  
  boolean_policy {
    enforced = true
  }
}

# Folder Structure
resource "google_folder" "production" {
  display_name = "Production"
  parent       = "organizations/${var.org_id}"
}

resource "google_folder" "development" {
  display_name = "Development"
  parent       = "organizations/${var.org_id}"
}

# Project
resource "google_project" "app_prod" {
  name            = "App Production"
  project_id      = "app-prod-${random_id.project.hex}"
  folder_id       = google_folder.production.name
  billing_account = var.billing_account
  
  labels = {
    environment = "production"
    managed_by  = "terraform"
  }
}

# Enable APIs
resource "google_project_service" "compute" {
  project = google_project.app_prod.project_id
  service = "compute.googleapis.com"
  
  disable_on_destroy = false
}

resource "google_project_service" "container" {
  project = google_project.app_prod.project_id
  service = "container.googleapis.com"
  
  disable_on_destroy = false
}
```

---

## Networking

### VPC Network

```
┌─────────────────────────────────────────────────────────────┐
│                    VPC Network                               │
│                  (10.0.0.0/16)                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │            Shared VPC (Host Project)                │   │
│  │                                                     │   │
│  │  ┌─────────────────┐   ┌─────────────────┐         │   │
│  │  │ Subnet: web     │   │ Subnet: app     │         │   │
│  │  │ 10.0.1.0/24     │   │ 10.0.2.0/24     │         │   │
│  │  │ us-central1     │   │ us-central1     │         │   │
│  │  └─────────────────┘   └─────────────────┘         │   │
│  │                                                     │   │
│  │  ┌─────────────────┐   ┌─────────────────┐         │   │
│  │  │ Subnet: db      │   │ Subnet: gke     │         │   │
│  │  │ 10.0.3.0/24     │   │ 10.0.4.0/22     │         │   │
│  │  │ us-central1     │   │ us-central1     │         │   │
│  │  └─────────────────┘   └─────────────────┘         │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Terraform: VPC Completa

```hcl
# VPC Network
resource "google_compute_network" "main" {
  name                            = "main-vpc"
  project                         = var.project_id
  auto_create_subnetworks         = false
  routing_mode                    = "REGIONAL"
  delete_default_routes_on_create = true
}

# Subnets
resource "google_compute_subnetwork" "web" {
  name          = "subnet-web"
  project       = var.project_id
  region        = var.region
  network       = google_compute_network.main.id
  ip_cidr_range = "10.0.1.0/24"
  
  private_ip_google_access = true
  
  log_config {
    aggregation_interval = "INTERVAL_5_SEC"
    flow_sampling        = 0.5
    metadata             = "INCLUDE_ALL_METADATA"
  }
}

resource "google_compute_subnetwork" "app" {
  name          = "subnet-app"
  project       = var.project_id
  region        = var.region
  network       = google_compute_network.main.id
  ip_cidr_range = "10.0.2.0/24"
  
  private_ip_google_access = true
}

resource "google_compute_subnetwork" "gke" {
  name          = "subnet-gke"
  project       = var.project_id
  region        = var.region
  network       = google_compute_network.main.id
  ip_cidr_range = "10.0.4.0/22"
  
  private_ip_google_access = true
  
  secondary_ip_range {
    range_name    = "gke-pods"
    ip_cidr_range = "10.1.0.0/16"
  }
  
  secondary_ip_range {
    range_name    = "gke-services"
    ip_cidr_range = "10.2.0.0/20"
  }
}

# Cloud Router (para NAT)
resource "google_compute_router" "main" {
  name    = "main-router"
  project = var.project_id
  region  = var.region
  network = google_compute_network.main.id
}

# Cloud NAT
resource "google_compute_router_nat" "main" {
  name                               = "main-nat"
  project                            = var.project_id
  router                             = google_compute_router.main.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
  
  log_config {
    enable = true
    filter = "ERRORS_ONLY"
  }
}

# Default route para internet
resource "google_compute_route" "default_internet" {
  name             = "default-internet-route"
  project          = var.project_id
  dest_range       = "0.0.0.0/0"
  network          = google_compute_network.main.id
  next_hop_gateway = "default-internet-gateway"
  priority         = 1000
}
```

### Firewall Rules

```hcl
# Allow internal traffic
resource "google_compute_firewall" "allow_internal" {
  name    = "allow-internal"
  project = var.project_id
  network = google_compute_network.main.id
  
  allow {
    protocol = "icmp"
  }
  
  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }
  
  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }
  
  source_ranges = ["10.0.0.0/8"]
}

# Allow SSH from IAP
resource "google_compute_firewall" "allow_ssh_iap" {
  name    = "allow-ssh-iap"
  project = var.project_id
  network = google_compute_network.main.id
  
  allow {
    protocol = "tcp"
    ports    = ["22"]
  }
  
  source_ranges = ["35.235.240.0/20"]  # IAP IP range
  target_tags   = ["allow-ssh"]
}

# Allow Health Checks
resource "google_compute_firewall" "allow_health_check" {
  name    = "allow-health-check"
  project = var.project_id
  network = google_compute_network.main.id
  
  allow {
    protocol = "tcp"
  }
  
  source_ranges = [
    "130.211.0.0/22",
    "35.191.0.0/16"
  ]
  
  target_tags = ["allow-health-check"]
}
```

---

## Load Balancing

### Tipos de Load Balancer

| Tipo | Escopo | Protocolo | Uso |
|------|--------|-----------|-----|
| **HTTP(S) LB** | Global | HTTP/HTTPS | Aplicações web |
| **TCP Proxy** | Global | TCP | Aplicações TCP |
| **SSL Proxy** | Global | SSL | SSL offloading |
| **Network LB** | Regional | TCP/UDP | Alta performance |
| **Internal TCP/UDP** | Regional | TCP/UDP | Tráfego interno |

### Terraform: HTTP(S) Load Balancer

```hcl
# External IP
resource "google_compute_global_address" "lb" {
  name    = "lb-external-ip"
  project = var.project_id
}

# SSL Certificate (managed)
resource "google_compute_managed_ssl_certificate" "lb" {
  name    = "lb-ssl-cert"
  project = var.project_id
  
  managed {
    domains = [var.domain_name]
  }
}

# Health Check
resource "google_compute_health_check" "http" {
  name    = "http-health-check"
  project = var.project_id
  
  timeout_sec         = 5
  check_interval_sec  = 10
  healthy_threshold   = 2
  unhealthy_threshold = 3
  
  http_health_check {
    port         = 80
    request_path = "/health"
  }
}

# Backend Service
resource "google_compute_backend_service" "web" {
  name        = "web-backend"
  project     = var.project_id
  protocol    = "HTTP"
  port_name   = "http"
  timeout_sec = 30
  
  health_checks = [google_compute_health_check.http.id]
  
  backend {
    group           = google_compute_instance_group_manager.web.instance_group
    balancing_mode  = "UTILIZATION"
    max_utilization = 0.8
    capacity_scaler = 1.0
  }
  
  cdn_policy {
    cache_mode = "CACHE_ALL_STATIC"
    default_ttl = 3600
    max_ttl     = 86400
  }
  
  log_config {
    enable      = true
    sample_rate = 1.0
  }
}

# URL Map
resource "google_compute_url_map" "main" {
  name            = "main-url-map"
  project         = var.project_id
  default_service = google_compute_backend_service.web.id
  
  host_rule {
    hosts        = [var.domain_name]
    path_matcher = "main"
  }
  
  path_matcher {
    name            = "main"
    default_service = google_compute_backend_service.web.id
    
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
resource "google_compute_target_https_proxy" "main" {
  name    = "main-https-proxy"
  project = var.project_id
  url_map = google_compute_url_map.main.id
  
  ssl_certificates = [google_compute_managed_ssl_certificate.lb.id]
}

# Global Forwarding Rule
resource "google_compute_global_forwarding_rule" "https" {
  name        = "https-forwarding-rule"
  project     = var.project_id
  target      = google_compute_target_https_proxy.main.id
  port_range  = "443"
  ip_address  = google_compute_global_address.lb.address
  ip_protocol = "TCP"
}

# HTTP to HTTPS redirect
resource "google_compute_url_map" "http_redirect" {
  name    = "http-redirect"
  project = var.project_id
  
  default_url_redirect {
    https_redirect         = true
    strip_query            = false
    redirect_response_code = "MOVED_PERMANENTLY_DEFAULT"
  }
}

resource "google_compute_target_http_proxy" "redirect" {
  name    = "http-redirect-proxy"
  project = var.project_id
  url_map = google_compute_url_map.http_redirect.id
}

resource "google_compute_global_forwarding_rule" "http" {
  name        = "http-forwarding-rule"
  project     = var.project_id
  target      = google_compute_target_http_proxy.redirect.id
  port_range  = "80"
  ip_address  = google_compute_global_address.lb.address
  ip_protocol = "TCP"
}
```

### Terraform: Regional Network Load Balancer

```hcl
# Health Check
resource "google_compute_region_health_check" "tcp" {
  name    = "tcp-health-check"
  project = var.project_id
  region  = var.region
  
  timeout_sec         = 5
  check_interval_sec  = 5
  healthy_threshold   = 2
  unhealthy_threshold = 2
  
  tcp_health_check {
    port = 80
  }
}

# Regional Backend Service
resource "google_compute_region_backend_service" "app" {
  name          = "app-backend"
  project       = var.project_id
  region        = var.region
  protocol      = "TCP"
  load_balancing_scheme = "EXTERNAL"
  
  health_checks = [google_compute_region_health_check.tcp.id]
  
  backend {
    group          = google_compute_instance_group_manager.app.instance_group
    balancing_mode = "CONNECTION"
  }
}

# Forwarding Rule
resource "google_compute_forwarding_rule" "app" {
  name                  = "app-forwarding-rule"
  project               = var.project_id
  region                = var.region
  ip_protocol           = "TCP"
  load_balancing_scheme = "EXTERNAL"
  port_range            = "80"
  backend_service       = google_compute_region_backend_service.app.id
}
```

---

## Compute Engine

### Terraform: Instance Template e MIG

```hcl
# Service Account
resource "google_service_account" "vm" {
  account_id   = "vm-service-account"
  display_name = "VM Service Account"
  project      = var.project_id
}

# Instance Template
resource "google_compute_instance_template" "web" {
  name_prefix  = "web-template-"
  project      = var.project_id
  machine_type = "e2-medium"
  region       = var.region
  
  tags = ["web", "allow-health-check", "allow-ssh"]
  
  disk {
    source_image = "debian-cloud/debian-11"
    auto_delete  = true
    boot         = true
    disk_size_gb = 20
    disk_type    = "pd-ssd"
  }
  
  network_interface {
    subnetwork = google_compute_subnetwork.web.id
    
    # Sem IP externo (usa NAT)
  }
  
  service_account {
    email  = google_service_account.vm.email
    scopes = ["cloud-platform"]
  }
  
  metadata_startup_script = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
    systemctl enable nginx
    echo "<h1>Instance: $(hostname)</h1>" > /var/www/html/index.html
  EOF
  
  shielded_instance_config {
    enable_secure_boot          = true
    enable_vtpm                 = true
    enable_integrity_monitoring = true
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

# Managed Instance Group
resource "google_compute_region_instance_group_manager" "web" {
  name               = "web-mig"
  project            = var.project_id
  region             = var.region
  base_instance_name = "web"
  
  version {
    instance_template = google_compute_instance_template.web.id
  }
  
  target_size = 2
  
  named_port {
    name = "http"
    port = 80
  }
  
  auto_healing_policies {
    health_check      = google_compute_health_check.http.id
    initial_delay_sec = 300
  }
  
  update_policy {
    type                  = "PROACTIVE"
    minimal_action        = "REPLACE"
    max_surge_fixed       = 3
    max_unavailable_fixed = 0
    replacement_method    = "SUBSTITUTE"
  }
}

# Autoscaler
resource "google_compute_region_autoscaler" "web" {
  name    = "web-autoscaler"
  project = var.project_id
  region  = var.region
  target  = google_compute_region_instance_group_manager.web.id
  
  autoscaling_policy {
    max_replicas    = 10
    min_replicas    = 2
    cooldown_period = 60
    
    cpu_utilization {
      target = 0.7
    }
  }
}
```

---

## Google Kubernetes Engine (GKE)

### Terraform: GKE Cluster

```hcl
# GKE Cluster
resource "google_container_cluster" "main" {
  name     = "main-cluster"
  project  = var.project_id
  location = var.region
  
  # Usar node pools separados
  remove_default_node_pool = true
  initial_node_count       = 1
  
  # Network
  network    = google_compute_network.main.id
  subnetwork = google_compute_subnetwork.gke.id
  
  ip_allocation_policy {
    cluster_secondary_range_name  = "gke-pods"
    services_secondary_range_name = "gke-services"
  }
  
  # Private cluster
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }
  
  # Master authorized networks
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = var.admin_cidr
      display_name = "Admin"
    }
  }
  
  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
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
  }
  
  # Security
  network_policy {
    enabled  = true
    provider = "CALICO"
  }
  
  binary_authorization {
    evaluation_mode = "PROJECT_SINGLETON_POLICY_ENFORCE"
  }
  
  # Logging and Monitoring
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }
  
  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS"]
    
    managed_prometheus {
      enabled = true
    }
  }
  
  # Maintenance
  maintenance_policy {
    recurring_window {
      start_time = "2024-01-01T09:00:00Z"
      end_time   = "2024-01-01T17:00:00Z"
      recurrence = "FREQ=WEEKLY;BYDAY=SA,SU"
    }
  }
  
  release_channel {
    channel = "REGULAR"
  }
}

# System Node Pool
resource "google_container_node_pool" "system" {
  name       = "system-pool"
  project    = var.project_id
  location   = var.region
  cluster    = google_container_cluster.main.name
  
  node_count = 1
  
  autoscaling {
    min_node_count = 1
    max_node_count = 3
  }
  
  node_config {
    machine_type = "e2-medium"
    disk_size_gb = 50
    disk_type    = "pd-ssd"
    
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
    
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
    
    labels = {
      "nodepool-type" = "system"
    }
    
    taint {
      key    = "CriticalAddonsOnly"
      value  = "true"
      effect = "PREFER_NO_SCHEDULE"
    }
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
}

# Application Node Pool
resource "google_container_node_pool" "app" {
  name     = "app-pool"
  project  = var.project_id
  location = var.region
  cluster  = google_container_cluster.main.name
  
  autoscaling {
    min_node_count = 2
    max_node_count = 10
  }
  
  node_config {
    machine_type = "e2-standard-4"
    disk_size_gb = 100
    disk_type    = "pd-ssd"
    
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
    
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
    
    labels = {
      "nodepool-type" = "app"
    }
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
}
```

---

## Cloud Storage

### Terraform: Bucket Seguro

```hcl
# Cloud Storage Bucket
resource "google_storage_bucket" "data" {
  name          = "${var.project_id}-data"
  project       = var.project_id
  location      = var.region
  storage_class = "STANDARD"
  
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
  
  versioning {
    enabled = true
  }
  
  encryption {
    default_kms_key_name = google_kms_crypto_key.bucket.id
  }
  
  lifecycle_rule {
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
    condition {
      age = 30
    }
  }
  
  lifecycle_rule {
    action {
      type          = "SetStorageClass"
      storage_class = "COLDLINE"
    }
    condition {
      age = 90
    }
  }
  
  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      age = 365
    }
  }
  
  logging {
    log_bucket        = google_storage_bucket.logs.name
    log_object_prefix = "data-bucket/"
  }
  
  labels = {
    environment = var.environment
    managed_by  = "terraform"
  }
}

# Logs Bucket
resource "google_storage_bucket" "logs" {
  name          = "${var.project_id}-logs"
  project       = var.project_id
  location      = var.region
  storage_class = "STANDARD"
  
  uniform_bucket_level_access = true
  public_access_prevention    = "enforced"
  
  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      age = 90
    }
  }
}

# KMS Key Ring
resource "google_kms_key_ring" "main" {
  name     = "main-keyring"
  project  = var.project_id
  location = var.region
}

# KMS Key para Storage
resource "google_kms_crypto_key" "bucket" {
  name     = "bucket-key"
  key_ring = google_kms_key_ring.main.id
  
  rotation_period = "7776000s"  # 90 days
  
  lifecycle {
    prevent_destroy = true
  }
}
```

---

## IAM e Segurança

### Boas Práticas IAM

| Prática | Descrição |
|---------|-----------|
| ✅ Princípio do menor privilégio | Conceda apenas permissões necessárias |
| ✅ Use Service Accounts | Para aplicações e serviços |
| ✅ Workload Identity | Para workloads em GKE |
| ✅ IAM Conditions | Para acesso contextual |
| ✅ Organization Policies | Para governança |
| ❌ Evite roles básicas | Use roles predefinidas ou custom |
| ❌ Não use chaves de SA | Prefira Workload Identity |

### Terraform: IAM Configuration

```hcl
# Service Account para aplicação
resource "google_service_account" "app" {
  account_id   = "app-service-account"
  display_name = "Application Service Account"
  project      = var.project_id
}

# Custom Role
resource "google_project_iam_custom_role" "app_role" {
  role_id     = "appCustomRole"
  title       = "App Custom Role"
  description = "Custom role for application"
  project     = var.project_id
  
  permissions = [
    "storage.objects.get",
    "storage.objects.list",
    "storage.objects.create",
    "pubsub.topics.publish",
    "secretmanager.versions.access"
  ]
}

# IAM Binding
resource "google_project_iam_binding" "app" {
  project = var.project_id
  role    = google_project_iam_custom_role.app_role.id
  
  members = [
    "serviceAccount:${google_service_account.app.email}"
  ]
  
  condition {
    title       = "Only in business hours"
    description = "Access only during business hours"
    expression  = "request.time.getHours('America/Sao_Paulo') >= 9 && request.time.getHours('America/Sao_Paulo') <= 18"
  }
}

# Workload Identity Binding
resource "google_service_account_iam_binding" "workload_identity" {
  service_account_id = google_service_account.app.name
  role               = "roles/iam.workloadIdentityUser"
  
  members = [
    "serviceAccount:${var.project_id}.svc.id.goog[${var.k8s_namespace}/${var.k8s_service_account}]"
  ]
}
```

### Organization Policies

```hcl
# Require OS Login
resource "google_organization_policy" "os_login" {
  org_id     = var.org_id
  constraint = "compute.requireOsLogin"
  
  boolean_policy {
    enforced = true
  }
}

# Restrict public IPs
resource "google_organization_policy" "vm_external_ip" {
  org_id     = var.org_id
  constraint = "compute.vmExternalIpAccess"
  
  list_policy {
    deny {
      all = true
    }
  }
}

# Require Uniform Bucket Access
resource "google_organization_policy" "uniform_bucket" {
  org_id     = var.org_id
  constraint = "storage.uniformBucketLevelAccess"
  
  boolean_policy {
    enforced = true
  }
}
```

---

## Terraform para GCP

### Estrutura de Projeto

```
gcp-infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       └── ...
├── modules/
│   ├── networking/
│   ├── gke/
│   ├── compute/
│   └── storage/
├── backend.tf
└── provider.tf
```

### Provider Configuration

```hcl
terraform {
  required_version = ">= 1.0.0"
  
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
  }
  
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "terraform/state"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

provider "google-beta" {
  project = var.project_id
  region  = var.region
}
```

---

## Próximos Passos

- 📘 [Introdução ao IaC](../iac/introduction.md)
- 📖 [Melhores Práticas](../iac/best-practices.md)
- ☁️ [Guias AWS](./aws-guides.md)
- 🔷 [Azure Bicep Examples](./azure-bicep-examples.md)
- 🔒 [Conformidade ISO 27001](../compliance/iso-27001.md)