# OpenTelemetry Agent

The OpenTelemetry collector offers a vendor-agnostic implementation of how to receive, process and export telemetry data. 
In this chart, the collector will be deployed as a daemonset, meaning the collector will run as an `agent` on each node. Agent runs in host network mode allowing you to easily send application telemetry data.

The included agent provides:

- [Coralogix Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/coralogixexporter) -  Coralogix exporter is preconfigured to enrich data using Kubernetes Attributes, which allows quick correlation of telemetry signals using consistent ApplicationName and SubsytemName fields.
- [Kubernetes Attributes Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor) Kubernetes Attributes Processor, enriches data with Kubernetes metadata, such as Deployment information.
- [Kubernetes Log Collection](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver) - native Kubernetes Log collection with Opentelemetry Collector. No need to run multiple agents such as fluentd, fluent-bit or filebeat.
- [Host Metrics](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/hostmetricsreceiver) - native Linux monitor resource collection agent. No need to run Node exporter or vendor agents.
- [Kubelet Metrics](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/kubeletstatsreceiver) - Fetches running container metrics from the local Kubelet. 
- [OTLP Metrics](https://github.com/open-telemetry/opentelemetry-collector/blob/main/receiver/otlpreceiver/README.md) - Send application metrics via OpenTelemetry protocol.
- Traces - You can send data in various format, such as [Jaeger](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/jaegerreceiver), [OpenTelemetry Protocol](https://github.com/open-telemetry/opentelemetry-collector/blob/main/receiver/otlpreceiver/README.md) or [Zipkin](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/zipkinreceiver).
- [Span Metrics](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/spanmetricsprocessor) - Traces are converted to Requests, Duration and Error metrics using spanmetrics processor.
- [Zpages Extension](https://github.com/open-telemetry/opentelemetry-collector/tree/main/extension/zpagesextension) - You can investigate latency and error issues by navigating to Pod's localhost:55516 web server. Routes are desribed in [OpenTelemetry documentation](https://github.com/open-telemetry/opentelemetry-collector/tree/main/extension/zpagesextension#exposed-zpages-routes)

## Prerequisites

###  Secret Key

Follow the [private key docs](https://coralogix.com/docs/private-key/) tutorial to obtain your secret key tutorial to obtain your secret key.

OpenTelemetry Agent require a `secret` called `coralogix-keys` with the relevant `private key` under a secret key called `PRIVATE_KEY`, inside the `same namespace` that the chart is installed in.


```bash
kubectl create secret generic coralogix-keys \
  --from-literal=PRIVATE_KEY=<private-key>
```

The created secret should look like this:
```yaml
apiVersion: v1
data:
  PRIVATE_KEY: <encrypted-private-key>
kind: Secret
metadata:
  name: coralogix-keys
  namespace: <the-release-namespace>
type: Opaque 
```

## Installation

First make sure to add our Helm charts repository to the local repos list with the following command:
```bash
helm repo add coralogix-charts-virtual https://cgx.jfrog.io/artifactory/coralogix-charts-virtual
```

In order to get the updated Helm charts from the added repository, please run: 
```bash
helm repo update
```

Install the charts:
```bash
helm upgrade --install otel-coralogix-agent coralogix-charts-virtual/opentelemetry-coralogix \
  -f values.yaml
```

# How to use it

## Available Endpoints

Applications can send OTLP Metrics and Jaeger, Zipkin and OTLP traces to the local nodes, as `otel-agent` is using hostNetwork .

| Protocol | Port 
| --- | --- 
| Zipkin | 9411 
| Jaeger GRPC | 6832 
| Jaeger Thrift binary | 6832
| Jaeger Thrift compact | 6831 
| Jaeger Thrift http | 14268
| OTLP GRPC | 4317
| OTLP HTTP | 4318

### Example Application environment configuration:

The following 
```
env:
  - name: NODE
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "$(NODE):4417"
```

# Performance of the Collector
## Picking the right tracing SDK span processor

OpenTelemetry tracing SDK supports two strategies to create an application traces, a “SimpleSpanProcessor” and a “BatchSpanProcessor.” 
While the SimpleSpanProcessor submits a span every time a span is finished, the BatchSpanProcessor processes spans in batches, and buffers them until a flush event occurs. Flush events can occur when the buffer is full or when a timeout is reached.

Picking the right tracing SDK span processor can have an impact on the performance of the collector.
We switched our SDK span processor from SimpleSpanProcessor to BatchSpanProcessor and noticed a massive performance improvement in the collector:



| Span Processor  | Agent Memory Usage	                          | Agent CPU Usage                     | Latency Samples       |
|---------|------------------------------------------|------------------------------------- | --------------------------------- |
| SimpleSpanProcessor    |    3.7 GB   | 0.5 | >1m40s |
| BatchSpanProcessor   | 600 MB  | 0.02 | >1s <10s| 


In addition, it improved the buffer performance of the collector, when we used the SimpleSpanProcessor, the buffer queues were getting full very quickly,
and after switching to the BatchSpanProcessor, it stopped becoming full all the time, therefore stopped dropping data.

#### Example:
```
import BatchSpanProcessor from "@opentelemetry/sdk-trace-base";
tracerProvider.addSpanProcessor(new BatchSpanProcessor(exporter));
```

# Infrastructure Monitoring

## Log Collection

Default installation collects Kubernetes logs. 

## Dashboards

Under the `dashboard` directory, there are:
- Host Metrics Dashboard
- Kubernetes Pod Dashboard
- Span Metrics Dashboard
- Otel-Agent Grafana dashboard

# Dependencies

This chart uses [openetelemetry-collector](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-collector) help chart. 
