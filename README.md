# terraform-best-practices

Terraform Best Practices for AWS users.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  

- [Always Run terraform command with var-file](#always-run-terraform-command-with-var-file)
- [Manage s3 backend for tfstate files](#manage-s3-backend-for-tfstate-files)
  - [Notes](#notes)
- [Manage multiple Terraform modules and environment easily with Terragrunt](#manage-multiple-terraform-modules-and-environment-easily-with-terragrunt)
- [Retrieves state meta data from a remote backend](#retrieves-state-meta-data-from-a-remote-backend)
- [Use share modules](#use-share-modules)
  - [Notes](#notes-1)
- [Isolate environment](#isolate-environment)
- [Use terraform import to include as more resources you can](#use-terraform-import-to-include-as-more-resources-you-can)
- [Avoid hard code the resources](#avoid-hard-code-the-resources)
- [Format terraform codes](#format-terraform-codes)
- [Enable version control on terraform state files bucket](#enable-version-control-on-terraform-state-files-bucket)
- [Generate README for each module about input and output variables](#generate-readme-for-each-module-about-input-and-output-variables)
- [Update terraform version](#update-terraform-version)
- [Run terraform from docker container](#run-terraform-from-docker-container)
- [Troubleshooting with messy output](#troubleshooting-with-messy-output)
- [Run test](#run-test)
  - [Quick start](#quick-start)
  - [Run test within docker container](#run-test-within-docker-container)
- [Some updates for terraform 0.10.x](#some-updates-for-terraform-010x)
- [Minimum AWS permissions necessary for a Terraform run](#minimum-aws-permissions-necessary-for-a-terraform-run)
- [Useful documents you should read](#useful-documents-you-should-read)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Always Run terraform command with var-file

```
$ cat config/dev.tfvars

name = "dev-stack"
s3_terraform_bucket = "dev-stack-terraform"
tag_team_name = "hello-world"
 
$ terraform plan -var-file=config/dev.tfvars
```

With `var-file`, you can easily manage environment (dev/stag/uat/prod) variables.

With `var-file`, you avoid to run terraform with long list of key-value pairs ( `-var foo=bar` )

## Manage s3 backend for tfstate files

Terraform doesn't support [Interpolate variables in terraform backend config](https://github.com/hashicorp/terraform/pull/12067), normally you write a seperate script to define s3 backend bucket name for different environments, but I recommend to hard code it directly as below

Add below codes in terraform configuration files.
```
$ cat main.tf

terraform {
  required_version = "~> 0.10"

  backend "s3" {
    encrypt = true
  }
}
```

Define backend variables for particular environment
```
$ cat config/backend-dev.conf
bucket  = "<unique_bucke_name>-terraform-development"
key     = "development/service-1.tfstate"
encrypt = true
region  = "ap-southeast-2"
kms_key_id = "alias/terraform"
dynamodb_table = "terraform-lock"
```

### Notes

- bucket - s3 bucket name, has to be globally unique.
- key - Set some meanful names for different services and applications, such as vpc.tfstate, application_name.tfstate, etc
- dynamodb_table - optional when you want to enable [State Locking](https://www.terraform.io/docs/state/locking.html)

After you set `config/backend-dev.conf` and `config/dev.tfvars` properly (for each environment). You can easily run terraform as below:

```
env=dev
terraform get -update=true
terraform init -backend-config=config/backend-${env}.conf
terraform plan -var-file=config/${env}.tfvars
terraform apply -var-file=config/${env}.tfvars
```

## Manage multiple Terraform modules and environment easily with Terragrunt

Terragrunt is a thin wrapper for Terraform that provides extra tools for working with multiple Terraform modules. https://www.gruntwork.io

Sample for reference: https://github.com/gruntwork-io/terragrunt-infrastructure-live-example

Its README is too much talking, if you need a quick start, follow below steps:

```
# Install terraform and terragrunt
# Make sure you are in right aws account
$ aws s3 ls
# use terragrunt to deploy
$ git clone https://github.com/gruntwork-io/terragrunt-infrastructure-live-example.git
$ cd terragrunt-infrastructure-live-example
# for example, you want to deploy mysql in stage non-prod at region us-east-1
$ cd non-prod/us-east-1/stage/mysql
$ terragrunt plan
# Confirm everything works
$ terragrunt apply
```

So if you followed the setting in terragrunt properly, you don't need to care the backend state files and variable file path in different environments, even more, you can run `terragrunt plan-all` to plan all modules together.

## Retrieves state meta data from a remote backend

Normally we have several layers to manage terraform resources, such as network, database, application layers. After you create the basic network resources, such as vpc, security group, subnets, nat gateway in vpc stack. Your database layer and applications layer should always refer the resource from vpc layer directly via `terraform_remote_state` data srouce. 

```
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config{
    bucket = "${var.s3_terraform_bucket}"
    key = "${var.environment}/vpc.tfstate"
    region="${var.aws_region}"
  }
}
 
# Retrieves the vpc_id and subnet_ids directly from remote backend state files.
resource "aws_xx_xxxx" "main" {
  # ...
  subnet_ids = "${split(",", data.terraform_remote_state.vpc.data_subnets)}"
  vpc_id     = "${data.terraform_remote_state.vpc.vpc_id}"
}
```

## Use share modules

Manage terraform resource with shared modules, this will save a lot of coding time. 

For detail, you can start from below links: 

[terraform module usage](https://www.terraform.io/docs/modules/usage.html)

[Terraform Module Registry](https://registry.terraform.io/)

[Terraform aws modules](https://github.com/terraform-aws-modules)

### Notes

terraform modules don't support `count` parameter currently. You can follow up this ticket for updates: https://github.com/hashicorp/terraform/issues/953

## Isolate environment

Sometimes, developers like to create a security group and share it to all non-prod (dev/qa) environments. Don't do that, create resources with different name for each environment and each resource.

```
variable "application" {
  description = "application name"
  default = ""
}

variable "environment" {
  description = "environment name"
  default = ""
}

locals {
  name_prefix    = "${var.application}-${var.environment}"
}

resource "<any_resource>" {
  name = "${local.name_prefix}-<resource_name>"
  ...
}
```
Wth that, you will easily define the resource with meaningful and unique name, and you can build more same application stack for different developers without change a lot. For example, you update the environment to dev1, dev2, etc.

## Use terraform import to include as more resources you can

Sometimes developers manually created resources directly. You need to mark these resource and use `terraform import` to include them in codes.

[terraform import](https://www.terraform.io/docs/import/usage.html)

## Avoid hard code the resources

A sample:
```
account_number=“123456789012"
account_alias="mycompany"
```

The current aws account id or account alias can be input directly via [data sources](https://www.terraform.io/docs/providers/aws/).

```
# The attribute `${data.aws_caller_identity.current.account_id}` will be current account number. 
data "aws_caller_identity" "current" {}

# The attribue `${data.aws_iam_account_alias.current.account_alias}` will be current account alias
data "aws_iam_account_alias" "current" {}

# Set as [local values](https://www.terraform.io/docs/configuration/locals.html)
locals {
  account_id    = "${data.aws_caller_identity.current.account_id}"
  account_alias = "${data.aws_iam_account_alias.current.account_alias}"
}
```

## Format terraform codes

Always run `terraform fmt` to format terraform configuration files and make them neatly.

I used below codes in Travis CI pipeline (you can re-use it in any pipelines) to validate and format check the codes before you can merge it to master branch.

      - find . -type f -name "*.tf" -exec dirname {} \;|sort -u | while read m; do (terraform validate -check-variables=false "$m" && echo "√ $m") || exit 1 ; done
      - if [ `terraform fmt | wc -c` -ne 0 ]; then echo "Some terraform files need be formatted, run 'terraform fmt' to fix"; exit 1; fi
      

## Enable version control on terraform state files bucket

Always set backend to s3 and enable version control on this bucket. 

If you'd like to manage terraform state bucket as well, recommend to use this repostory I wrote [tf_aws_tfstate_bucket](https://github.com/BWITS/tf_aws_tfstate_bucket) to create the bucket and replica to other regions automatically. 

## Generate README for each module about input and output variables

You needn't manually manage `USAGE` about input variables and outputs. [terraform-docs](https://github.com/segmentio/terraform-docs) can do this job automatically.

```
$ brew install terraform-docs
$ cd terraform/modules/vpc
$ terraform-docs md . > README.md
```

For details on how to run `terraform-docs`, check this repository: https://github.com/segmentio/terraform-docs

There is a simple sample for you to start [tf_aws_acme](https://github.com/BWITS/tf_aws_acme), the README is generatd by `terraform-docs`

## Update terraform version

Hashicorp doesn't have a good qa/build/release process for their software and does not follow semantic versioning rules.

For example, `terraform init` isn't compatible between 0.9 and 0.8. Now they are going to split providers and use "init" to install providers as plugin in coming version 0.10

So recommend to keep updating to latest terraform version

## Run terraform from docker container

Terraform releases official docker containers that you can easily control which version you can run.

Recommend to run terraform docker container, when you set your build job in CI/CD pipeline.

```
TERRAFORM_IMAGE=hashicorp/terraform:0.9.8
TERRAFORM_CMD="docker run -ti --rm -w /app -v ${HOME}/.aws:/root/.aws -v ${HOME}/.ssh:/root/.ssh -v `pwd`:/app $TERRAFORM_IMAGE"
```

## Troubleshooting with messy output

Sometime, you applied the changes several times, the plan output always prompts there are some changes, essepecially in iam and s3 policy.  It is hard to troubleshooting the problem with messy json output in one line.

With the tool [terraform-landscape](https://github.com/coinbase/terraform-landscape), it improves Terraform plan output to be easier for reading, you can easily find out where is the problem. For details, please go through the project at https://github.com/coinbase/terraform-landscape

    # Install terraform_landscape
    gem install terraform_landscape
    # On MacOS, you can install with brew
    brew install terraform_landscape

    terraform plan -var-file=${env}/${env}.tfvars -input=false -out=plan -lock=false |tee report
    landscape < report

    # run terraform-landscape container as command
    alias landscape="docker run -i --rm -v $(pwd):/apps alpine/landscape:0.1.18"
    landscape --help
    terraform plan |tee report
    landscape - < report
    # Or
    terraform plan | landscape -

## Run test

Recommend to add [awspec](https://github.com/k1LoW/awspec) tests through [kitchen](https://kitchen.ci/) and [kitchen-terraform](https://newcontext-oss.github.io/kitchen-terraform/).

### Quick start

Reference: repo [terraform-aws-modules/terraform-aws-eks](https://github.com/terraform-aws-modules/terraform-aws-eks#testing)

### Run test within docker container

Reference: [README for terraform awspec container](https://github.com/alpine-docker/bundle-terraform-awspec)

## Some updates for terraform 0.10.x

After Hashicorp splits terraform providers out of terraform core binary from v0.10.x, you will see errors to complain aws, template, terraform provider version are not installed when run `terraform init`

```
* provider.aws: no suitable version installed
  version requirements: "~> 1.0"
```
Please add below codes to `main.tf`

```
provider "aws" {
  version = "~> 1.0"
  region  = "${var.region}"
}

provider "template" {
  version = "~> 1.0"
}

provider "terraform" {
  version = "~> 1.0"
}
```
## Minimum AWS permissions necessary for a Terraform run

There will be no answer for this. But with below iam policy you can easily get started.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSpecifics",
            "Action": [
                "ec2:*",
                "rds:*",
                "s3:*",
                "sns:*",
                "sqs:*",
                "iam:*",
                "elasticloadbalancing:*",
                "autoscaling:*",
                "cloudwatch:*",
                "cloudfront:*",
                "route53:*",
                "ecr:*",
                "logs:*",
                "ecs:*",
                "application-autoscaling:*",
                "logs:*",
                "events:*",
                "elasticache:*",
                "es:*",
                "kms:*",
                "dynamodb:*"
            ],
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Sid": "DenySpecifics",
            "Action": [
                "iam:*User*",
                "iam:*Login*",
                "iam:*Group*",
                "iam:*Provider*",
                "aws-portal:*",
                "budgets:*",
                "config:*",
                "directconnect:*",
                "aws-marketplace:*",
                "aws-marketplace-management:*",
                "ec2:*ReservedInstances*"
            ],
            "Effect": "Deny",
            "Resource": "*"
        }
    ]
}
```

Depend on your company or project requirement, you can easily update the resources in `Allow` session which terraform commands should have, and add deny policies in `Deny` session if some of permissions are not required.

## Useful documents you should read

[terraform tips & tricks: loops, if-statements, and gotchas](https://blog.gruntwork.io/terraform-tips-tricks-loops-if-statements-and-gotchas-f739bbae55f9)
