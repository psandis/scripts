# Terraform Scripts & Examples

Infrastructure as Code for AWS and GCP. Covers VPCs, compute instances, databases, DNS, load balancers, and more.

---

## Table of Contents

- [Terraform Basics](#terraform-basics)
- [AWS Full Web Stack](#aws-full-web-stack)
- [GCP Full Web Stack](#gcp-full-web-stack)
- [Shared Patterns](#shared-patterns)
- [State Management](#state-management)
- [Tips](#tips)

---

## Terraform Basics

### Install Terraform

```bash
# macOS
brew install terraform

# Linux
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

---

### Core Commands

```bash
terraform init          # Initialize working directory, download providers
terraform plan          # Preview changes
terraform apply         # Apply changes
terraform destroy       # Tear down everything
terraform fmt           # Format .tf files
terraform validate      # Validate configuration
terraform output        # Show output values
terraform state list    # List resources in state
terraform state show <resource>  # Show details of a resource
```

---

### Project Structure

```
project/
├── main.tf             # Primary resources
├── variables.tf        # Input variables
├── outputs.tf          # Output values
├── providers.tf        # Provider configuration
├── terraform.tfvars    # Variable values (DO NOT commit secrets)
└── modules/
    ├── networking/
    ├── compute/
    └── database/
```

---

## AWS Full Web Stack

A complete example: VPC → EC2 instance with Docker → RDS PostgreSQL → Route53 DNS → ALB with SSL.

### providers.tf

```hcl
terraform {
  required_version = ">= 1.5"
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

### variables.tf

```hcl
variable "aws_region" {
  default = "eu-west-1"
}

variable "project_name" {
  default = "myapp"
}

variable "domain_name" {
  description = "Your domain (e.g. example.com)"
  type        = string
}

variable "db_username" {
  description = "Database master username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}

variable "instance_type" {
  default = "t3.small"
}

variable "ssh_public_key" {
  description = "Path to SSH public key"
  default     = "~/.ssh/id_ed25519.pub"
}
```

### VPC & Networking

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = { Name = "${var.project_name}-vpc" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.project_name}-igw" }
}

resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true
  tags                    = { Name = "${var.project_name}-public-a" }
}

resource "aws_subnet" "public_b" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "${var.aws_region}b"
  map_public_ip_on_launch = true
  tags                    = { Name = "${var.project_name}-public-b" }
}

resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "${var.aws_region}a"
  tags              = { Name = "${var.project_name}-private-a" }
}

resource "aws_subnet" "private_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "${var.aws_region}b"
  tags              = { Name = "${var.project_name}-private-b" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${var.project_name}-public-rt" }
}

resource "aws_route_table_association" "public_a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}
```

### Security Groups

```hcl
resource "aws_security_group" "web" {
  name   = "${var.project_name}-web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

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

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project_name}-web-sg" }
}

resource "aws_security_group" "db" {
  name   = "${var.project_name}-db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "PostgreSQL from web servers"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project_name}-db-sg" }
}
```

### EC2 Instance (Docker Host)

```hcl
resource "aws_key_pair" "deployer" {
  key_name   = "${var.project_name}-key"
  public_key = file(var.ssh_public_key)
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  key_name               = aws_key_pair.deployer.key_name
  subnet_id              = aws_subnet.public_a.id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    set -e
    apt-get update
    apt-get install -y ca-certificates curl gnupg
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | tee /etc/apt/sources.list.d/docker.list
    apt-get update
    apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    systemctl enable docker
    usermod -aG docker ubuntu
  EOF

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
  }

  tags = { Name = "${var.project_name}-web" }
}

resource "aws_eip" "web" {
  instance = aws_instance.web.id
  domain   = "vpc"
  tags     = { Name = "${var.project_name}-eip" }
}
```

### RDS PostgreSQL

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "${var.project_name}-db-subnet"
  subnet_ids = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  tags       = { Name = "${var.project_name}-db-subnet" }
}

resource "aws_db_instance" "postgres" {
  identifier             = "${var.project_name}-db"
  engine                 = "postgres"
  engine_version         = "15"
  instance_class         = "db.t3.micro"
  allocated_storage      = 20
  storage_type           = "gp3"
  db_name                = "appdb"
  username               = var.db_username
  password               = var.db_password
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
  skip_final_snapshot    = true
  publicly_accessible    = false

  tags = { Name = "${var.project_name}-db" }
}
```

### Route53 DNS

```hcl
data "aws_route53_zone" "main" {
  name = var.domain_name
}

resource "aws_route53_record" "web" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"
  ttl     = 300
  records = [aws_eip.web.public_ip]
}

resource "aws_route53_record" "www" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "www.${var.domain_name}"
  type    = "CNAME"
  ttl     = 300
  records = [var.domain_name]
}
```

### ACM Certificate (for ALB)

```hcl
resource "aws_acm_certificate" "main" {
  domain_name               = var.domain_name
  subject_alternative_names = ["*.${var.domain_name}"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }

  tags = { Name = "${var.project_name}-cert" }
}

resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  ttl     = 60
  records = [each.value.record]
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}
```

### outputs.tf

```hcl
output "web_public_ip" {
  value = aws_eip.web.public_ip
}

output "db_endpoint" {
  value = aws_db_instance.postgres.endpoint
}

output "domain" {
  value = var.domain_name
}

output "ssh_command" {
  value = "ssh ubuntu@${aws_eip.web.public_ip}"
}
```

---

## GCP Full Web Stack

Equivalent stack on Google Cloud: VPC → GCE instance with Docker → Cloud SQL PostgreSQL → Cloud DNS.

