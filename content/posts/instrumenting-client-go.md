---
title: "Instrumenting k8s.io/client-go with metrics"
date: 2021-09-11T11:00:51+01:00
draft: false
---

The Kubernetes [client-go](https://pkg.go.dev/k8s.io/client-go) package provides
a really handy set of interfaces for instrumenting its [REST
client](https://pkg.go.dev/k8s.io/client-go/tools/metrics) and
[workqueue](https://pkg.go.dev/k8s.io/client-go/util/workqueue#MetricsProvider) 
packages.

This is something that I didn't find highlighted or documented anywhere but I
was sure must exist when I found myself thinking about the best way to implement
metrics for my Kubernetes controllers.

It provides a very flexible and straightforward way to produce useful and
meaningful metrics without having to reinvent the wheel.

[The Kubernetes discovery code in Prometheus has a great example of using
both.](https://github.com/prometheus/prometheus/blob/v2.29.2/discovery/kubernetes/client_metrics.go)
