+++
date = '2026-07-03T13:44:21-03:00'
title = 'Shift-Left IaC: Building a Tested Terraform Module Factory with MiniStack'
author = 'Roberto Scudeller'
tags = ['terraform', 'infrastructure-as-code', 'shift-left', 'ministack', 'automation', 'ci/cd', 'github-actions']
+++


At **AS Inc Example** we decided to use Terraform as our cloud provisioning tool after a long debate about using an alternative approach with a Kubernetes operator. Many of our engineers are already familiar with Terraform structure language and it usually is the balance point when you compare with other solutions like Pulumi and Crossplane. We do evaluate these tools, they are amazing, but using a standard tool like Terraform, or its open-source version OpenTofu, sounds like a straight forward path.
In favor of Crossplane, we can highlight the same Kubernetes configuration language and a constant loop to detect and fix configuration drifts. However, it has the same Chicken and egg paradox issue as we discussed in our previous post: that we should have a Kubernetes cluster up and running to start deploying cloud infrastructure, but to have a Kubernetes cluster we need to implement it using code. We are convinced that we will return to it soon, as our company expands in the direction of multiple teams and microservices.

## The Standard

To keep everything in the same pattern and easy to review, we decided to use internal terraform tools such as `terraform fmt` and `terraform validate`. The `fmt` command option formats the code and makes it easy to read. Besides, it can be used in the CI/CD workflow to verify after a commit. In contrast with `validate` which requires a pre-initialization using `init` is doable inside a CI/CD.
Another tool that we adopted to keep our higher standards was **TFLint**, and this is very common in many CI/CD workflows. It can validate our code and find missing good practices, for instance, a variable without a type. [Our workflow](https://github.com/betorvs/iac-terraform-modules/blob/main/.github/workflows/push.yaml#L57-L76).
We decided to have a central repository with all Terraform modules and we should version each module individually. Using modules makes it easy to reuse code, avoiding anyone to start from scratch on every new project and we can make sure that we can apply any security policies in bulk because we do have a central place to target. For instance, we should have a AWS S3 bucket module with public access disabled and it will guarantee that no new S3 will become public.
To test the code, we discussed internally the usage of frameworks like **Terratest** and builtin method with `terraform test`. Both methods have their own limitations and benefits, such as [Terratest is a Go framework](https://terratest.gruntwork.io/docs/) and we can have new engineers coming who are not comfortable with it and `terraform test` will not give the full capabilities that we can have with a real programming language. As of now, we decided to start with `Terraform test` and require at least one assertion to verify if our module is working properly.
The last, but not least standard, is to Keep It Simple Sir (KISS, and I just changed it here to make sure that you notice), because we should not have a full blend of options for a simple module to be maintained by us. If we require a full optionally module it is probably better to use an open community one instead. 
Another important verification that we decided to use is [checkov](https://www.checkov.io/1.Welcome/What%20is%20Checkov.html), as this will test against best practices for all kinds of resources that we have plans to use such as Helm Charts, Terraform code, Github Workflows. Use it in as many cases as you can and always check when it makes sense to fix and when you should add a skip tag on your resource (for example my AWS S3 module code [does not require logging enabled](https://github.com/betorvs/iac-terraform-modules/blob/main/aws/s3bucket/main.tf#L4), but it is still a good practice). We check it in every [push](https://github.com/betorvs/iac-terraform-modules/blob/main/.github/workflows/push.yaml#L86) and in the [releases](https://github.com/betorvs/iac-terraform-modules/blob/main/.github/workflows/terraform-module-releaser.yml#L80-L96) we do post it in security GitHub tab.  

## Budget limitations 

As many other companies, here at **AS Inc Example** (my imaginary company with budget limitations) we do not have a full sandbox account in our cloud provider to test all our code before deciding how to make it. And as many engineers that already work with Infrastructure as code faced in the past, even with a code with correct format and passed all linters, it can have issues to apply. For instance, a resource name which follows DNS naming convention can break your code because this is a string variable that you input to your code. 
How can we go through a release pipeline with modules without real testing it? Here we decided to use a Cloud Provider emulator such as [MiniStack](https://github.com/ministackorg/ministack) for AWS. We can use it in our local machines and in CI/CD such as Github Workflows. We decided to use this emulator because it covers a lot of cases and in our local computers we can even test some AWS services. For instance, AWS DynamoDB can be fully emulated, from Terraform code to deploy it to test adding and removing keys on it. And many other cases are included too. Check it here <sup>[1](https://github.com/ministackorg/ministack#real-database-endpoints-rds) [2](https://github.com/ministackorg/ministack#real-redis-endpoints-elasticache) [3](https://github.com/ministackorg/ministack#ecs-with-real-containers)</sup>.

## The Release Workflow

As we know how to format, lint and test our code, we decided to open this repository in Github, and use tagging in Github to make versions of our code. To make it happen we agree that some code should be handled by another private repository, which will use these modules based on versions and we must not commit anything related to our internal policies or data externally. 
What does this mean? For instance, we do have a module to create a GitHub provider and role to be used in GitHub Workflows, but the content of AWS IAM Policy must be in our private repository and this module should only accept it as a parameter, in this case a variable. In this policy we will allow this AWS Role to access many AWS resources such as AWS ECR, AWS EC2 and so on. This is a hard requirement and before opening a pull request to create a new module it is discussed in an internal proposal. This is a request for proposal methodology in which our engineer creates an internal document, like a Google Doc, and they describe what that module will cover and what it proposes to solve. It must compare with external community options and with our modules repository, and guide why this is important. It must verify any security and data lake issues to make sure we do not expose any confidential information, such as AWS IAM Policies. It should cover budget implications and what kind of tests we can cover without creating real resources in our cloud provider. All engineers are welcome to contribute and ask questions. We can talk more about it in a future post. 

Example of module usage:
```hcl
module "github_oidc" {
  source = "git::https://github.com/betorvs/iac-terraform-modules.git?ref=aws/github-oidc/v1.0.0"

  role_attach_policy_arn = aws_iam_policy.github_policy.arn

  role_description = "github oidc role"

  repositories = [
    "repo:asincexample/iac-terraform:ref:refs/heads/main",
  ]
}
```


We do have a clear vision on how to make a workflow and now the missing connector is how to tag and version it. In the past we usually filter which module was changed and using a Linux bash based step we run a couple of “git” commands to create this tag. Nowadays we decided to use an open Github Actions which covers it and even publishes information inside a GitHub Wiki inside our repository. It's pretty cool and you can check our code [here](https://github.com/betorvs/iac-terraform-modules/blob/main/.github/workflows/terraform-module-releaser.yml).

Concluding our post, we structured how to propose new Infrastructure as code modules, what should look like a new code using formaters and linters. A budget covered pipeline to test this code using `terraform test` and a complete release workflow to create a tag for each change in these modules. I hope it can guide you in the future. 
