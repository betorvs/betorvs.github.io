+++
date = '2026-06-24T14:52:39-03:00'
title = 'Chicken and Egg Padadox'
author = 'Roberto Scudeller'
tags = [ 'ArgoCD', 'kubernetes', 'cluster bootstrap', 'terraform', 'CRDs', 'custom-resources']
+++

Implementing environments in cloud providers, or on-premises, nowadays is usually done using code, instead of that old wise guy who owns all kinds of bash scripts to install everything. A long time ago implementing a Linux server meant a lot of scripting and good luck to make it happen on our deadline. This automation era started quite a long time ago and I remembered starting using Puppet and it was a second generation of automation tooling. At **AS Inc Example** we decided to use Terraform for all cloud related resources and Kubernetes for all applications. 
Where do we find the chicken and egg paradox? We decided to share in this post the main cases we usually find when creating an infrastructure from scratch. 

## ArgoCD and Kubernetes

As we shared in this blog, we decided to use ArgoCD and start with a Bootstrap application which reads our repository and implements all applications in our Kubernetes cluster. Going over each ArgoCD wave, we will find ArgoCD itself, but wait? How can ArgoCD manage itself? 
Looking in the Kubernetes Control Operator loop, we can understand how this is possible. ArgoCD is looking for its own Kubernetes manifests, in the time of powerful AI tools we know that ArgoCD has no clue that it is applying its own configuration in Kubernetes. And what happens is ArgoCD will run `kubectl apply` using these files, or generated files in case of using Helm, and it will go to Kubernetes API Server. It will authenticate, validate and so on until store it in Etcd. From that point, Kubernetes Controller will assign it to Scheduler Controller; it will check for requests, limits, taints, constraints and assign it to a node and finally Kubelet will finish this deployment. In short, ArgoCD started an asynchronous update of its own and did not wait for this to finish, Kubernetes will handle it. 
ArgoCD is our first chicken and egg problem in the Kubernetes ecosystem and as you follow this is a manageable by Controller loop and we should only focus our attention when an application crashes after this update. For instance, if the ArgoCD application controller or repo-server crashes after it, and we receive an alert from our monitoring system, we should act immediately, as this compromises the whole automation in our environment. 

## Terraform state

Before we start our Kubernetes cluster in our cloud provider (my imaginary company Kubernetes is running a local machine but this is not the main case here), we should start with our monorepo with all terraform code that will create our resources, but to make it shareable between platform engineers, we need to use a remote state, but why? There's nothing inside our cloud provider, right? 
Terraform, or OpenTofu, is a great tool that has evolved a lot in the last years and in the recent version, higher than `1.14`, it handles it elegantly. We start creating a bucket, for instance AWS S3 bucket, with required configuration, and run it using terraform without a remote backend. This will create local files, like `terraform.tfstate`, and it will store your resource details. After it and if our code runs correctly, we will have a new, empty bucket to use as a remote backend state for our terraform. Create a new `backend.tf`, with all information required. Using a `state.config` file, key value file with the same configuration, we are able to re-run our `terraform init -backend-config=state.config` and terraform automatically offers to import local state to remote state. Just accept it and our paradox is over. In case you face any issue with this step, [this issue](https://github.com/hashicorp/terraform-provider-aws/issues/45316) recommends to use a different profile for terraform state, since in this phase, terraform is not loading AWS SDK as during plan phase, then authentications like `aws login` with IAM users will not be supported, at least not at the time of this post. 
Quick note: `state.config` has `profile=terraform` and backend.tf has `profile=default` for instance.  

`state.config`:
```
bucket  = "example-tfstate-007" 
key     = "global/s3/terraform.tfstate"
region  = "us-east-1"
profile = "terraform"
```

and `backend.tf`:
```h
terraform {
  backend "s3" {
    bucket       = "example-tfstate-007"
    key          = "global/s3/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
    profile      = "default"
  }
}
```

## Kubernetes Custom Resources

Yes, it can be another chicken and egg problem. Let me use our ArgoCD waves as an example. We installed Cert-Manager, Dvorah admission controller in wave 3, but imagine that we want to enable metric collectors on these applications. We are talking about `ServiceMonitors`, or `PodMonitors`, but prometheus-stack only happens in wave 5. If we try to install these Custom resources from Prometheus Operator, it will fail in ArgoCD, mark this application as OutofSync/Error, and the main objective is to have metrics collected will not happen, right? 
To resolve this, our recommended approach is to split Prometheus-stack deployment in two waves, in wave 2, [we deployed all Custom Resource Definitions](https://github.com/betorvs/as-corp-ex-argocd-repo/blob/blog_post_05/apps/platform/prometheus-stack/prometheus-crds-helm.yaml) and in wave 5, we deployed Prometheus Operator via [Prometheus-stack Helm chart](https://github.com/betorvs/as-corp-ex-argocd-repo/blob/blog_post_05/apps/platform/prometheus-stack/prometheus-helm.yaml). 
You can consider it as a minor issue and do not pay attention, but remember that our goal is to handle any Disaster Recovery readiness from day zero, which means, having Custom Resources in the right wave will make our recovery smooth and less error prone. Any engineer can do it without being involved in all the processes before it.  

Chicken and egg problems are good exercises about how to organize code, infrastructure and internal processes. We should always think before adding a new ticket in the backlog because we don't want to handle it right now. Having a good strategy for Disaster recovery will not use all our time and in the future, when a crisis occurs or during an audit exercise, we will be almost ready, requiring just a few minor adjustments. 
