+++
date = '2026-06-06T14:23:48-03:00'
author = 'Roberto Scudeller'
title = 'ArgoCD, Projects and Kubernetes cluster bootstrap'
+++

We have great plans for our infrastructure and we start with a clean Kubernetes cluster, and we want to start it as soon as possible to allow our developers to start deploying new applications there. 

As we observed in our previous experiences, ArgoCD is a great way to follow the GitOps method and we decided to use it as many colleagues already have experience with it. We are preparing a full journal post about how we created our Kubernetes resources using Infrastructure as code in our main cloud provider and hope you'll come back to check it out. This post will be about Kubernetes bootstrap with ArgoCD. 

## The Strategy: GitOps & The App of Apps Pattern

After reading about “App of Apps” in ArgoCD, we decided to start a simple deployment, using a full manifest deployment, to keep it simple, and adding our first "Application" in our cluster. We talked a lot about what should be a good name for our Application that will create all other applications and we decided to use `bootstrap` as this is a clear term in our industry. 

After applying our first application, we connect ArgoCD with our git repository. We divided it into two teams, platform and security. Then we create an `AppProject` for each team and we agree that we will not use the `default` project right now. We did not touch any ArgoCD config map yet. This is a future task and we don't want to focus on it now. 

We agree that we should follow one big mono repository now, just to keep it simple (in the end, it's just me and my imagination) but I will discuss some good approaches here that can help your company. 

- Splitting Applications, or ApplicationSets, in one repository and pointing this to a centralized Kubernetes manifest repository. For instance, `argocd-applications-repo` and `kubernetes-manifests-repo`, then all co-workers will only access manifests repository and only platform teams will have access to applications repo. This is a good approach and split responsibilities very well, using it with code owners will help to divide and review pull requests in a good GitOps way. Inside the applications repository you can keep all AppProjects definitions and no one from any other team will touch it. 
- Using a similar structure as `argocd-applications-repo` and a specific team repository for kubernetes manifests is another good way to divide this workload. Each team, like access, metadata, billing and others, will have their own Kubernetes repository and ArgoCD will read it from that point. 

In both cases, splitting repositories by purpose is a good idea, it only will increase the operational overhead, but if you have multiple teams and a lot of people working, it's a good idea. And after a couple of time, you will need to work in some centralized workflows, continuous integration pipelines, to validate this kubernetes manifests and make it compliance with your company. 

## Designing the Repository Structure

This is our directory tree:
```bash
├── apps
├── bootstrap
└── manifests
```

In `bootstrap` directory, we keep "bootstrap" first application and `AppProjects` and applications pointing to `apps/team`. In our case, we started with platform and security projects. 

To keep it simple, we decided that inside `apps/` we will keep only ArgoCD `Applications`. And in case we want to deploy anything that is not a helm chart, we should add these raw manifest files in `manifests/` directory and an example of it is our full raw manifest for ArgoCD. 

## Multi-Tenancy and Hardening Security

What does this bring to us as a possibility? Or our contributors should add a raw manifest, or a wrapper helm chart (helm chart that imports another chart as dependency and only configure that application, maybe I will cover it in a next post), or they should ask to allow external repositories in their own project. Some examples: Security wants to use Falco, then it was added as `.spec.sourceRepos` in the security project. Same for the Cert Manager from the platform team. Always check what you want to allow from it. In a case of having splitted git repositories per team, you should only allow them to use their own project with limitations and create some alarms when anyone creates a new application using the “default” project. 

Another important security control here is to not allow creation of `AppProject` because it will allow any team to cross permission boundaries creating a higher privilege ArgoCD project that can create all kinds of resources. Our security team faced some issues because it first tried to block it as `AppProject` was a Cluster Kubernetes custom resource and after a couple of tests, they discovered that AppProject is a Kubernetes custom namespace resource, then using a `spec.namespaceResourceBlacklist` inside AppProject does this restriction. 

Please check out our repository [argocd-app-of-apps-repo](https://github.com/betorvs/as-corp-ex-argocd-repo) and check this [blog_post_02](https://github.com/betorvs/as-corp-ex-argocd-repo/tree/blog_post_02) in case too much happens when you access it. 
