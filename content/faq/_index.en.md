+++
title = 'Frequently asked questions (FAQ)'
linkTitle = 'FAQ'
description = 'Frequently asked questions of the AWS Integration & Automation team about Terraform standards'
date = '2022-01-12'
weight = 998
chapter = true
+++
## AWS providers and associated modules demystified

Terraform leverages providers to interact with infrastructure resources. In the case of AWS, there are two official providers available that can be used; the AWS Provider for Terraform, and the Terraform AWS Cloud Control Provider (AWSCC) provider. The AWS Provider for Terraform has been in existence for over 14 years, and is hand coded (with some code generated). The AWSCC Provider is the newest provider and is automatically generated from the CloudFormation Registry which means new features and services can be supported in Terraform right away.

Our partners at HashiCorp have a detailed blog post on this topic (https://www.hashicorp.com/blog/announcing-terraform-aws-cloud-control-provider-tech-preview). We encourage you to read that blog post.

The AWS Integration & Automation (AWS I&A) team authors and publishes Terraform modules that have validated provenance. Modules use specific providers in their implementation, and for convenience we have decided on a naming convention that will allow you to know what provider is being used by the module you are leveraging. This naming convention is `terraform-<provider>-<modulename>`.

For example, you will see a module named `terraform-aws-rds-aurora` which informs you that this module is leveraging the AWS Provider for Terraform. You would also see another module named `terraform-awscc-waf` which informs you that it is leveraging the AWSCC Provider. [You can use both providers side by side](https://www.hashicorp.com/blog/announcing-terraform-aws-cloud-control-provider-tech-preview#using-two-providers) and we often do when writing AWSCC modules. The long term goal is to build robust modules that fully utilize the AWSCC provider.

We standardize on semantic versioning to provide assurance of compatibility and provide robust testing of modules to provide assurance of functionality.

The AWS CloudControl API and associated AWSCC Provider for Terraform will enable our customers to leverage the newest features and services using the IaC tooling of their choice. The AWS I&A team will prefer the AWSCC Provider when authoring or updating module choice and. Provider and the CloudControl APIs we see the AWSCC Provider mature and move into GA, you will see our newest modules prefer to use the AWSCC Provider.


## What is the `user_agent`?

```terraform
provider "awscc" {
  user_agent = [{
    product_name    = "terraform-awscc-label"
    product_version = "0.0.4"
    comment         = "V1/AWS-D69B4015/376222271"
  }]
}
```

In AWS I&A repos you will see a `user_agent` defined in the provider block. This is a [feature that was added to AWSCC](https://github.com/hashicorp/terraform-provider-awscc/pull/247) and is optional. `user-agent` data is passed to AWS Cloud Control APIs during `terraform apply`. This is useful for discovery and compliance purposes, as the user-agent for each API call is available as a CloudTrail event. Teams can use this data to identify workloads running in their account such as 3rd party products, OU's, etc. The above example defines 3 fields:

- `product_name`: The user friendly name of the module / repo. github repo names are fungible.
- `product_version`: The tag version of the module.
- `comment`:
  - `V1`: Version 1 of our defined meta-data payload format.
  - `AWS-D69B4015`: A unique, searchable identifier.
  - `376222271`: The github repo id, should be unique to each module and non-fungible. You can find a repo ID using `gh api repos/aws-ia/terraform-awscc-label --jq .id`. Download `gh` [here](https://cli.github.com/)

`user_agent` is a list with entries that are appeneded allowing you to define many in the same block and/or define one per module. We define a `user_agent` payload per module. This means that a Root module execution will push module meta data to your CloudTrail log.

## Recommended deployment patterns for Terraform

There are many ways to deploy or execute Terraform. We recommend either via [HashiCorp Terraform Cloud](https://github.com/aws-ia/terraform-hashicorp-cloud_workspace) or via [AWS S3 with DynamoDB for state locking](https://www.terraform.io/language/settings/backends/s3).


## Are modules protected by CI Automation?

Terraform Module pull requests (PRs) are subject to continuous integration (CI) automation for quality, linting, and security scanning. We currently leverage [AWS CodePipeline](https://aws.amazon.com/codepipeline/) and [AWS CodeCommit](https://aws.amazon.com/codecommit/), a process that is private to AWS employees only. It is a heavy work-in-progress and we intend to expose information about it ASAP. For this reason, [GitHub Actions](https://github.com/features/actions) has been disabled. We are open to feedback on what should be included in our CI so please use your PR to discuss.