### providers.tf

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
  zone    = var.gcp_zone
}
```

### variables.tf

```hcl
variable "gcp_project_id" {
  description = "GCP project ID"
  type        = string
}

variable "gcp_region" {
  default = "europe-west1"
}

variable "gcp_zone" {
  default = "europe-west1-b"
}

variable "project_name" {
  default = "myapp"
}

variable "domain_name" {
  description = "Your domain (e.g. example.com)"
  type        = string
}

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

variable "machine_type" {
  default = "e2-small"
}

variable "ssh_public_key" {
  description = "Path to SSH public key"
  default     = "~/.ssh/id_ed25519.pub"
}

variable "ssh_user" {
  default = "deploy"
}
```

### VPC & Networking

```hcl
resource "google_compute_network" "main" {
  name                    = "${var.project_name}-vpc"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "public" {
  name          = "${var.project_name}-public"
  ip_cidr_range = "10.0.1.0/24"
  region        = var.gcp_region
  network       = google_compute_network.main.id
}

resource "google_compute_subnetwork" "private" {
  name          = "${var.project_name}-private"
  ip_cidr_range = "10.0.10.0/24"
  region        = var.gcp_region
  network       = google_compute_network.main.id
  purpose       = "PRIVATE"
}
```

### Firewall Rules

```hcl
resource "google_compute_firewall" "allow_ssh" {
  name    = "${var.project_name}-allow-ssh"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["web"]
}

resource "google_compute_firewall" "allow_http_https" {
  name    = "${var.project_name}-allow-http-https"
  network = google_compute_network.main.name

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["web"]
}
```

### GCE Instance (Docker Host)

```hcl
resource "google_compute_address" "web" {
  name   = "${var.project_name}-ip"
  region = var.gcp_region
}

resource "google_compute_instance" "web" {
  name         = "${var.project_name}-web"
  machine_type = var.machine_type
  zone         = var.gcp_zone
  tags         = ["web"]

  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts"
      size  = 20
      type  = "pd-ssd"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.public.id
    access_config {
      nat_ip = google_compute_address.web.address
    }
  }

  metadata = {
    ssh-keys = "${var.ssh_user}:${file(var.ssh_public_key)}"
  }

  metadata_startup_script = <<-EOF
    #!/bin/bash
    set -e
    apt-get update
    apt-get install -y ca-certificates curl gnupg
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | tee /etc/apt/sources.list.d/docker.list
    apt-get update
    apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
    systemctl enable docker
    usermod -aG docker ${var.ssh_user}
  EOF
}
```

### Cloud SQL PostgreSQL

```hcl
resource "google_sql_database_instance" "postgres" {
  name             = "${var.project_name}-db"
  database_version = "POSTGRES_15"
  region           = var.gcp_region

  settings {
    tier = "db-f1-micro"

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.main.id
    }

    backup_configuration {
      enabled = true
    }
  }

  deletion_protection = false
}

resource "google_sql_database" "appdb" {
  name     = "appdb"
  instance = google_sql_database_instance.postgres.name
}

resource "google_sql_user" "app" {
  name     = "appuser"
  instance = google_sql_database_instance.postgres.name
  password = var.db_password
}
```

### Cloud DNS

```hcl
resource "google_dns_managed_zone" "main" {
  name     = "${var.project_name}-zone"
  dns_name = "${var.domain_name}."
}

resource "google_dns_record_set" "web" {
  name         = "${var.domain_name}."
  type         = "A"
  ttl          = 300
  managed_zone = google_dns_managed_zone.main.name
  rrdatas      = [google_compute_address.web.address]
}

resource "google_dns_record_set" "www" {
  name         = "www.${var.domain_name}."
  type         = "CNAME"
  ttl          = 300
  managed_zone = google_dns_managed_zone.main.name
  rrdatas      = ["${var.domain_name}."]
}
```

### outputs.tf

```hcl
output "web_public_ip" {
  value = google_compute_address.web.address
}

output "db_connection" {
  value = google_sql_database_instance.postgres.connection_name
}

output "domain" {
  value = var.domain_name
}

output "ssh_command" {
  value = "ssh ${var.ssh_user}@${google_compute_address.web.address}"
}

output "dns_nameservers" {
  value = google_dns_managed_zone.main.name_servers
}
```

---

## Shared Patterns

### terraform.tfvars example

```hcl
# AWS
aws_region   = "eu-west-1"
project_name = "myapp"
domain_name  = "example.com"
db_username  = "appuser"
db_password  = "change-me-to-something-secure"

# GCP
gcp_project_id = "my-project-123456"
gcp_region     = "europe-west1"
gcp_zone       = "europe-west1-b"
```

> Never commit `terraform.tfvars` with real secrets. Use environment variables or a secrets manager instead.

---

### .gitignore for Terraform

```
*.tfstate
*.tfstate.*
*.tfvars
.terraform/
.terraform.lock.hcl
crash.log
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```

---

## State Management

### Remote State with S3 (AWS)

```hcl
terraform {
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Remote State with GCS (GCP)

```hcl
terraform {
  backend "gcs" {
    bucket = "myapp-terraform-state"
    prefix = "prod"
  }
}
```

---

## Tips

- Always run `terraform plan` before `apply`
- Use workspaces or separate directories for dev/staging/prod
- Tag everything for easier cleanup and cost tracking
- Use `terraform import` to bring existing resources under management
- Store state remotely with locking to avoid conflicts
- Use `-target=resource` to apply a single resource during debugging
- Set `deletion_protection = true` on databases in production
