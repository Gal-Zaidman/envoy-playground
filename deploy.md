# Preparations

## Create the echo-grpc and reverse-grpc container image

1. Clone grpc-gke-nlb-tutorial

    `git clone https://github.com/GoogleCloudPlatform/grpc-gke-nlb-tutorial`

2. Build the echo-grpc app

    ```
    pushd grpc-gke-nlb-tutorial.git/echo-grpc
    docker build -t echo-grpc:latest .
    ```
3. Login to dockerhub account

    `docker login`

4. Push the image to your docker hub account

    ```bash
    docker tag echo-grpc gzaidman/echo-grpc
    docker push gzaidman/echo-grpc
    ```

## Create a kind cluster

1. [Install a kind cluster](https://kind.sigs.k8s.io/docs/user/quick-start/) with the following config:

`cat multi-node.yaml`
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "loadbalancer-ready=true"
- role: worker
- role: worker
```

    `kind create cluster --config multi-node.yaml`

1. Create the envoy namespace which we will work on

    `kubectl apply -f 00-envoy-ns.yaml`

## Setup Metallb

 Metallb is required to provide an externalIP to our loadBalancer Service, follow the [kind docs](https://kind.sigs.k8s.io/docs/user/loadbalancer/)

1. Create the metallb namespace
   
    `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/namespace.yaml`

2. Create the memberlist secrets
   
   `kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"`

3. Apply metallb manifest

    `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/metallb.yaml`

4. Get the address pool used by loadbalancers, output would be something like 172.XX.0.0/16

    `docker network inspect -f '{{.IPAM.Config}}' kind`
    
5. Create the configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.XX.255.200-172.XX.255.250
```

## Install Prometheus

Follow the [Quickstart instructions](https://github.com/prometheus-operator/kube-prometheus/tree/main#quickstart)

```bash
## Create the namespace and CRDs, and then wait for them to be available before creating the remaining resources

kubectl apply --server-side -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl apply -f manifests/
```

** Note that the kube-prometheus project installs a lot of components in the K8S cluster, most of them are probably not required for our use case, but it is the easiest way to get a full functioning prometheus and grafana solution

# Process

1. Create the config file for the envoy sidecar containers
    ```bash
    kubectl -n envoy create configmap envoy-echo-grpc --from-file=echo-grpc/envoy.yaml

    kubectl -n envoy create configmap envoy-reverse-grpc --from-file=reverse-grpc/envoy.yaml
    ```

- exposing sidecar on 8786 port on the container
- Using Filter envoy.httpconnectionmanager to handle HTTP trafic
- route_config, is used to define the routes for each domain to their respective clusters. Here we are keeping the domain as *, allowing all domains to pass-through.
- cluster, is used to defines the services that will be called based on the route. The lb_policy ROUNDROBIN, with type STATIC because it is a sidecar and needs to communicate to only one pod always which leads to the reason for keeping the address in socketaddress as localhost while port_value is what will be exposed by that particular service’s deployment.

2. Create the backend services deployment 

    ```bash
    kubectl apply -f echo-grpc/01-echo-grpc-deploy.yaml
    kubectl apply -f reverse-grpc/01-reverse-grpc-deploy.yaml
    ```

echo-grpc and reverse-grpc are test application and envoy is being deployed in the same pod as a sidecar container.
Config volumes are mounted so that the envoy can read the configmaps.

3.  Create the envoy services

    ```bash
    kubectl apply -f echo-grpc/02-echo-grpc-service.yaml
    kubectl apply -f reverse-grpc/02-reverse-grpc-service.yaml
    ```
    
4. Create the front-end envoy (GW) service

    `kubectl apply -f front-envoy/01-envoy-service.yaml`

3. Creating self-signed certificates
   
```bash
//Get the external IP
kubectl get svc/envoy -n envoy -owide

//Copy the LoadBalancer address in the EXTERNAL-IP section and do a nslookup and copy the IP address
nslookup <your load balancer aadess>

//Create a self-signed cert and key
openssl req -x509 -nodes -newkey rsa:2048 -days 365 -keyout privkey.pem -out cert.pem -subj "/CN=<ip-address>"

//Create a Kubernetes TLS Secret called envoy-certs that contains the self-signed SSL/TLS certificate and key:
kubectl create secret tls envoy-certs --key privkey.pem --cert cert.pem --dry-run -o yaml
```

4.  Create the front-end envoy (GW) config file
    kubectl -n envoy create configmap envoy-echo-grpc --from-file=echo-grpc/envoy.yaml

    `kubectl -n envoy create configmap front-envoy --from-file=front-envoy/envoy.yaml`
    
5.  Create the front-end envoy (GW) deployment

    `kubectl apply -f front-envoy/02-envoy-deployment.yaml`

6. Configure the prometheus operator to scrap for envoy metrics

    `kubectl apply -f service-monitor.yaml`

7. Edit the prometheus CRD to include the new sevice monitor

    ```bash
    kubectl -n monitoring edit prometheuses k8s

    # Add the following section
    serviceMonitorSelector:
      matchLabels:
        serviceMonitorSelector: envoy
    ```

8. Edit the prometheus cluster role to have required permissions

```bash
kubectl -n monitoring edit clusterrole prometheus-k8s
- apiGroups:
  - ""
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
```

9. Recreate the role binding because it is not updated after you update the role

```bash
kubectl -n monitoring get clusterrolebinding prometheus-k8s -oyaml >> tmp.yaml
kubectl -n monitoring delete clusterrolebinding prometheus-k8s
kubectl -n monitoring create -f tmp.yaml
rm tmp.yaml
```

# Refrences

 https://www.loginradius.com/blog/async/service-mesh-with-envoy/

 https://cloud.google.com/architecture/exposing-grpc-services-on-gke-using-envoy-proxy

 https://blog.scottlowe.org/2019/02/01/scraping-envoy-with-prometheus-operator/