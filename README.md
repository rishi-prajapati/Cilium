# Enhancing Kubernetes Network Observability using Cilium, Hubble and Prometheus

### Desciption
This project focuses on enhancing Kubernetes network observability by deploying Cilium with Hubble, integrated with Prometheus and Grafana. It enables real-time traffic monitoring and visualization of Pod-to-Pod, Pod-to-External, and External-to-Pod communication.

By leveraging Cilium’s eBPF-powered networking and Hubble’s observability features, we gain deep insights into Kubernetes traffic patterns. Prometheus is used for metrics collection, and Grafana provides visual dashboards to monitor and analyze network flows.

### How to run the project?

### Prerequisites

- Docker installed
- Helm installed
- kubectl installed
- Kind installed
- MetalLB installed

## Create a cluster
`kind create cluster --config kind.yaml`
## Add the Prometheus helm repository
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
## Install the Prometheus stack
`helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace`
## Add the Cilium repository
`helm repo add cilium https://helm.cilium.io/`
## Install cilium in monitoring namespace

```
helm install cilium cilium/cilium --version 1.16.5 \
  --namespace kube-system \
  --set prometheus.enabled=true \
  --set operator.prometheus.enabled=true \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set hubble.metrics.enableOpenMetrics=true \
  --set hubble.metrics.enabled="{dns,drop,tcp,flow,flow-to-world,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"
```
## Modify helm values
`helm get values cilium --namespace kube-system --output yaml > cilium.yaml`
## Add this under `metrics.enabled`
```
flows-to-world:labelsContext=source_namespace,source_app,destination_namespace,destination_app
flow:labelsContext=source_namespace,source_app,destination_namespace,destination_app
```
## Reapply the changes and restart the cilium agent
```
helm upgrade cilium cilium/cilium --namespace kube-system -f cilium.yaml && \
kubectl rollout restart daemonset/cilium -n kube-system && \
kubectl rollout restart daemonset/cilium-envoy -n kube-system
```
## Likewise customize the prometheus Helm chart values and add the configuration in `additionalScrapeConfigs` in `values.yaml`
`helm show values prometheus-community/kube-prometheus-stack -n monitoring > values.yaml`
```
- job_name: 'kubernetes-pods'
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels:
        [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: ${1}:${2}
      target_label: __address__

- job_name: 'kubernetes-endpoints'
  scrape_interval: 30s
  kubernetes_sd_configs:
    - role: endpoints
  relabel_configs:
    - source_labels:
        [__meta_kubernetes_service_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels:
        [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: (.+)(?::\d+);(\d+)
      replacement: $1:$2
```
## Apply the changes
`helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml`
## Access Hubble UI
`kubectl port-forward -n kube-system svc/hubble-ui 12000:80 &`

## Testing
### Pod to Pod Communication
To facilitate Pod-to-Pod communication, two deployments (deployment-a and deployment-b) and corresponding services (service-a and service-b) are created using the deployment.yaml manifest.
![default](https://github.com/user-attachments/assets/d96fc850-b9d8-48b4-812b-e52520a45113)

### External World to Pod Communication
To enable communication from the external world to your Kubernetes pods, Use MetalLB for LoadBalancer Support in kind
### Install MetalLB
`kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml`
`metallb-config.yaml` is created apply the changes
`kubectl apply -f metallb-config.yaml`

A service of type LoadBalancer is created using the `foo-service.yaml` manifest. This service facilitates external access to the application running inside the Kubernetes cluster. Additionally, two pods are deployed using the `foo-app.yaml` and `bar-app.yaml` manifests to handle incoming traffic and provide seamless connectivity to external clients. 

Apply these files.
### Test the service
Access the service using external ip
`curl http://172.18.255.X`
![etop](https://github.com/user-attachments/assets/17f18929-d4ba-41ff-860e-abb29322fef5)

### Pod to External world communication
To validate that a pod can communicate with the external world, deploy a pod using a lightweight Alpine container to periodically ping a public domain like google.com. A manifest is named as `deployment-alpline.yaml`
`kubectl apply -f deployment-alpine.yaml`
![ptoe](https://github.com/user-attachments/assets/d831dea6-06e7-4f89-8fc1-d3f914b24e0f)

### Create Dashboards to monitor traffic
#### Access Grafana
`kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring`
- Username: `admin`
- Password: `prom-operator`

  - ### Pod A to Pod B Communication
    ```
    sum by(source_app, destination_app, destination_namespace, source_namespace)(rate(hubble_flows_processed_total{source_app="pod-a", destination_app="pod-b", source_namespace="default", destination_namespace="default"}[$__rate_interval]))
    ```
  - ### Pod B to Pod A Communication
    ```
    sum by(source_app, destination_app, destination_namespace, source_namespace)(rate(hubble_flows_processed_total{source_app="pod-b", destination_app="pod-a"}[$__rate_interval]))
    ```
  - ### Pod to External World Communication
    ```
    sum by(source_app)(rate(hubble_flows_to_world_total{source_app="test-connectivity", source_namespace="default"}[$__rate_interval]))
    ```
  - ### External world to Pod communication
    ```
    sum by(destination_app)(rate(hubble_flows_processed_total{destination_namespace="default", destination_app="http-echo"}[$__rate_interval]))
    ```
    ![grafana](https://github.com/user-attachments/assets/aea31226-28a4-491e-ad14-9cbcc6b7cd73)

    

