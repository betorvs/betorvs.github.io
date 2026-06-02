+++
date = '2026-06-01T16:22:56-03:00'
title = 'E2E Helm Tests'
author = 'Roberto Scudeller'
tags = [ 'helm', 'kubernetes', 'e2e-framework', 'kind']
+++

# Moving Beyond Helm Lint: How to Run True E2E Tests on Your Charts

We’ve all been there: `helm lint` passes perfectly, your syntax looks flawless, and you confidently push your chart to production—only for it to crash because of a misspelled environment variable or a misconfigured service account. 

Linting ensures your files are valid YAML and follow best practices. But it doesn't tell you if your templates will *actually* deploy and function correctly inside a real Kubernetes cluster.

To fix that, we need automated End-to-End (E2E) testing. In this post, I’ll walk you through a simple, reproducible workflow to spin up a local ephemeral cluster, deploy your chart, run your test suites, and clean everything up.

---

## The E2E Testing Workflow

First try to run this example create in this repository below then apply it to your own helm chart.

1. Clone it:

```bash
    git clone git@github.com:betorvs/article-e2e-helm-tests.git
```
    
2. Make sure you have all requirements and using [Task](https://taskfile.dev/)

```bash
    task init
    task tidy
    task e2e-tests
```

3. Check the content of `example_kind_test.go` and verify `TestExample` function to understand what you can improve to test your application better.  

```go
podName := ""
for _, pod := range podList.Items {
	if strings.HasPrefix(pod.Name, "example-e2e-kind") {
		t.Logf("Found pod %v with status %v", pod.Name, pod.Status.Phase)
		podName = pod.Name
		break
	}
}
if podName == "" {
	t.Fatalf("Failed to find pod with name prefix example-e2e-kind")
}
```
<strong>Note:</strong> In this case I was looking for this pod in any state, but you can be more precise here and even call specific commands to test your application.


---

## Why This Matters

By wrapping this exact sequence into a local go script and re-use it in your CI pipeline (like GitHub Actions), you achieve absolute confidence in your deployments. If a template change breaks a deployment hook or an ingress configuration, you catch it in seconds on your local machine instead of minutes into a production outage.

All the code, sample charts, and automated scripts for this setup are available in my repository. 

Check out the full implementation here: **[github.com/betorvs/article-e2e-helm-tests](https://github.com/betorvs/article-e2e-helm-tests)**

Happy deploying!