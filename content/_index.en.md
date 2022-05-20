+++
title = "The AWS Integration & Automation team's best practices for Terraform"
linkTitle = 'Home'
description = 'AWS IA Terraform Module Standards'
date = '2022-02-15'
weight = 1
chapter = true
+++

The standards and guide-lines detailed below are a set of working rules and principles for writing modules that will be stewarded by the AWS I&A team. It is based on input from many experienced Terraform users at AWS, HashiCorp, and community grown conventions.

We have standardized on the official [Terraform Glossary](https://www.terraform.io/docs/glossary). If you see unfamiliar terms, such as [Root Module](https://www.terraform.io/docs/glossary#root-module), check the external glossary.

**Why Focus on Modules?**

Publishing a Terraform module is the gold-standard for easing AWS customer on-boarding to new services. Modules allow for flexible but opinionated deployments that follow AWS best practices and enforce proper security. For questions, please contact AWS I&A: <aws-ia-eng@amazon.com>

## Table of Contents

* [Module Structure](#-module-structure)
* [Provider Configuration Guidelines](#-provider-configuration-guidelines)
* [General HCL Configuration Guidelines](#-general-hcl-configuration-guidelines)
* [Variable & Output Declaration Guidelines](#-variables-declaration-guidelines)
* [Output Declaration Guidelines](#-output-guidelines)
* [Pull Request Guidelines](#-pull-request-guidelines)

## Module Structure

All modules must maintain a similar structure that contains module code, examples, sub-modules (optional), and functional tests. These are laid out in our terraform repo template as boilerplate but are also detailed here:

### **Example repo - [terraform-aws-label](https://github.com/aws-ia/terraform-aws-label):**

```bash
$ tree
├── examples
│   ├── basic
│   │   ├── main.tf
│   │   ├── outputs.tf
│   │   └── variables.tf
│   └── formatted_tags
│       ├── main.tf
│       └── variables.tf
├── modules
│   └── my_sub_module/
├── test
│   ├── examples_basic_test.go
│   ├── examples_formatted_tags_test.go
│   └── label_test.go
```

All modules must contain at least one working deployment example. This is called `basic` by convention, but that is not required. Include examples for various usage patterns.

Examples can be complex at times because some deployments require large amounts of dependencies. In that case, you should include the required resources as a sub-module. [This module’s](https://github.com/aws-ia/terraform-aws-route53-recovery-controller/blob/main/examples/basic/main.tf#L23)`basic` example requires a multi-region configuration to be deployed prior to testing the module’s components. This is accomplished by referencing a sub-module in [basic/main.tf](https://github.com/aws-ia/terraform-aws-route53-recovery-controller/blob/main/examples/basic/main.tf#L23).

All modules must provide tests to guarantee provided functionality. Testing can be done using [Terratest](https://terratest.gruntwork.io/docs/getting-started/quick-start/) or the native [Terraform Test](https://www.terraform.io/language/modules/testing-experiment) (at the time of writing this, `terraform test` is still experimental and alpha, `terratest` is preferred until future releases of `terraform test` ). Tests should verify each example as well any other functionality. Example specific tests should be titled `examples_<example_name>_test.go`, tests that are generic to the module should be titled `<module_name>_test.go`

Please include your `go.mod` and `go.sum` files after running `go mod init github.com/aws-ia/<module name>`

### **File Naming:**

**Most** modules should contain only the following file names:

* `main.tf` - Your Terraform resources
* `outputs.tf` - Module outputs
* `provider.tf` - The [terraform block](https://www.terraform.io/language/settings) with `required_providers`
* `variables.tf` - Variable declarations

**Other common files:**

* `data.tf` - Includes `locals` declarations and data sources. Note: its common to have an occasional local or data source in main.tf instead.
* `alias.tf` - Included if you have aliased providers to declare, [example here.](https://github.com/aws-ia/terraform-aws-route53-recovery-controller/blob/main/alias.tf)
* `versions.tf` - Alternate name for `provider.tf`. This convention comes from the `terraform0.12-upgrade` command where once terraform code was upgraded from v0.11.x to v0.12.x a `versions.tf` file was create to enforce `terraform { required_version = ">= 0.12.0}"`

**Service named files:**

Often users want to create several files and separate terraform resources by service. This urge should be stifled as much as possible in favor defining resources in main.tf. If a collection of resources, for example IAM Roles and Policies, exceed 150 lines then it is reasonable to break that into its own files such as iam.tf. Otherwise all resource code should be defined in the main.tf.

## Provider Configuration Guidelines

Provider blocks should be declared in root modules by consumers of modules.

Provider blocks should not be declared in modules unless they are to specify `alias` providers to be used. No [authentication or configuration](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) parameters should be set unnecessarily to avoid enforcing arbitrary credential chain selections. If they must be set, they should be set using a variable with `default = null` to allow users to omit the configuration.

It is OK to define a `awscc` `user_agent` block in a module because these are appended to the provider block inherited from the root module.

## General HCL Configuration Guidelines

### **Resource Meta Name**

Resource meta names should be snake-cased and should be contextual to the resource being created. Meta names should not be used generically, aka `aws_s3_bucket.self` and should not be repetitive, aka `aws_s3_bucket.bucket`.

**Examples:**

```hcl
data "aws_region" "current" {}
```

The resource meta name is “current” is contextual as the data provided is for the current region. It would be possible to create `data "aws_region" "alternative" {}` as well to provide region information for an alternative region.

```hcl
resource "aws_route53recoveryreadiness_cell" "per_region" {
  for_each  = toset(var.regions)
  cell_name = "${var.name}-${each.value}"
}
```

The resource meta name is contextual in that the resources created are looped over a region variable and are thus created `per_region`.

### Resource Name

**Use prefix attributes where possible** - Many resources allow name values passed to generate randomness at the end using a prefix. Examples:

* `[aws_iam_role](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role#argument-reference)` has both `name` and `name_prefix`
* `[aws_s3_bucket](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket#argument-reference)` has both a `bucket` to name bucket and a `bucket_prefix` to name the bucket with a prefix.

### Dynamic Resources

### `for_each` vs `count`

Terraform can dynamically create resources using either [count](https://www.terraform.io/language/meta-arguments/count#the-count-meta-argument) or [for_each](https://www.terraform.io/language/meta-arguments/for_each). `for_each` should **always** be preferred over `count` except for circumstances where only count = 0 or 1. The reasoning for this comes from the behavior fundamental to lists vs maps; Lists are ordered; say you create 3 subnets `[subnet0, subnet1, subnet2]` . if you have to erase subnet 0 or 1, terraform's state file will see a change to the list and cause cascading unexpected changes. Using `for_each` resources are named using the map key

`aws_subnet.test[0].id` vs `aws_subnet.test["private_subnet0"].id`

You can delete `"private_subnet0"` without any fear of unintended consequences.

### Using `for_each` with a list

Because lists can be so helpful, you will often find a situation where you have a list and you want to create a resource dynamically. Since `for_each` requires a map, convert your list to a set `toset(var.mylist)` and terraform will use each entry as a key. Example:

```hcl
resource "aws_ssm_parameter" "params_from_list" {
  for_each = toset(["drew", "tony", "andy"])

  name  = each.key
  type  = "String"
  value = each.value
}

$ terraform state list
aws_ssm_parameter.params_from_list["drew"]
aws_ssm_parameter.params_from_list["tony"]
aws_ssm_parameter.params_from_list["andy"]

$ terraform state show aws_ssm_parameter.params_from_list["drew"]
{ ...
name  = "drew"
value = "drew"
... }
```

### Default Tags

All resource that can accept tags should. The terraform `aws` provider has a `default_tags` feature should not be used inside a module in favor of allowing the root module to define `default_tags`.

### Tags with AWSCC support

If creating resources with both `aws` and `awscc` provider its helpful to have sanitized tags for each provider because the formats are different (`aws` is `{ tagname = tagvalue }` and `awscc` is `{ Key = tagname, Value = tagvalue }`. The [terraform-aws-label](https://github.com/aws-ia/terraform-aws-label) module can accept either provider version and outputs tags formatted for both.

```hcl
module "aws_tags" {
  source = "aws-ia/label/aws"

  # AWS provider format as input
  tags = {
    "service" : "authorize",
    "managed_by" : "terraform"
  }
}

module "awscc_tags" {
  source = "aws-ia/label/aws"

  # AWSCC provider format
  tags = [
    { "key" : "service", "value" : "measurements" },
    { "key" = "managed_by", "value" : "terraform" }
  ]
}

# To use outputtted tags formatted for aws provider
module.aws_tags.aws_tags
module.awscc_tags.aws_tags

# To use outputtted tags formatted for awscc provider
module.aws_tags.tags
module.awscc_tags.tags
```

### Prefer “Attachment” Resources over Embedded

Some resources have pseudo resources embedded as attributes in them. Where possible, you should avoid using these embedded resource attributes and instead you should use the unique resource to attach that pseudo-resource. These resource relationships can cause chicken/egg issues that are unique per resource. Example is `aws_security_group`:

**Using embedded attribute (avoid this pattern):**

```hcl
resource "aws_security_group" "allow_tls" {
  ...
  ingress {
    description      = "TLS from VPC"
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = [aws_vpc.main.cidr_block]
    ipv6_cidr_blocks = [aws_vpc.main.ipv6_cidr_block]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
}
```

**With attachment resources (preferred):**

```hcl
resource "aws_security_group" "allow_tls" {
  ...
}

resource "aws_security_group_rule" "example" {
  type              = "ingress"
  description      = "TLS from VPC"
  from_port        = 443
  to_port          = 443
  protocol         = "tcp"
  cidr_blocks      = [aws_vpc.main.cidr_block]
  ipv6_cidr_blocks = [aws_vpc.main.ipv6_cidr_block]
  security_group_id = aws_security_group.allow_tls.id
}
```

## Variables Declaration Guidelines

Module variables are useful in most situations. Knowing which default values to set is crucial. Here we outline some usage patterns and advanced usage considerations.

### Variable & Output Declarations

All variables must have a defined `type` and `description`.

### Required Variables

To make a variable required for user to set, omit a `default` in the variable declaration and [consider if setting](https://www.terraform.io/language/values/variables#disallowing-null-input-values) `nullable = false` makes sense.

### Disruptive Optional Attributes as Variables

Many resource attributes are disruptive to the API calls for `Create` and `Update` conditions. For example, `aws_vpc` requires either a defined `cidr_block` or deriving a cidr from AWS IPAM using `ipv4_ipam_pool_id` and `ipv4_netmask_length`, however, you cannot pass all 3 attributes to the API even of they’re empty.

To account for situations like this, you can define the variable with a `default = null` , if the null value is used with the resource, terraform omits the attribute from the configuration.

### Variable Value Validation

Terraform allows you to [validate the content](https://www.terraform.io/language/values/variables#custom-validation-rules) a user passes to a variable. This should be used as appropriate. Examples:

**Example: Can be either `assertion` or `gating`:**

```hcl
variable "safety_rule_type" {
  description = "Type of safety rules to create. Can only be \"assertion\" or \"gating\"."
  type        = string
  default     = "assertion"

  validation {
    condition     = var.safety_rule_type == lower("assertion") || var.safety_rule_type == lower("gating")
    error_message = "Safety rule type can only be \"assertion\" or \"gating\"."
  }
}
```

More validation examples can be found [here.](https://dev.to/drewmullen/terraform-variable-validation-with-samples-1ank)

### Custom Objects

Terraform allows you to create [custom object types](https://www.terraform.io/language/values/variables#object) to constrain input that is allowed.

```hcl
variable "safety_rules" {
  description = "Configuration of the Safety Rules. Key is the name applied to the rule."

  type = map(object({
    wait_period_ms = number
    inverted       = bool
    threshold      = number
    type           = string
    name_suffix    = string
  }))
}
```

The attributes may be made optional using the [experimental feature](https://www.terraform.io/language/expressions/type-constraints#experimental-optional-object-type-attributes) `module_variable_optional_attrs` which is set in your terraform block:

```hcl
terraform {
  required_version = ">= 0.15.0"
  experiments      = [module_variable_optional_attrs]
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 3.68"
    }
  }
}

variable "hosted_zone" {
  description = "Info about the hosted zone. If the `name` or `zone_id` is not passed, a search will be performed using the values provided. Leave null to not create Route53 Alias records (required for LB functionality)."

  type = object({
    name         = optional(string)
    private_zone = optional(bool)
    vpc_id       = optional(number)
    tags         = optional(map(string))
    zone_id      = optional(string)
  })

  default = {
    name    = null
    zone_id = null
  }
}
```

### Defaults.tf for Custom Objects

When using variables with type `object` you can set defaults, however, if the user overrides even 1 attribute all defaults are overwritten. A way around this is by creating local variables that provide defaults in the absense of a particular value. Below is an example:

We have a custom object "user" with a required name and optional age. If user does not specify age, we want the value to be "none of your business!".

```terraform
variable "user" {
    type = object({
      name = string
      age  = optional(string)
  })
  default = {
    name    = ""
    age     = "none of your business!"
  }
}

output "user" {
  value = var.user
}
```

However, with the above code `age` will be `null` value. We can get around this using the defaults.tf convention and then reference using the local instead of the variable. Name of the local should be identical to the variable.

```terraform
locals {
  user = {
    name = var.user.name
    age = try(var.user.age, "none of your business")
  }
}

output "user" {
  value = local.user
}
```

### Validation vs Custom Objects

Custom objects are very nice but if used with `optional()` the resultant keys are set within the object as `null` unless specified. This can occasionally [cause a hindrance .](https://discuss.hashicorp.com/t/experiment-feedback-optional-attribute-keys-should-not-be-included-in-variable-value-unless-specified/34063) Sometimes it is better to avoid defining a custom object and instead enforce organization using `validation` blocks instead. Example:

**First key must be like a valid region, 2nd key must be contained in a list:**

```hcl
/*
cells_definition = {
  us-west-2 = {
     elasticloadbalancing = "arn:aws:elasticloadbalancing:us-west-2:<>:loadbalancer/app/<>"
     autoscaling          = "arn:aws:autoscaling:us-west-2:<>:autoScalingGroup:*:autoScalingGroupName/<>
  }
}
*/

validation {
  condition = alltrue([for _, k in keys(var.cells_definition) : can(regex("[a-z][a-z]-[a-z]+-[1-9]", k))]) && alltrue(flatten([
     for arns in var.cells_definition : [
     for service, arn in arns : contains(["apigateway", "autoscaling", "cloudwatch", "dynamodb", "ec2-volume",
                                            "ec2-vpc", "ec2-vpn-gw", "ec2-vpn-cgw", "ec2-vpn-conn", "elasticloadbalancing",
                                            "kafka", "lambda", "rds", "route53", "sns", "sqs"], service)
     ]
  ]))
error_message = "Supported service names are the keys defined in var.resource_type_name."
}
```

### Documenting Complex Maps

Documenting maps with expected keys can be difficult. We will show an [example from our friends at Palo Alto](https://github.com/PaloAltoNetworks/terraform-aws-vmseries-modules/blob/3492f4248f60b14a47927dd7dd58dfcd5ddc9aca/modules/vmseries/variables.tf#L69) of using a [heredoc](https://linuxize.com/post/bash-heredoc/) to express documentation in a human-readable format. Example is a shortened version from [here.](https://github.com/PaloAltoNetworks/terraform-aws-vmseries-modules/blob/3492f4248f60b14a47927dd7dd58dfcd5ddc9aca/modules/vmseries/variables.tf#L69-L107)

````terraform
variable "interfaces" {
  description = <<-EOF
  Map of the network interface specifications. Available types include `mgmt`, `public`, and `private`. Types have associated options listed below.

  Available options:
  - `device_index`       = (Required|int) Determines order in which interfaces are attached to the instance. Interface with `0` is attached at boot time.
  - `subnet_id`          = (Required|string) Subnet ID to create the ENI in.
  - `name`               = (Optional|string) Name tag for the ENI. Defaults to instance name suffixed by map's key.

  Example:
  ```
  interfaces = {
    mgmt = {
      device_index       = 0
      subnet_id          = aws_subnet.mgmt.id
      name               = "mgmt"
    }
  }
  ```
EOF

  validation {
    error_message = "Only valid key values for interface type are \"public\", \"private\", or \"mgmt\"."
    condition = length(setsubtract(keys(var.subnets), [
      "public",
      "private",
      "mgmt"
    ])) == 0
  }````

## Output Guidelines

Terraform allows you to provide formatted outputs from your modules. There are several ways to think about these. Many times you can assume which output values will be most relevant to your end user. When choosing which values to output think about:

* How will your module be used in a root module?
* How will your module be used as a `data.terraform_remote_state`?
* Does it make sense to output entire resources instead of formatted output?

## Pull Request Guidelines

Soon™ we will have a shared CI process across all I&A modules. In the mean time, documented here are the expectations for code submitted as a PR. Details about the upcoming CI can be found in the FAQ.

### Documentation

READMEs should be auto-generated using [terraform-docs](https://github.com/terraform-docs/terraform-docs). A common pattern used is to define a `.terraform-docs.yaml` in the [root](https://github.com/aws-ia/terraform-aws-route53-recovery-controller/blob/main/.terraform-docs.yaml), using a header for instructional documentation and the `terraform-docs` table appended to the bottom for content about provider versions, variable documentation, etc.

You must also run `terraform fmt -recursive` in your root directory.

### Static Analysis

Required Tools:

* [tflint][]
* [tfsec][]
* [kics][]

Run [tflint][]:

```bash
$ tflint --only=terraform_deprecated_interpolation --only=terraform_deprecated_index --only=terraform_unused_declarations --only=terraform_comment_syntax --only=terraform_documented_outputs --only=terraform_documented_variables --only=terraform_typed_variables --only=terraform_module_pinned_source --only=terraform_naming_convention --only=terraform_required_version --only=terraform_required_providers --only=terraform_standard_module_structure --only=terraform_workspace_remote
```

Run [tfsec][] in all root modules:

```bash
tfsec .
```

Run [kics][] in your root directory:

```bash
kics scan -p ./ -o ./
```

**TODO**: update kics or tfsec (need to determine which) to ignore overlapping tests

Your best judgment should be used when ignoring linting & security findings. For precarious `ignore`s please provide an explanation in the PR and/or via comment in the code base.

## Semantic Versioning

Official releases should be published using GitHub tags and releases based on [semantic versioning](https://semver.org/) guidelines. Once 1.0.0 has been published, you must consider module functionality lifecycle, breaking changes must be marked accordingly.

Release titles should be `vX.X.X`, tags should be `X.X.X`

## Publishing to Terraform Registry

Each module must be published to the Terraform Registry once its ready for use. Once the entry is created, future tags flow automatically to the registry. The initial creation will soon be automated. In the meantime, please reach out to the I&A team to create initial registry entries for new modules.

## Disallowed resources

The AWS I&A team does not permit the use of some resources in our modules for various reasons, such as the [AWS CloudFormation stack resource][tf_cfn] in the [`aws` provider][tf_aws], since it does not support drift detection, nor does it record the resources deployed by the template in state.
The current list of disallowed resources can be found in the [tfsec][tfsec_cfg] & [tflint config][tflint_cfg] config files.

[kics]: https://kics.io/
[tf_aws]: https://registry.terraform.io/providers/hashicorp/aws/latest
[tf_cfn]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudformation_stack
[tflint]: https://github.com/terraform-linters/tflint
[tflint_cfg]: https://github.com/aws-ia/terraform-repo-template/blob/main/.tflint.hcl
[tfsec]: https://aquasecurity.github.io/tfsec/
[tfsec_cfg]: https://github.com/aws-ia/terraform-repo-template/tree/main/.tfsec
