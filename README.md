![AWS-ia banner](/assets/ghbg.png)

Guide Stage: `Alpha`
> Note: AWS-ia Terraform standards are updated often. Please be sure to refer to this document often to keep updated!

## License
- Licensed under Apache 2.0
- Include Amazon NOTICE.txt

## Taxanomoy
**Root Module:**  Every Terraform configuration has at least one module, typically known as its root module. In the case of AWS-ia modules the Root module are allways ment to be reuseable modules or build blocks. 

**Child Modules:** A module that has been called by another module is often referred to as a child module.

**Types of Reuseable AWS-ia Modules:**
- AWS-ia Service Modules: A container that provide sensibale default for deploying an AWS service based on AWS-ia standards _(Example:VPC/Aurora/SQS)_ 
- AWS-ia Partner Modules: A container that leverags one or more _AWS Service Modules_ to build the infrastrure needed to instantiate a AWS partner product _(Example:Magento)_

## Module Structure

```
../terraform-partnername-productname
├── LICENSE
├── NOTICE.txt
├── deploy
│   └── main.tf
├── main.tf
└── setup_workspace
    ├── variables.tf
    └── workspace.tf
```
**File: deploy/main.tf**
```
# Example deployment using Terraform Cloud

# Defaults to TFC for remote backend
terraform {
  backend "remote" {}
  required_version = ">= 1.0.5"
}

# Use baseline VPC from aws-ia registery
module "workload_vpc" {
  source               = "aws-ia/vpc/aws"
  version              = "0.0.3"
  name                 = "workload-vpc"
  region               = var.region
  cidr                 = "10.0.0.0/16"
  public_subnets       = ["10.0.0.0/20"]
  private_subnets_A    = ["10.0.16.0/20", "10.0.32.0/20", "10.0.48.0/20"]
  enable_dns_hostnames = true
  tags                 = {}
  create_vpc           = true
}

# Deploys example workload instance via root module
module "workload" {
  source = "../"
  # interface to be defined
  # 
}
```

**File: /main.tf**

```
# Root Module
# No backends should be defined by default in Root Module
# This module is intended to be called by one or more other modules and must not contain any provider blocks
```

**File: /setup_workspace/**

Creates and Terraform workspace in TFC. See registry [link](https://registry.terraform.io/modules/aws-ia/cloud_workspace/hashicorp/latest) for more information


## Deployment Tyoes
**Local** - Write terraform state and configuration to local working dir
**Terraform Cloud** Writes terraform state and configuration to remote TFC workspace and stream it locally

> For more into in how to setup TFC Workspace refert to the [cloud_workspace module](https://registry.terraform.io/modules/aws-ia/cloud_workspace/hashicorp/latest)

## Development Standards

### Minimum Terraform Version 

**Binary Version** `1.0.5`

**Provider Version** ~> `v3.58.0`

#### AWS-ia Module Standards
Module `/main.tf` Must not contain provider block definitions

Depoly Example `/deploy/main.tf` 
 - When creating an example terraform cloud deployment for the module are creating refer to the source directly.
```
module "self_deployment_example" {
source = "../"
}
 ```
 - If the example terraform cloud deployment requires a VPC or any other AWS Service use the Terraform registry path.
 - Lock the version of any dependancies to the latest version at the time of release.
```
module "vpc" {
  source    = "aws-ia/vpv/aws"
  version   = "0.0.2"
  }
```
 

