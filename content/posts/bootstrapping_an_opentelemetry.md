+++
date = '2026-08-03T11:48:46-03:00'
title = 'SRE Horizons: Bootstrapping an OpenTelemetry Data Plane with Prometheus and Loki'
author = 'Roberto Scudeller'
tags = [ 'kubernetes', 'CRDs', 'custom-resources', 'prometheus', 'open-telemetry', 'loki', 'grafana', 'observability', 'collector']
+++

Before going deeper with our stack, here at AS Inc Example, we discussed a lot about if we should use an internal tool or go straight to a third party vendor. Our engineers (a conversation between me and my second cup of coffee) argued that going directly with a third party will teach every developer how to use it with caution and always keep an eye in the budget, on the other hand, we discuss what will happen if someone forgot to remove a heavy logging application in debug mode in production for a couple of minutes or even worse, hours. Using a third party vendor is very beneficial as you can test all observability pillars without effort from platform teams. If you have a good budget, go for it and share your strategies with us.  
We mentioned in one of our previous posts about Prometheus Operator and its own Custom resources and how to handle it, and now, we will discuss our observability stack and what we delivered and what we can do in the future. Prometheus is a very popular choice for metrics and this is an important pillar for observability.

## The Three Pillars

Metrics, Logs and Traces are the pillars of observability. This is the basic setup that every good solution should bring to any platform. Right? Not in all cases. As we are starting in AS Inc Example, we don't want to cover all pillars at once, since we won't have a need for it yet. We decided to cover logs and metrics as a starting point. 
Why? Logs will cover what is going wrong with our applications and Metrics will cover from issues to capacity of our applications. And Traces will be handy when we work with multiple applications connected to each other, we are not talking yet about microservices, just when we do have more than one application talking with others. Traces are very useful. 

Prometheus is a no-doubt leader in metrics solutions when we talk about Kubernetes and metrics. Choosing a Prometheus Stack Helm to deploy all components to monitor our cluster was our first step of course, and it brings Node exporter, kube-state-metrics, Grafana and Alert Manager, Prometheus and its operator. Using it with ServiceMonitors and PodMonitors will give us an advantage in scaling metrics to all services and developers teams. 
For logging we decided to use Grafana Loki as this architecture uses a cheaper storage solution and many logs should be kept for long retention. Besides it gives us a great visibility in Grafana and lets us connect using command line tools and so on. Integrations like Examplars are very cool and we have an intention to use them in the future. 
We intentionally left one piece out of our platform offering to focus on keeping what we decided to use up to date and without issues. Having a full solution in the beginning will cost our teams extra effort to keep everything working, updated and configured.
No plans for other tools that we can use together like profiling or frontend synthetic tools. 

## Observability is easy

We discussed a lot about observability over time. In the beginning, everything works perfectly and everyone is happy. Over time, however, hidden operational and infrastructure costs start to surface.
Starting with the main causes of issues: High cardinality labels. In this case, both Prometheus and Loki will face it in the future. This is a design used by Prometheus to search and expose metrics using labels from Kubernetes resources. Depending on how many different labels and what kind of information our engineering will store there, it can grow exponentially and make Prometheus, or Loki, start facing issues with their own data. It's very well documented how to use this kind of solution and how to avoid this issue. 
Another problem that was discussed was about operational costs in both solutions, many of our engineers have issues managing this solution when it requires scale. Grafana Loki was designed to scale using microservices, splitting each role into different services which can help but over time it becomes even more complicated than our own services. Requiring more engineers and more time on tasks to keep it up and running. 
Prometheus' drawback is related with memory usage as Service Monitors and Pod Monitors start growing a lot and all scraping, plus heavier queries, for instance using offset to compare the current data with weeks ago, can use a huge amount of memory and it's not uncommon to find an engineer that worked on tasks or incidents related with it and the output was to increase Prometheus memory limits. 
Observability is a great improvement in the Tech industry but over time it can create hidden costs because it will require a lot of operational tasks that no one posts about. 


## Open Telemetry Collector

That's the reason why we decided to use Open Telemetry Collector. It can proxy all kinds of observability data, from logs, metrics to traces and this can handle it to any kind of backend, like Prometheus and Grafana Loki and Grafana Tempo (for traces). 
Why do we choose it? This is because we can decide to migrate our stack in the future. Replace Prometheus? Yes. Maybe using Grafana Mimir, or Victoria Metrics, or an external vendor. And this is the same for Grafana Loki as well. 
Using a vendor agnostic collector will give our platform team mobility to handle it in different ways as it changes overtime. 
This collector is growing in adoption and becoming an industry standard. We recommend to everyone to try it and test how it can help you. And share with us your results. 

Choosing a good tooling, or a highly trending tool is not enough anymore for Observability, we need to be prepared to change it in the future, maybe now we do have enough engineers to take care of it, but something can change and we need to drop our in house solution and point our collector to a new business partner which will store and show our data. Are you prepared for it? For most engineers this is very painful since they spend hours learning these tools and that solution is part of their stack and they want to keep it around like a favorite pet, but this is the most difficult part of observability, to be ready for changes. When you have in-house solutions, having a large number of wrong logs or metrics, having a bad trace sampling is not a problem. This will only cause an incident to the platform team. However, using an external vendor will cause a hole in the platform team budget. What we decided here is to start looking at these numbers with our developers teams and having it exposed to them. For instance, exposing a usage dashboard per team. 

We hope this post helps your teams to discuss what's important and what can be in their backlog. Share with us your thoughts and thanks for reading. 
