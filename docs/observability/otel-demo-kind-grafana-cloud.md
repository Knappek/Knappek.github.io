# Deploy OpenTelemetry Demo on a Kind cluster

We're following a slightly amended version of the [official OTel Demo App](https://opentelemetry.io/docs/platforms/kubernetes/helm/demo/) deployment on Kubernetes with the difference of integrating our Cluster with Grafana Cloud using Alloy.

## Prerequisites

- [Grafana Cloud stack](https://grafana.com/docs/grafana-cloud/get-started/)

## Deploy

1. Deploy a kind cluster with 3 Control Plane nodes and 3 Worker nodes by using the following `config.yaml`:

    ```yaml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
      - role: control-plane
      - role: control-plane
      - role: control-plane
      - role: worker
      - role: worker
      - role: worker
    ```

1. Deploy the cluster with

    ```shell
    kind create cluster --config config.yaml --name kind-3cp-3worker
    ```

1. For the OpenTelemetry Demo create the following `values.yaml` which disables all kinds of deployments which we don't need because we have Alloy deployed on the cluster already:

    ```yaml
    opentelemetry-collector:
      enabled: false
    jaeger:
      enabled: false
    prometheus:
      enabled: false
    grafana:
      enabled: false
    opensearch:
      enabled: false
    ```

1. Then deploy the demo:

    ```shell
    helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
    helm install otel-demo open-telemetry/opentelemetry-demo --values values.yaml -n otel-demo --create-namespace
    ```
