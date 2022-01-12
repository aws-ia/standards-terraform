+++
title = "The AWS Integration & Automation team's best practices for Terraform"
linkTitle = "Home"
chapter = true
weight = 1
+++
#### Guide Stage: `Alpha`

{{% notice note %}}
The aws-ia Terraform standards are updated often. Please be sure to refer to this document often to keep updated!
{{% /notice %}}

## License

* Licensed under Apache 2.0
* Include Amazon NOTICE.txt

## Taxonomy

**Root module**: Every Terraform configuration has at least one module, typically known as its root module. In the case of aws-ia modules, the root module is always meant to be made up of reuseable modules or build blocks.

**Child modules**: A module that has been called by another module is often referred to as a child module.

Types of reuseable aws-ia Modules:

* aws-ia Service Modules: A container that provides a sensible default for deploying an AWS service based on aws-ia standards (Example: VPC/Aurora/SQS)
* aws-ia Partner Modules: A container that leverags one or more AWS Service Modules to build the infrastructure needed to instantiate an AWS partner product (Example: Magento)

## Module Structure

```text
../terraform-partnername-productname
├── LICENSE
├── NOTICE.txt
├── deploy
│   └── main.tf
├── main.tf
└── setup_workspace
    ├── variables.tf
    └── workspace.tf
```

**File**: `deploy/main.tf`

```hcl
# Example deployment using Terraform Cloud

# Defaults to TFC for remote backend
terraform {
  backend "remote" {}
  required_version = ">= 1.0.5"
}

# Use baseline VPC from aws-ia registery
<REPLACE ME!>

# Deploys example workload instance via root module
module "workload" {
  source = "../"
  # interface to be defined
  #
}
```

**File**: `/main.tf`

```hcl
# Root Module
# No backends should be defined by default in Root Module
# This module is intended to be called by one or more other modules and must not contain any provider blocks
```

**File**: `/setup_workspace/`

Creates the Terraform workspace in TFC. See the [registry](https://registry.terraform.io/modules/aws-ia/cloud_workspace/hashicorp/latest) for more information.

## Deployment Types

**Local**: Write Terraform state and configuration to local working directory.

**Terraform Cloud (TFC)**: Writes Terraform state and configuration to remote TFC workspace and stream it locally

{{% notice note %}}
For more into in how to setup TFC Workspace refer to the cloud_workspace module
{{% /notice %}}

## Development standards

### Minimum Terraform version

**Binary version**: `>= 1.0.5`

**Provider Version**: `~> v3.58.0`

### aws-ia module standards

Module `/main.tf` must not contain provider block definitions.

Deploy example `/deploy/main.tf` - When creating an example Terraform Cloud deployment for the module are creating refer to the source directly.

```hcl
module "self_deployment_example" {
  source = "../"
}
```

* If the example terraform cloud deployment requires a VPC or any other AWS service, use the Terraform registry path.
* Lock the version of any dependancies to the latest version at the time of release.

```hcl
module "vpc" {
  source    = "aws-ia/vpc/aws"
  version   = "0.0.2"
}
```
