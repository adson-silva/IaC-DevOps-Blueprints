# AWS Guides

Guias e exemplos práticos para configurar serviços no AWS como S3, EC2 e IAM.

---

## Índice

1. [Visão Geral da AWS](#visão-geral-da-aws)
2. [IAM - Identity and Access Management](#iam---identity-and-access-management)
3. [EC2 - Elastic Compute Cloud](#ec2---elastic-compute-cloud)
4. [S3 - Simple Storage Service](#s3---simple-storage-service)
5. [VPC - Virtual Private Cloud](#vpc---virtual-private-cloud)
6. [RDS - Relational Database Service](#rds---relational-database-service)
7. [Lambda - Serverless Computing](#lambda---serverless-computing)
8. [Segurança e Compliance](#segurança-e-compliance)

---

## Visão Geral da AWS

### Regiões e Zonas de Disponibilidade

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS Global                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────┐   ┌─────────────────────┐        │
│   │ Region: us-east-1   │   │ Region: sa-east-1   │        │
│   │ (N. Virginia)       │   │ (São Paulo)         │        │
│   │                     │   │                     │        │
│   │ ┌────┐ ┌────┐ ┌────┐│   │ ┌────┐ ┌────┐ ┌────┐│        │
│   │ │AZ-a│ │AZ-b│ │AZ-c││   │ │AZ-a│ │AZ-b│ │AZ-c││        │
│   │ └────┘ └────┘ └────┘│   │ └────┘ └────┘ └────┘│        │
│   └─────────────────────┘   └─────────────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Serviços Mais Utilizados

| Categoria | Serviço | Descrição |
|-----------|---------|-----------|
| **Compute** | EC2 | Máquinas virtuais |
| **Compute** | Lambda | Serverless |
| **Storage** | S3 | Object storage |
| **Storage** | EBS | Block storage |
| **Database** | RDS | Banco relacional |
| **Database** | DynamoDB | NoSQL |
| **Network** | VPC | Rede virtual |
| **Security** | IAM | Identidade |

---

## IAM - Identity and Access Management

### Conceitos Principais

```
┌─────────────────────────────────────────────────────────────┐
│                          IAM                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐  associa  ┌──────────┐  anexada  ┌─────────┐ │
│  │  Users   │──────────▶│  Groups  │──────────▶│ Policies│ │
│  └──────────┘           └──────────┘           └─────────┘ │
│                                                     │       │
│  ┌──────────┐                                       │       │
│  │  Roles   │◀──────────────────────────────────────┘       │
│  └──────────┘                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Terraform: Criar Usuário IAM

```hcl
# Usuário IAM
resource "aws_iam_user" "developer" {
  name = "developer-user"
  path = "/developers/"
  
  tags = {
    Department = "Engineering"
    Role       = "Developer"
  }
}

# Credenciais de acesso programático
resource "aws_iam_access_key" "developer" {
  user = aws_iam_user.developer.name
}

# Grupo para desenvolvedores
resource "aws_iam_group" "developers" {
  name = "developers"
  path = "/developers/"
}

# Adicionar usuário ao grupo
resource "aws_iam_user_group_membership" "developer" {
  user = aws_iam_user.developer.name
  groups = [aws_iam_group.developers.name]
}

# Política inline para o grupo
resource "aws_iam_group_policy" "developers_policy" {
  name  = "developers-policy"
  group = aws_iam_group.developers.name
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:Describe*",
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = "*"
      }
    ]
  })
}
```

### Terraform: Criar Role IAM

```hcl
# Role para EC2
resource "aws_iam_role" "ec2_role" {
  name = "ec2-application-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
  
  tags = {
    Purpose = "EC2 Application Role"
  }
}

# Política para a role
resource "aws_iam_role_policy_attachment" "s3_read" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

# Instance Profile para EC2
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-application-profile"
  role = aws_iam_role.ec2_role.name
}
```

### Políticas AWS Gerenciadas vs Custom

| Tipo | Quando Usar |
|------|-------------|
| **AWS Managed** | Casos de uso comuns, manutenção automática |
| **Customer Managed** | Requisitos específicos, maior controle |
| **Inline** | Políticas únicas para um recurso específico |

### Boas Práticas IAM

1. ✅ Princípio do menor privilégio
2. ✅ Usar roles em vez de chaves de acesso
3. ✅ Habilitar MFA
4. ✅ Rotacionar credenciais regularmente
5. ✅ Auditar com CloudTrail
6. ❌ Nunca usar a conta root
7. ❌ Não compartilhar credenciais

---

## EC2 - Elastic Compute Cloud

### Tipos de Instância

| Família | Uso | Exemplo |
|---------|-----|---------|
| **t3** | Uso geral, burstable | t3.micro, t3.small |
| **m6i** | Uso geral, balanced | m6i.large, m6i.xlarge |
| **c6i** | Compute optimized | c6i.large, c6i.2xlarge |
| **r6i** | Memory optimized | r6i.large, r6i.xlarge |
| **g4dn** | GPU | g4dn.xlarge |

### Terraform: Instância EC2 Básica

```hcl
# Data source para AMI mais recente
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web server"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "web-sg"
  }
}

# Key Pair
resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("~/.ssh/id_rsa.pub")
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  key_name               = aws_key_pair.deployer.key_name
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public[0].id
  iam_instance_profile   = aws_iam_instance_profile.ec2_profile.name
  
  root_block_device {
    volume_type           = "gp3"
    volume_size           = 20
    delete_on_termination = true
    encrypted             = true
  }
  
  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
  EOF
  
  tags = {
    Name        = "web-server"
    Environment = var.environment
  }
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### Terraform: Auto Scaling Group

```hcl
# Launch Template
resource "aws_launch_template" "web" {
  name_prefix   = "web-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  
  vpc_security_group_ids = [aws_security_group.web.id]
  
  iam_instance_profile {
    name = aws_iam_instance_profile.ec2_profile.name
  }
  
  block_device_mappings {
    device_name = "/dev/xvda"
    
    ebs {
      volume_size           = 20
      volume_type           = "gp3"
      delete_on_termination = true
      encrypted             = true
    }
  }
  
  user_data = base64encode(<<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
    echo "<h1>Instance: $INSTANCE_ID</h1>" > /var/www/html/index.html
  EOF
  )
  
  tag_specifications {
    resource_type = "instance"
    
    tags = {
      Name = "web-asg-instance"
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  vpc_zone_identifier = aws_subnet.public[*].id
  target_group_arns   = [aws_lb_target_group.web.arn]
  health_check_type   = "ELB"
  
  min_size         = 2
  max_size         = 4
  desired_capacity = 2
  
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
  
  tag {
    key                 = "Environment"
    value               = var.environment
    propagate_at_launch = true
  }
}

# Scaling Policy
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "scale-up"
  autoscaling_group_name = aws_autoscaling_group.web.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = 1
  cooldown               = 300
}

resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80
  
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.web.name
  }
  
  alarm_actions = [aws_autoscaling_policy.scale_up.arn]
}
```

---

## S3 - Simple Storage Service

### Classes de Armazenamento

| Classe | Uso | Custo |
|--------|-----|-------|
| **Standard** | Acesso frequente | $$ |
| **Standard-IA** | Acesso infrequente | $ |
| **One Zone-IA** | Dados reproduzíveis | $ |
| **Glacier Instant** | Arquivamento rápido | ¢ |
| **Glacier Deep** | Arquivamento longo prazo | ¢ |
| **Intelligent-Tiering** | Padrões variáveis | $$ |

### Terraform: Bucket S3 Completo

```hcl
# Bucket principal
resource "aws_s3_bucket" "data" {
  bucket = "minha-empresa-data-${var.environment}"
  
  tags = {
    Name        = "Data Bucket"
    Environment = var.environment
  }
}

# Configuração de acesso público
resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Versionamento
resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Criptografia
resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

# Lifecycle
resource "aws_s3_bucket_lifecycle_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  
  rule {
    id     = "archive-old-data"
    status = "Enabled"
    
    filter {
      prefix = "data/"
    }
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
    
    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
  
  rule {
    id     = "delete-temp"
    status = "Enabled"
    
    filter {
      prefix = "temp/"
    }
    
    expiration {
      days = 7
    }
  }
}
```

> 📖 Veja o exemplo completo em [Terraform S3 Setup](../iac/examples/terraform-s3-setup.md)

---

## VPC - Virtual Private Cloud

### Arquitetura de VPC

```
┌─────────────────────────────────────────────────────────────┐
│                      VPC (10.0.0.0/16)                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────┐ ┌─────────────────────────┐   │
│  │  Public Subnet AZ-a     │ │  Public Subnet AZ-b     │   │
│  │  10.0.1.0/24            │ │  10.0.2.0/24            │   │
│  │  ┌───────┐ ┌───────┐    │ │  ┌───────┐ ┌───────┐    │   │
│  │  │  EC2  │ │  NAT  │    │ │  │  EC2  │ │  NAT  │    │   │
│  │  └───────┘ └───────┘    │ │  └───────┘ └───────┘    │   │
│  └────────────┬────────────┘ └────────────┬────────────┘   │
│               │                           │                 │
│               ▼                           ▼                 │
│           ┌───────────────────────────────────┐            │
│           │      Internet Gateway             │            │
│           └───────────────────────────────────┘            │
│                                                             │
│  ┌─────────────────────────┐ ┌─────────────────────────┐   │
│  │  Private Subnet AZ-a    │ │  Private Subnet AZ-b    │   │
│  │  10.0.11.0/24           │ │  10.0.12.0/24           │   │
│  │  ┌───────┐ ┌───────┐    │ │  ┌───────┐ ┌───────┐    │   │
│  │  │  EC2  │ │  RDS  │    │ │  │  EC2  │ │  RDS  │    │   │
│  │  └───────┘ └───────┘    │ │  └───────┘ └───────┘    │   │
│  └─────────────────────────┘ └─────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Terraform: VPC Completa

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "main-vpc"
  }
}

# Subnets Públicas
resource "aws_subnet" "public" {
  count = 2
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
    Tier = "Public"
  }
}

# Subnets Privadas
resource "aws_subnet" "private" {
  count = 2
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 11}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "private-subnet-${count.index + 1}"
    Tier = "Private"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "main-igw"
  }
}

# Elastic IP para NAT Gateway
resource "aws_eip" "nat" {
  count  = 2
  domain = "vpc"
  
  tags = {
    Name = "nat-eip-${count.index + 1}"
  }
}

# NAT Gateway
resource "aws_nat_gateway" "main" {
  count = 2
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = {
    Name = "nat-gw-${count.index + 1}"
  }
  
  depends_on = [aws_internet_gateway.main]
}

# Route Table Pública
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "public-rt"
  }
}

# Route Table Privada
resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  
  tags = {
    Name = "private-rt-${count.index + 1}"
  }
}

# Associações de Route Table
resource "aws_route_table_association" "public" {
  count = 2
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = 2
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Data source para AZs
data "aws_availability_zones" "available" {
  state = "available"
}
```

---

## RDS - Relational Database Service

### Terraform: RDS MySQL

```hcl
# Subnet Group para RDS
resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id
  
  tags = {
    Name = "Main DB Subnet Group"
  }
}

# Security Group para RDS
resource "aws_security_group" "rds" {
  name        = "rds-sg"
  description = "Security group for RDS"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description     = "MySQL from app servers"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }
  
  tags = {
    Name = "rds-sg"
  }
}

# RDS Instance
resource "aws_db_instance" "main" {
  identifier = "main-database"
  
  # Engine
  engine               = "mysql"
  engine_version       = "8.0"
  instance_class       = "db.t3.micro"
  
  # Storage
  allocated_storage     = 20
  max_allocated_storage = 100
  storage_type          = "gp3"
  storage_encrypted     = true
  
  # Network
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false
  
  # Database
  db_name  = "appdb"
  username = "admin"
  password = var.db_password # Use Secrets Manager
  
  # Backup
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"
  
  # Monitoring
  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn
  
  # Other
  multi_az               = true
  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "main-database-final"
  
  tags = {
    Name        = "main-database"
    Environment = var.environment
  }
}
```

---

## Lambda - Serverless Computing

### Terraform: Lambda Function

```hcl
# IAM Role para Lambda
resource "aws_iam_role" "lambda" {
  name = "lambda-execution-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

# Lambda Function
resource "aws_lambda_function" "processor" {
  filename         = "lambda.zip"
  function_name    = "data-processor"
  role             = aws_iam_role.lambda.arn
  handler          = "index.handler"
  source_code_hash = filebase64sha256("lambda.zip")
  runtime          = "nodejs18.x"
  timeout          = 30
  memory_size      = 256
  
  environment {
    variables = {
      BUCKET_NAME = aws_s3_bucket.data.id
      TABLE_NAME  = aws_dynamodb_table.main.name
    }
  }
  
  vpc_config {
    subnet_ids         = aws_subnet.private[*].id
    security_group_ids = [aws_security_group.lambda.id]
  }
  
  tags = {
    Name = "data-processor"
  }
}

# Trigger S3
resource "aws_lambda_permission" "s3" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.data.arn
}
```

---

## Segurança e Compliance

### Checklist de Segurança AWS

- [ ] MFA habilitado em todas as contas
- [ ] CloudTrail ativo em todas as regiões
- [ ] GuardDuty ativo
- [ ] Config Rules configuradas
- [ ] Security Hub habilitado
- [ ] VPC Flow Logs ativos
- [ ] S3 com bloqueio de acesso público
- [ ] Criptografia em repouso para todos os dados
- [ ] Secrets Manager para credenciais
- [ ] WAF para aplicações web

### Terraform: Security Hub

```hcl
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/cis-aws-foundations-benchmark/v/1.4.0"
  
  depends_on = [aws_securityhub_account.main]
}

resource "aws_securityhub_standards_subscription" "aws_foundational" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/aws-foundational-security-best-practices/v/1.0.0"
  
  depends_on = [aws_securityhub_account.main]
}
```

---

## Próximos Passos

- 📘 [Introdução ao IaC](../iac/introduction.md)
- 📖 [Melhores Práticas](../iac/best-practices.md)
- 🔒 [Conformidade ISO 27001](../compliance/iso-27001.md)