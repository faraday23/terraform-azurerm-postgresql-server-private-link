# Azure Database for PostgreSQL Server

This repo contains an example Terraform configuration that deploys a PostgreSQL database using a private link endpoint in Azure.
For more info, please see https://docs.microsoft.com/en-us/azure/postgresql/overview.

## Version compatibility

| Module version    | Terraform version | AzureRM version |
|-------------------|-------------------|-----------------|
| >= 1.x.x          | 0.13.x            | >= 2.2.0        |


## Example Usage

```hcl
variable "srvr_id" {
  description = "identifier appended to srvr name for more info see https://github.com/openrba/python-azure-naming#azuredbforpostgresql"
  type        = string
}

variable "srvr_id_replica" {
  description = "identifier appended to srvr name for more info see https://github.com/openrba/python-azure-naming#azuredbforpostgresql"
  type        = string
}

variable "names" {
  description = "names to be applied to resources"
  type        = map(string)
}

variable "tags" {
  description = "tags to be applied to resources"
  type        = map(string)
}

# Configure Azure Providers
provider "azurerm" {
  version = ">=2.2.0"
  subscription_id = "00000000-0000-0000-0000-00000000"
  features {}
}

##
# Pre-Build Modules 
##

module "subscription" {
  source          = "github.com/Azure-Terraform/terraform-azurerm-subscription-data.git?ref=v1.0.0"
  subscription_id = "00000000-0000-0000-0000-00000000"
}

module "rules" {
  source = "git@github.com:openrba/python-azure-naming.git?ref=tf"
}

# For tags and info see https://github.com/Azure-Terraform/terraform-azurerm-metadata 
# For naming convention see https://github.com/openrba/python-azure-naming 
module "metadata" {
  source = "github.com/Azure-Terraform/terraform-azurerm-metadata.git?ref=v1.1.0"

  naming_rules = module.rules.yaml
  
  market              = "us"
  location            = "useast1"
  sre_team            = "alpha"
  environment         = "sandbox"
  project             = "postgresql"
  business_unit       = "iog"
  product_group       = "tfe"
  product_name        = "postgresql"
  subscription_id     = module.subscription.output.subscription_id
  subscription_type   = "nonprod"
  resource_group_type = "app"
}

# postgresql-server storage account
module "storage_acct" {
  source = "../postgresql_module/storage_account"
  # Required inputs 
  # Pre-Built Modules  
  location              = module.metadata.location
  names                 = module.metadata.names
  tags                  = module.metadata.tags
  resource_group_name   = "app-postgresql-sandbox-useast1"
}

# postgresql-server module
module "postgresql_server" {
  source = "github.com/openrba/terraform-azurerm-postgresql-server-vnet-rule.git?ref=v1.0.0"
  # Required inputs 
  srvr_id                   = "01"
  srvr_id_replica           = "02"
  resource_group_name       = "app-postgresql-sandbox-useast1"
  # Replica server required inputs
  enable_replica            = true
  create_mode               = "Replica"
  creation_source_server_id = module.postgresql_server.primary_postgresql_server_id
  # Pre-Built Modules  
  location = module.metadata.location
  names    = module.metadata.names
  tags     = module.metadata.tags
  # postgresql server and database audit policies and advanced threat protection 
  enable_threat_detection_policy = true
  # Storage endpoints for atp logs
  storage_endpoint               = module.storage_acct.primary_blob_endpoint
  storage_account_access_key     = module.storage_acct.primary_access_key  
  # Enable azure ad admin
  enable_postgresql_ad_admin     = true
  ad_admin_login_name            = "first.last@risk.regn.net"
  ad_admin_login_name_replica    = "first.last@risk.regn.net"
  # private link endpoint
  enable_private_endpoint        = false
  # Virtual network - for Existing virtual network
  enable_vnet_rule                 = true
  vnet_resource_group_name         = "app-postgresql-sandbox-useast1"      #must be existing resource group within same region as primary server
  vnet_replica_resource_group_name = "app-postgresql-sandbox-westus"       #must be existing resource group within same region as replica server
  virtual_network_name             = "vnet-postgresql-sandbox-eastus-1337" #must be existing vnet with available address space
  virtual_network_name_replica     = "vnet-postgresql-sandbox-westus"      #must be existing vnet with available address space
  allowed_cidrs                    = ["192.168.2.0/24"]   #must be unique available address space within vnet
  allowed_cidrs_replica            = ["172.18.1.0/24"]    #must be unique available address space within vnet
  subnet_name_primary              = "default2" #must be unique subnet name 
  subnet_name_replica              = "default2" #must be unique subnet name 
  # Firewall Rules to allow client IP
  enable_firewall_rules            = false
  firewall_rules = [
                {name             = "desktop-ip"
                start_ip_address  = "209.243.55.98"
                end_ip_address    = "209.243.55.98"}]
}
```
## Required Inputs

| Name | Description | Type | Default | 
|------|-------------|------|---------|
| resource_group_name | names to be applied to resources | `map of string` | n/a |
| location | Location for all resources | `string` | `"eastus"` | 
| replica_mysql_server_location | Azure region Location of replica server | `string` | `"westus"` | 
| names | names to be applied to resources | `map of string` | n/a |
| tags | tags to be applied to resources | `map of string` | n/a |
| enable_replica | Toggles on/off creation of replica server | `string` | `"false"` | 
| create_mode | Specifies to create a replica server, for example "Replica" will create replica server | `string` | `"Default"`| 
| event_scheduler | Indicates the status of the Event Scheduler. | `string` | n/a | 
| creation_source_server_id | the source server ID to use. use this only when creating a read replica server | `string` | n/a | 
| sku_name | Specifies the SKU Name for this MySQL Server. The name of the SKU, follows the tier + family + cores pattern (e.g. B_Gen4_1, GP_Gen5_8). | `string` | `"10240"` | 
| storage_mb | Azure database for MySQL pricing tier | `string` | `"GeneralPurpose"` | 
| mysql_version | MySQL version 5.7 or 8.0 | `string` | `"8.0"` | 
| enable_threat_detection_policy | toggles on or off the threat detection policy | `boolean` | `"false"` | 
| enable_mysql_ad_admin | toggles on or off creation of azure active directory administrator | `boolean` | `"false"` | 
| ad_admin_login_name | Specifies name of azure active directory administrator | `boolean` | `"false"` | 
| ad_admin_login_name_replica | Specifies name of azure active directory administrator | `boolean` | `"false"` | 
| enable_private_endpoint | toggles on/off private link endpoint (cannot be used inconjunction with vnet rules) | `boolean` | `"false"` | 
| virtual_network_name | Specifies name of virtual network, needs to be existing virtual network name. | `string` | n/a | 
| allowed_subnets | Specifies name cidr block of allowed subnets. must be unique available address space within vnet. | `list of string` | n/a | 
| subnet_name | Specifies name cidr block of allowed subnets. must be unique available address space within vnet. | `list of string` | n/a | 
| enable_vnet_rule | toggles on / off creation of vnet rules | `boolean` | `"false"` | 

## Quick start

1.Install [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli).\
2.Sign into your [Azure Account](https://docs.microsoft.com/en-us/cli/azure/authenticate-azure-cli?view=azure-cli-latest)


```
# Login with the Azure CLI/bash terminal/powershell by running
az login

# Verify access by running
az account show --output jsonc

# Confirm you are running required/pinned version of terraform
terraform version
```

Deploy the code:

```
terraform init
terraform plan -out azure-quickstart-01.tfplan
terraform apply azure-quickstart-01.tfplan
```
