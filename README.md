# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Episode : LInkerd - The Observability support
This repository contains the files utilized during the tutorial presented in the dedicated IsItObservable episode related to Linkerd.
<p align="center"><img src="/image/linkerd-stacked-color.png" width="40%" alt="LinkerD Logo" /></p>

What you will learn
* How to deploy linkerd
* how to ingest Linkerd Metrics
* how to enable the OpenTelemetry support


This repository showcase the usage of the OpenTelemtry Collector with :
* the HipsterShop
* K6 ( to generate load in the background)
* The OpenTelemetry Operator
* Nginx ingress controller
* Prometheus
* Linkerd
* Dynatrace

## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm

If you don't have any dynatrace tenant , then let's [start a trial on Dynatrace](https://www.dynatrace.com/trial/)

## Deployment Steps in GCP

You will first need a Kubernetes cluster with 2 Nodes.
You can either deploy on Minikube or K3s or follow the instructions to create GKE cluster:
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
    cloudtrace.googleapis.com \
    clouddebugger.googleapis.com \
    cloudprofiler.googleapis.com \
    --project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```

### 3.Clone the Github Repository
```
git clone https://github.com/isItObservable/Linkerd
cd Linkerd
```
### 4.Deploy Nginx Ingress Controller 
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
#### 1. get the ip adress of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* hipstershop
```
IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```
#### 2. Update the manifest files

update the following files to update the ingress definitions :
```
sed -i "s,IP_TO_REPLACE,$IP," kubernetes-manifests/k8s-manifest.yaml
```

### 6.Dynatrace
#### 1. Dynatrace Tenant - start a trial
If you don't have any Dyntrace tenant , then i suggest to create a trial using the following link : [Dynatrace Trial](https://bit.ly/3KxWDvY)
Once you have your Tenant save the Dynatrace (including https) tenant URL in the variable `DT_TENANT_URL` (for example : https://dedededfrf.live.dynatrace.com)
```
DT_TENANT_URL=<YOUR TENANT URL>
```


#### 2. Create the Dynatrace API Tokens
The dynatrace operator will require to have several tokens: 
* Token to deploy and configure the various components 
* Token to ingest metrics and Traces

##### Token to deploy
Create a Dynatrace token ( left menu Access TOken/Create a new token), this token requires to have the following scope:
* Create ActiveGate tokens
* Read entities
* Read Settings
* Write Settings
* Access problem and event feed, metrics and topology
* Read configuration
* Write configuration
* Paas integration - installer downloader
<p align="center"><img src="/image/operator_token.png" width="40%" alt="operator token" /></p>

Save the value of the token . We will use it later to store in a k8S secret
```
DYNATRACE_API_TOKEN=<YOUR TOKEN VALUE>
```
##### Token to ingest data
Create a Dynatrace token with the following scope:
* ingest metrics
* ingest OpenTelemetry traces
<p align="center"><img src="/image/data_ingest.png" width="40%" alt="data token" /></p>
Save the value of the token . We will use it later to store in a k8S secret

```
DATA_INGEST_TOKEN=<YOUR TOKEN VALUE>
```

##### Create the k8s secret for our tokenks
```
kubectl create namespace dynatrace
kubectl -n dynatrace create secret generic dynakube --from-literal="apiToken=$DYNATRACE_API_TOKEN" --from-literal="dataIngestToken=$DATA_INGEST_TOKEN"
```
#### 3. Deploy deploy the Dynatrace operator
```
kubectl apply -f https://github.com/Dynatrace/dynatrace-operator/releases/latest/download/kubernetes.yaml
kubectl apply -f https://github.com/Dynatrace/dynatrace-operator/releases/latest/download/kubernetes-csi.yaml
```
Update the Dynakube CRD with your tenant url:
```
sed -i 's,TENANTURL_TOREPLACE,$DT_TENANT_URL,' dynatrace/dynakube.yaml
kubectl apply -f dynatrace/dynakube.yaml
```


### 5.Prometheus

#### 1. Deploy Prometheus without Grafana
 ```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack  --set grafana.enabled=false 
```

#### 2. Enable remote Writer
To be able to send the Collector metrics to Prometheus , we need to enable the remote writer feature.
To enable this feature we will need to edit the CRD containing all the settings of promethes: prometehus

To get the Prometheus object named use by prometheus we need to run the following command:
```
kubectl get Prometheus
```
here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.32.1   1          22h
```
We will need to add an extra property in the configuration object :
```
enableFeatures:
- remote-write-receiver
```
so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```
After the update your Prometheus object should look  like :
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  generation: 2
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 30.0.1
    chart: kube-prometheus-stack-30.0.1
    heritage: Helm
    release: prometheus
  name: prometheus-kube-prometheus-prometheus
  namespace: default
spec:
  alerting:
  alertmanagers:
  - apiVersion: v2
    name: prometheus-kube-prometheus-alertmanager
    namespace: default
    pathPrefix: /
    port: http-web
  enableAdminAPI: false
  enableFeatures:
  - remote-write-receiver
  externalUrl: http://prometheus-kube-prometheus-prometheus.default:9090
  image: quay.io/prometheus/prometheus:v2.32.1
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  podMonitorNamespaceSelector: {}
  podMonitorSelector:
  matchLabels:
  release: prometheus
  portName: http-web
  probeNamespaceSelector: {}
  probeSelector:
  matchLabels:
  release: prometheus
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector:
  matchLabels:
  release: prometheus
  securityContext:
  fsGroup: 2000
  runAsGroup: 2000
  runAsNonRoot: true
  runAsUser: 1000
  serviceAccountName: prometheus-kube-prometheus-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
  matchLabels:
  release: prometheus
  shards: 1
  version: v2.32.1
```
### 3. Get the prometheus svc name 
```
PROM_SVC_NAME=$(kubectl get svc -l  app.kubernetes.io/instance=prometheus,app=kube-prometheus-stack-prometheus -ojson | jq -j '.items[].metadata.name')
```
update the hispter-shop deployment file :
```
sed -i "s,PROM_SVC_TO_REPLACE,$PROM_SVC_NAME," kubernetes-manifests/k8s-manifest.yaml
```

### 6.Deploy OpenTelemetry Operator

#### 1. Cert-Manager
The OpenTelemetry operator requires to deploy the Cert-manager :
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```
#### 2. OpenTelemetry Operator
```
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

### 7. Configure the OpenTelemetry Collector

#### Requirements
To be able to ingest the Distributed traces generated by the Online Boutique , it would be requried to modify `openTelemetry-manifest.yaml`  with
- your Dynatrace Tenant URL ( your dynatrace url would be `https://<TENANTID>.live.dynatrace.com` )
- A dynatrace datainges API token


#### Udpate the openTelemetry manifest file
```

CLUSTERID=$(kubectl get namespace kube-system -o jsonpath='{.metadata.uid}')
CLUSTERNAME="YOUR OWN NAME"
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," otelemetry/openTelemetry-sidecar.yaml
sed -i "s,CLUSTER_NAME_TO_REPLACE,$CLUSTERNAME," otelemetry/openTelemetry-sidecar.yaml
sed -i "s,TENANTURL_TOREPLACE,$DT_TENANT_URL," otelemetry/openTelemetry.yaml
sed -i "s,DT_API_TOKEN_TO_REPLACE,$DATA_INGEST_TOKEN," otelemetry/openTelemetry.yaml
```
#### Deploy the OpenTelemetry Collector
```
kubectl apply -f otelemetry/rbac.yaml
kubectl apply -f otelemetry/openTelemetry.yaml
kubectl apply -f otelemetry/openTelemetry-sidecar.yaml
```
### 8. Deploy Linkerd

#### Install the CLI 
To install the CLI manually, run:
```
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
```
Once installed, verify the CLI is running correctly with:
```
linkerd version
```
#### Validate your k8s cluster
To check that your cluster is ready to install Linkerd, run:
```
linkerd check --pre
```
#### Install the control plane onto your cluster
```
linkerd install | kubectl apply -f -
```
#### Annotate the default  namespace
```
kubectl annotate namespaces default linkerd.io/inject=enabled
```
### 9. Deploy the hipster-shop
```
kubectl create ns hipster-shop
kubectl annotate namespaces hipster-shop linkerd.io/inject=enabled
kubectl apply -f otelemetry/openTelemetry-sidecar.yaml -n hipster-shop
kubectl apply -f kubernetes-manifests/k8s-manifest.yaml -n hipster-shop
```
### 10. Deploy the linkerd extensions
#### the viz Extension
```
helm repo add linkerd https://helm.linkerd.io/stable
helm repo add linkerd-edge https://helm.linkerd.io/edge
helm install linkerd-viz -n linkerd-viz --create-namespace linkerd/linkerd-viz --set prometheus.enabled=false --set  prometheusUrl="http://$PROM_SVC_NAME.default.svc.cluster.local:9090"
```
#### Update the Prometheus Scraping rules
```
kubectl create secret generic addtional-scrape-configs --from-file=prometheus/additionnalscraping.yaml
```
Then we need to update the Prometheus configuration to add the extra scraping rules
```
kubectl get Prometheus
```
here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.37.0   1          22h
```
We will need to add an extra property in the configuration object :
```
additionalScrapeConfigs:
  name: addtional-scrape-configs
  key: additionnalscraping.yaml
```
so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```

### 11 Service profile

#### Create service profiles
Let's first build the first service that we know quite well: the frontend .
```
kubectl apply -f linkerd/service-profile.yaml -n hipster-shop
```
Let's build the other serviceprofile usingthe Auto-creation feature provided by Linkerd. 
that Will create the service profile based on the observed trafic.
```
linkerd viz profile -n hipster-shop cartservice --tap deploy/cartservice --tap-duration 20s |  kubectl apply -f -
```
for the productcatalogservice  
```
linkerd viz profile -n hipster-shop productcatalogservice --tap deploy/productcatalogservice --tap-duration 20s |  kubectl apply -f -
```
for the currencyService 
```
linkerd viz profile -n hipster-shop currencyservice --tap deploy/currencyservice --tap-duration 20s |  kubectl apply -f -
```
and checkoutservice:
```
linkerd viz profile -n hipster-shop checkoutservice --tap deploy/checkoutservice --tap-duration 20s |  kubectl apply -f -
```




