+++
date = '2026-06-01T16:22:56-03:00'
title = 'E2E Helm Tests'
author = 'Roberto Scudeller'
tags = [ 'helm', 'kubernetes', 'e2e-framework', 'kind']
+++

# Moving Beyond Helm Lint: How to Run True E2E Tests on Your Charts

As part of the initial platform infrastructure assessment for AS Inc Example, one of my highest priority tasks was defining how we validate our configuration templates before they ever touch a GitOps pipeline or a live cluster.

Looking at common industry pitfalls, many teams rely solely on `helm lint`. My assessment of this approach is that it introduces an unacceptable risk for a production-grade platform. While linting ensures files are valid YAML and follow basic syntax rules, it represents a massive blind spot: it cannot verify if templates will actually deploy and function inside a live Kubernetes environment. A misspelled environment variable, a duplicate key, or a misconfigured service account will pass a lint check perfectly, only to cause a runtime crash later during an automated sync.

To eliminate this vulnerability from day one, I have established a mandatory End-to-End (E2E) testing standard for all core infrastructure charts at AS Inc Example. This entry documents the exact technical workflow of that assessment: spinning up an ephemeral local cluster, executing full template deployments, running live test suites, and handling automatic teardown.

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

For AS Inc Example, I wrapped this exact execution sequence into a local Go automation script that we can reuse directly within our GitHub Actions CI/CD pipelines.

By standardizing this validation mechanism, we achieve absolute confidence in our platform's core deployments. If a template modification accidentally breaks a deployment hook or an Ingress configuration, an engineer will catch it in seconds on their local machine—completely avoiding a costly production outage later.

The complete technical implementation, including the Go script and test templates developed during this assessment, is documented and available in the reference repository:

Check out the full implementation here: **[github.com/betorvs/article-e2e-helm-tests](https://github.com/betorvs/article-e2e-helm-tests)**

Happy deploying!