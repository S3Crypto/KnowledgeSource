---
title: "Introduction to Terraform for .NET Developers"
date_created: 2023-04-02
date_updated: 2023-04-02
authors: ["Repository Maintainers"]
tags: ["terraform", "iac", "infrastructure", "cloud", "devops"]
difficulty: "intermediate"
---

# Introduction to Terraform for .NET Developers

## Overview

Infrastructure as Code (IaC) is a practice of managing infrastructure through machine-readable definition files rather than manual processes. Terraform is a popular open-source IaC tool created by HashiCorp that allows developers to define and provision infrastructure using a declarative configuration language called HashiCorp Configuration Language (HCL).

This document introduces Terraform concepts, workflow, and best practices specifically relevant to .NET developers deploying applications to cloud platforms.

## Core Concepts

### Infrastructure as Code (IaC)

Infrastructure as Code treats infrastructure provisioning and management as a software engineering problem:

- Infrastructure is defined in code (configuration files)
- Configuration is version-controlled
- Changes follow software development practices (PR, review, testing)
- Infrastructure can be repeatedly and consistently deployed
- Changes are predictable and testable

### How Terraform Works

Terraform operates on these key principles:

1. **Declarative**: You define the desired state, not the steps to get there
2. **Resource Graph**: Terraform builds a dependency graph to determine order of operations
3. **Provider-Based**: Providers enable Terraform to work with various platforms (AWS, Azure, etc.)
4. **State Management**: Terraform tracks the real-world resource state in a state file

### Core Terraform Components

- **Configuration Files**: HCL files (`.tf`) that define your infrastructure
- **Providers**: Plugins that interface with cloud platforms, services, or APIs
- **Resources**: Infrastructure objects managed through providers
- **State File**: JSON file tracking real-world resource mappings
- **Modules**: Reusable, encapsulated units of Terraform configurations

## Terraform Workflow

The basic Terraform workflow consists of:

1. **Write** configuration files defining your infrastructure
2. **Plan** changes by generating an execution plan
3. **Apply** changes to create or modify infrastructure
4. **Destroy** infrastructure when no longer needed

```hcl
# Example workflow commands
terraform init      # Initialize working directory
terraform validate  # Validate configuration syntax
terraform plan      # Preview changes
terraform apply     # Apply changes
terraform destroy   # Destroy infrastructure
```

## Infrastructure for .NET Applications

### Typical .NET Application Infrastructure

A .NET application deployment typically requires:

- Compute resources (VMs, App Services, Kubernetes)
- Database services (SQL Server, CosmosDB)
- Storage solutions (Blob Storage, S3)
- Networking components (VNets, Subnets)
- Identity and access management
- Monitoring and logging services

### Azure Resources for .NET Applications

Below is a sample Terraform configuration for a basic .NET web application in Azure:

```hcl
# Configure the Azure provider
provider "azurerm" {
  features {}
}

# Create a resource group
resource "azurerm_resource_group" "example" {
  name     = "example-dotnet-app-rg"
  location = "West Europe"
}

# Create an App Service Plan
resource "azurerm_app_service_plan" "example" {
  name                = "example-app-service-plan"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  
  sku {
    tier = "Standard"
    size = "S1"
  }
}

# Create an App Service
resource "azurerm_app_service" "example" {
  name                = "example-dotnet-app"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  app_service_plan_id = azurerm_app_service_plan.example.id
  
  site_config {
    dotnet_framework_version = "v5.0"
    always_on                = true
  }
  
  app_settings = {
    "WEBSITE_NODE_DEFAULT_VERSION" = "10.14.1"
    "ASPNETCORE_ENVIRONMENT"       = "Production"
  }

  connection_string {
    name  = "Database"
    type  = "SQLAzure"
    value = "Server=tcp:${azurerm_sql_server.example.fully_qualified_domain_name},1433;Initial Catalog=${azurerm_sql_database.example.name};Persist Security Info=False;User ID=${var.sql_admin_username};Password=${var.sql_admin_password};MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
  }
}

# Create SQL Server
resource "azurerm_sql_server" "example" {
  name                         = "example-sqlserver"
  location                     = azurerm_resource_group.example.location
  resource_group_name          = azurerm_resource_group.example.name
  version                      = "12.0"
  administrator_login          = var.sql_admin_username
  administrator_login_password = var.sql_admin_password
}

# Create SQL Database
resource "azurerm_sql_database" "example" {
  name                = "example-db"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  server_name         = azurerm_sql_server.example.name
  
  edition = "Standard"
  
  tags = {
    environment = "production"
  }
}

# Variables
variable "sql_admin_username" {
  description = "SQL Server administrator username"
  type        = string
  sensitive   = true
}

variable "sql_admin_password" {
  description = "SQL Server administrator password"
  type        = string
  sensitive   = true
}

# Outputs
output "app_service_url" {
  value = "https://${azurerm_app_service.example.default_site_hostname}"
}

output "sql_server_fqdn" {
  value = azurerm_sql_server.example.fully_qualified_domain_name
}
```

