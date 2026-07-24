+++
date = '2026-07-24T17:14:18-03:00'
title = 'Overcoming the GitOps Blindspot with HashiCorp Vault and ESO'
author = 'Roberto Scudeller'
tags = ['gitops', 'secrets', 'vault', 'eso', 'hashicorp-vault', 'external-secrets']
+++

Many platform engineers will feel anxious to release their platform “babies” to grow with a good dose of coding and good practices as a proud father. However, when will this happen? We are running policies to validate security and other compliance rules, validating images, runtime monitors, right? Now it's time to start accepting end user applications inside our environment. 
Not that fast yet. Even good developers following 12 factor methodology still require passwords and other kinds of resources that must be protected from malicious actors. Then software such as Hashicorp Vault and External Secrets Operator came handy. 

## Vault

We decided to use Vault as this is an industry standard and we can start using this open-source version right away without requesting a budget for finance. As this is a standard we can use an enterprise version or use any other Cloud similar resources to replace it in case of operational overhead. 
Using it in a local Cluster is a good starting point but it still requires manual intervention when it's created. What kind of actions I'm talking about? After this service is running, it will still not be ready in Kubernetes. Our security engineers will execute initialization commands (as I did for my imaginary company in my local lab) and save important output. After a `vault init`, it outputs 5 unseal keys and initial root password. Save it because on every restart, it will be required. Remember to add a note in our Disaster Recovery document as well. 
After initialization, our Vault pod started and it's running, then it is time to enable Kubernetes authentication to allow others services to interact with it. Why do we need other services since Vault Helm charts can inject secrets using Kubernetes Admission Webhooks? In this case, you should be aware of the ArgoCD drift state on these resources. It can not work for all kinds of applications, our new ones can be designed with this in mind, but old ones cannot be compatible with it. It's always a trade-off and you can balance your case and choose it well. 
At **AS Inc Example** we decided to use External Secrets Operator as the main way to deliver sensitive data in our clusters.

## ESO

External Secrets Operator, aka ESO, is a fully compatible Kubernetes Operator that can connect with a vast number of secrets resources and deliver them as a simple Kubernetes Secret that can be consumed by our Pods. 
Connecting it with Hashicorp Vault is straightforward and requires another command inside Vault, but after it, you can create any kind of sensitive data in Vault and any developer should only create a Custom resource as External Secret. It will trigger ESO and it will consult Vault and output a secret as our developer expected. 
We decided to use Cluster Secret Store as our main resource and let developers sync it in their own namespaces. It's important to remember that when creating access to developers to any Kubernetes cluster, I'm talking more specifically about Role Based Access Control (RBAC) to access our cluster using `kubectl` command line tool, they should not have access to Kubernetes Secrets, because it will break the purpose of Vault and ESO and expose sensitive data. 

## Don't use passwords

We also recommend not using passwords in **AS Inc Example**, which makes everyone confused in the beginning. We explain that passwords inside Cloud providers require rotation, and many applications will require restarting to read it from environment variables, and writing passwords as secrets will require storing it in Kubernetes Etcd. As a security measure, we ensure that our Etcd is encrypted by default. Nevertheless, this is not the only way to access Cloud resources. 
We do recommend using Cloud provider roles to access resources. In AWS it's called IRSA, which stands for IAM Role for Service Accounts, or the newest Pod Identity. In other providers you can find the same kind of technology, such as GCP Workload Identity and Azure Microsoft Entra Workload ID. What this does basically is to use role based access from your application service account. Using it we avoid storing credentials, we can enforce Least privilege Access and automatic expiration as role base access is attached with session timeout. 

Hashicorp Vault can also create short-lived dynamic credentials on demand which automatically expire. Integrating our External Secrets Operator pipeline with these dynamic engines to eliminate static cloud keys completely is definitely a feature that we will cover in the future. 

Having a company option to deliver passwords and any kind of sensitive data is a foundation for a good platform. Here we decided to use these options. What about your company? Tell us what you decided to use and what your requirements are.
