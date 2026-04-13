# Module 8: Observability and Monitoring

## What is Observability?
Observability is the ability to understand the internal state of a system by examining its external outputs. In Kubernetes, observability is built on three pillars: logs, metrics, and traces. Together they allow you to detect problems, understand their root cause, and resolve them quickly. Without observability, operating a Kubernetes cluster in production is essentially flying blind.

## Logging
Logs are text records of events that occurred in your application or in Kubernetes system components. Kubernetes does not provide a built-in log aggregation system, but it makes container logs accessible through kubectl and supports integration with external logging platforms.

Key logging concepts:
- **kubectl logs pod-name** — view logs from a running Pod
- **kubectl logs pod-name -c container-name** — view logs from a specific container in a multi-container Pod
- **kubectl logs pod-name --previous** — view logs from the previous (crashed) container instance
- **kubectl logs -f pod-name** — stream (follow) logs in real time
- Logs are stored on the node where the Pod runs; they are lost if the node fails without a log aggregation system

Log aggregation tools:
- **Fluentd / Fluent Bit**: Lightweight log collectors deployed as DaemonSets to collect logs from all nodes and forward them to a backend
- **Elasticsearch + Kibana (EK)**: Popular log storage and visualization stack, often combined with Fluentd (EFK stack)
- **Loki + Grafana**: Lightweight log aggregation by Grafana Labs, designed specifically for cloud-native environments
- **Datadog / Splunk**: Enterprise log management platforms with Kubernetes integrations

## Metrics
Metrics are numerical measurements collected over time, used to track the health and performance of your cluster and applications. Kubernetes exposes metrics through the Metrics Server and through cAdvisor on each node.

Core metrics tools:
- **Metrics Server**: A lightweight in-cluster aggregator of resource usage data. Required for kubectl top and HPA to work.
  - kubectl top nodes — show CPU and memory usage per node
  - kubectl top pods — show CPU and memory usage per Pod
- **Prometheus**: The industry-standard open-source monitoring system for Kubernetes. Scrapes metrics from Pods, nodes, and Kubernetes components via HTTP endpoints.
- **Grafana**: Visualization platform that connects to Prometheus (and other sources) to display metrics as dashboards.
- **kube-state-metrics**: Exports metrics about the state of Kubernetes objects (Deployments, Pods, Nodes) rather than resource usage.
- **node-exporter**: Exports hardware and OS metrics from each node (CPU, disk, network).

## Prometheus and Grafana Setup Overview
Prometheus and Grafana are the most widely used monitoring stack for Kubernetes. Prometheus scrapes metrics from instrumented endpoints and stores them as time-series data. Grafana reads that data and displays it on customizable dashboards.

Typical setup:
1. Deploy Prometheus using the kube-prometheus-stack Helm chart (includes Grafana, alerting rules, and exporters)
2. Annotate Pods with prometheus.io/scrape: "true" to have Prometheus scrape them
3. Import pre-built Grafana dashboards (e.g., Kubernetes cluster overview dashboard ID 315)
4. Set up alerting rules in Prometheus to notify on high CPU, memory pressure, or Pod crash loops
5. Connect Alertmanager to Slack, PagerDuty, or email for notifications

## Distributed Tracing
Tracing tracks a request as it flows through multiple services in a microservices architecture. Each unit of work is called a span, and a collection of spans forms a trace. Tracing helps identify latency bottlenecks and failures in complex service graphs.

Popular tracing tools:
- **Jaeger**: Open-source distributed tracing system, CNCF project. Integrates with OpenTelemetry.
- **Zipkin**: Lightweight tracing system, widely used with Spring Boot applications.
- **OpenTelemetry**: The emerging standard for instrumentation. Provides a unified SDK and collector for logs, metrics, and traces.
- **Tempo (Grafana)**: Scalable distributed tracing backend that integrates natively with Grafana.

## Kubernetes Events
Kubernetes Events record what is happening inside the cluster — Pod scheduling decisions, image pulls, container restarts, and errors. Events are stored in etcd and expire after one hour by default.

Useful event commands:
- kubectl get events — list all events in the current namespace
- kubectl get events --sort-by=.metadata.creationTimestamp — sort by time
- kubectl describe pod pod-name — shows events related to a specific Pod at the bottom of the output
- kubectl get events --field-selector reason=BackOff — filter events by reason

## Health Checks and Uptime Monitoring
In addition to metrics and logs, external uptime monitoring ensures your Services and Ingresses are reachable from outside the cluster.

Tools for uptime monitoring:
- **Blackbox Exporter** (Prometheus): Probes endpoints over HTTP, HTTPS, TCP, and ICMP
- **Uptime Kuma**: Self-hosted uptime monitoring dashboard
- **Pingdom / Datadog Synthetics**: Cloud-based synthetic monitoring services

## Alerting Best Practices
Alerting is only valuable if alerts are actionable and not overwhelming. Over-alerting leads to alert fatigue, where operators start ignoring notifications.

Best practices:
- Alert on symptoms (high error rate, high latency) not just causes (high CPU)
- Set appropriate thresholds with sufficient duration before firing (e.g., CPU > 80% for 5 minutes)
- Group related alerts using Alertmanager routes
- Ensure every alert has a clear runbook or remediation guide
- Regularly review and prune alerts that are noisy or never actionable

## Why Observability is Non-Negotiable
In a distributed system like Kubernetes, failures are inevitable and often unexpected. Without logs, metrics, and traces, you cannot diagnose what went wrong, how long it lasted, or how to prevent it from happening again. Investing in observability is investing in the reliability and trustworthiness of your entire platform.