### AWS Resources for .NET Applications

For AWS deployments, a similar .NET application might be configured like this:

```hcl
# Configure the AWS provider
provider "aws" {
  region = "us-west-2"
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "dotnet-app-vpc"
  }
}

# Create subnets
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = "us-west-2${count.index == 0 ? "a" : "b"}"
  
  tags = {
    Name = "dotnet-public-${count.index}"
  }
}

# Create Elastic Beanstalk application
resource "aws_elastic_beanstalk_application" "app" {
  name        = "dotnet-example-app"
  description = ".NET Core example application"
}

# Create Elastic Beanstalk environment
resource "aws_elastic_beanstalk_environment" "env" {
  name                = "dotnet-example-env"
  application         = aws_elastic_beanstalk_application.app.name
  solution_stack_name = "64bit Amazon Linux 2 v2.2.9 running .NET Core"
  
  setting {
    namespace = "aws:autoscaling:launchconfiguration"
    name      = "InstanceType"
    value     = "t3.micro"
  }
  
  setting {
    namespace = "aws:elasticbeanstalk:environment"
    name      = "EnvironmentType"
    value     = "LoadBalanced"
  }
  
  setting {
    namespace = "aws:elasticbeanstalk:application:environment"
    name      = "ASPNETCORE_ENVIRONMENT"
    value     = "Production"
  }
}

# Create RDS database
resource "aws_db_instance" "default" {
  allocated_storage    = 20
  storage_type         = "gp2"
  engine               = "sqlserver-ex"
  engine_version       = "14.00"
  instance_class       = "db.t3.small"
  name                 = "dotnetdb"
  username             = var.db_username
  password             = var.db_password
  parameter_group_name = "default.sqlserver-ex-14.0"
  skip_final_snapshot  = true
}

# Variables
variable "db_username" {
  description = "Database administrator username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Database administrator password"
  type        = string
  sensitive   = true
}

# Outputs
output "beanstalk_endpoint" {
  value = aws_elastic_beanstalk_environment.env.endpoint_url
}

output "database_endpoint" {
  value = aws_db_instance.default.endpoint
}
```

## Best Practices

### Code Organization

1. **Use Modules**: Encapsulate related resources into reusable modules
2. **Separate Variables**: Keep variables in separate files
3. **Environment Separation**: Use workspace or directory structures to separate environments
4. **Consistent Naming**: Follow consistent naming conventions

Example directory structure:

```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
├── modules/
│   ├── sql_database/
│   ├── app_service/
│   └── networking/
└── shared/
    └── provider.tf
```

### State Management

1. **Remote State Storage**: Store state in a secure, remote backend like Azure Storage or S3
2. **State Locking**: Enable locking to prevent concurrent operations
3. **State Isolation**: Use separate state files for different environments

```hcl
# Azure remote state configuration
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "terraformstate12345"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### Secret Management

1. **Never store secrets in code**: Use environment variables, Azure KeyVault, or AWS KMS
2. **Mark variables as sensitive**: Use the `sensitive = true` attribute
3. **Use external secret stores**: Integrate with tools like HashiCorp Vault

### CI/CD Integration

Integrate Terraform with your CI/CD pipeline:

1. **Validate and plan in CI**: Run `terraform validate` and `terraform plan` on pull requests
2. **Apply in CD**: Apply changes after approval, triggered by merges to main
3. **Use automated testing**: Test infrastructure with tools like Terratest

Example Azure DevOps pipeline snippet:

```yaml
- stage: TerraformPlan
  jobs:
  - job: Plan
    steps:
    - task: TerraformTaskV2@2
      inputs:
        command: 'init'
        workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure/environments/$(Environment)'
        backendType: 'azurerm'
        backendServiceArm: 'AzureServiceConnection'
        backendAzureRmResourceGroupName: 'terraform-state-rg'
        backendAzureRmStorageAccountName: 'terraformstate12345'
        backendAzureRmContainerName: 'tfstate'
        backendAzureRmKey: '$(Environment).terraform.tfstate'
    
    - task: TerraformTaskV2@2
      inputs:
        command: 'plan'
        workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure/environments/$(Environment)'
        environmentServiceName: 'AzureServiceConnection'
        publishPlanResults: 'TerraformPlan'
