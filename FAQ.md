# FAQ

### What is the AWS I&A team's intention for Terraform AWSCC Provider

In late 2021 [AWS announced the Cloud Control (CC)](https://aws.amazon.com/blogs/aws/announcing-aws-cloud-control-api/) API & [HashiCorp announced the AWSCC provider](https://www.hashicorp.com/blog/announcing-terraform-aws-cloud-control-provider-tech-preview) which leverages the CC API. This document is an outline of the intention behind AWS Integration & Automation (I&A) team leveraging the (as of writing this) alpha state AWSCC provider.

[HashiCorp also announced a partnership with I&A](https://www.hashicorp.com/blog/aws-hashicorp-collaborate-new-terraform-modules) to build out Terraform modules for AWS. As part of that effort I&A will create repositories that will be title pre-fixed `terraform-awscc-` to indicate that we are using the AWSCC provider on one or more resources. Due to the alpha nature of the provider, these modules are marked accordingly as alpha.

The intention is for I&A to help refine AWSCC provider by building modules that utilize it. We believe we can provide a particularly useful set of eyes because of our intersection between customers, partners, AWS, & Terraform.

These modules will be versioned via [semver](https://semver.org/) have robust test suites to provide confidence over time that customers can rely on them. Until AWSCC is taken out of alpha status, you do so at your own risk.

We will also be providing modules titled `aws`. It is possible that in the future these will be converted to `awscc` but that will be discussed openly and it will be marked as a `major` change in the module.


### What is the `user_agent`?

```terraform
provider "awscc" {
  user_agent = [{
    product_name    = "terraform-awscc-label"
    product_version = "0.0.4"
    comment         = "V1/AWS-D69B4015/376222271"
  }]
}
```

In AWS I&A repos you will see a `user_agent` defined in the provider block. This is a [feature that was added to AWSCC](https://github.com/hashicorp/terraform-provider-awscc/pull/247) and is optional. User-agent data is passed to AWS Cloud Control APIs. This is useful for discovery and compliance purposes, as the user-agent for each API call is available as a CloudTrail event. Teams can use this data to identify workloads running in their account such as 3rd party products, OU's, etc.
