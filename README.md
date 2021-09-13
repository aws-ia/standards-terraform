![AWS-ia banner](/assets/ghbg.png)



## License
- Licensed under Apache 2.0
- Include Amazon NOTICE.txt

## Taxanomoy
**Root Module:**  Every Terraform configuration has at least one module, typically known as its root module. In the case of AWS-ia modules the Root module are allways ment to be reuseable modules or build blocks. 

**Child Modules:** A module that has been called by another module is often referred to as a child module.

**Types of Reuseable AWS-ia Modules:**
- AWS Service Modules: A container that provide sensibale default for deploying an AWS service based on AWS-ia standards _(Example:VPC/Aurora/SQS)_ 
- Partner Modules: A container that leverags one or more _AWS Service Modules_ to build the infrastrure needed to instantiate a AWS partner product _(Example:Magento)_

## Module Structure

## Deployments
### Terraform Cloud

### Local Backends

## Development Standards

### Minimum Terraform Version 
#### Supported `1.0.5`

#### Root Module
- module must not contain provider block definitions