```

## Provisioning vs. Deployment

It's important to distinguish between:

- **Infrastructure Provisioning**: Creating and managing infrastructure resources (Terraform)
- **Application Deployment**: Deploying your .NET application code (Azure DevOps, GitHub Actions, etc.)

Best practice is to:

1. Use Terraform to provision infrastructure
2. Use separate deployment tools for application code
3. Coordinate both processes in your CI/CD pipeline

## Common Patterns for .NET Applications

### Microservices Infrastructure

For microservices architecture, consider:

- Separate App Services or Kubernetes clusters for each service
- Service-specific databases with appropriate isolation
- Shared resources for common functions (API Management, etc.)
- Network segmentation for service boundaries

### Multi-Environment Setup

Structure your Terraform code to support multiple environments:

```hcl
# Define environment-specific variables
locals {
  environments = {
    dev = {
      app_service_sku = {
        tier = "Standard"
        size = "S1"
      }
      sql_edition = "Basic"
    }
    prod = {
      app_service_sku = {
        tier = "Premium"
        size = "P1v2"
      }
      sql_edition = "Standard"
    }
  }
  
  # Select current environment based on workspace
  env = local.environments[terraform.workspace]
}

# Use the environment-specific values
resource "azurerm_app_service_plan" "example" {
  # ...
  sku {
    tier = local.env.app_service_sku.tier
    size = local.env.app_service_sku.size
  }
}
```

### Database Migration Strategy

Plan for database migrations:

1. Use Terraform for schema creation (initial setup only)
2. Use specialized database migration tools (Entity Framework migrations, Flyway, etc.) for schema changes
3. Coordinate infrastructure changes with data migrations in your pipeline

## Common Pitfalls

### State Management Issues

1. **Lost State File**: Without state, Terraform loses track of resources
2. **Corrupted State**: Manual edits or conflicts can corrupt state
3. **State Drift**: Resources changed outside Terraform become inconsistent

### Security Concerns

1. **Credentials in Code**: Never commit credentials to version control
2. **Overly Permissive Roles**: Follow least privilege principle
3. **State File Contains Secrets**: State can contain sensitive values; secure appropriately

### Resource Dependencies

1. **Implicit Dependencies**: Terraform may not detect all dependencies
2. **Delete-Before-Create**: Some updates may destroy and recreate resources
3. **Provider-Specific Limitations**: Some resources have provider-specific dependencies

## Code Examples

### Modular Terraform Structure for .NET Web Application

#### Module Definition (modules/app_service/main.tf)

```hcl
variable "name" {}
variable "location" {}
variable "resource_group_name" {}
variable "app_service_plan_id" {}
variable "dotnet_version" { default = "v6.0" }
variable "app_settings" { default = {} }

resource "azurerm_app_service" "app" {
  name                = var.name
  location            = var.location
  resource_group_name = var.resource_group_name
  app_service_plan_id = var.app_service_plan_id
  
  site_config {
    dotnet_framework_version = var.dotnet_version
    always_on                = true
  }
  
  app_settings = var.app_settings
}

output "hostname" {
  value = azurerm_app_service.app.default_site_hostname
}

output "id" {
  value = azurerm_app_service.app.id
}
```

#### Module Usage (environments/dev/main.tf)

```hcl
module "api_app_service" {
  source              = "../../modules/app_service"
  name                = "example-api-dev"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  app_service_plan_id = azurerm_app_service_plan.main.id
  dotnet_version      = "v6.0"
  
  app_settings = {
    "ASPNETCORE_ENVIRONMENT" = "Development"
    "ConnectionStrings__DefaultConnection" = "Server=tcp:${module.sql_server.server_fqdn},1433;Initial Catalog=${module.sql_database.db_name};..."
    "AzureAd__ClientId"     = var.azure_ad_client_id
    "AzureAd__TenantId"     = var.azure_ad_tenant_id
  }
}
```

### Integrating Terraform with .NET Application Configuration

Using Terraform outputs to generate configuration:

```hcl
# Output connection strings and configuration values
output "app_settings" {
  value = {
    ConnectionStrings__DefaultConnection = "Server=tcp:${azurerm_sql_server.example.fully_qualified_domain_name},1433;Initial Catalog=${azurerm_sql_database.example.name};..."
    AzureStorage__ConnectionString       = azurerm_storage_account.example.primary_connection_string
    ServiceBus__ConnectionString         = azurerm_servicebus_namespace.example.default_primary_connection_string
  }
  sensitive = true
}
```

Then use a script to transform these outputs into your application's configuration format.

## Further Reading

- [Terraform Documentation](https://www.terraform.io/docs)
- [Azure Provider Documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform: Up & Running](https://www.terraformupandrunning.com/) by Yevgeniy Brikman
- [Infrastructure as Code](https://infrastructure-as-code.com/book/) by Kief Morris

## Related Topics

- [Azure for .NET Applications](../../azure/getting-started.md)
- [AWS for .NET Applications](../../aws/getting-started.md)
- [Containerization with Docker](../../../containers/docker/dotnet-containers.md)
- [CI/CD Pipelines for .NET](../../../devops/cicd-pipelines.md)
- [Terraform Modules for .NET Infrastructure](modules.md)