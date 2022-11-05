# Kubernetes documentation

## Components

### Node
a server (physical / virtual machine)

### Pod
- Contains a docker container
- Each pods get an IP, also get a new IP upon recreation

### Service
A permanent IP address, it is also a loadbalancer
- **External service**: a service that is accessible from an external source such as http://124.89.20.4:8080
- **Internal service**

### Ingress 
like external service but for https://my-app.com, does the forwarding to service

### ConfigMap
Just some configuration

### Secrets
config for secrets 

### Volumes
It’s a storage, if db pod get restarted, the data will be restarted as well, but with volumes this won't happen. 

### Deployment
A blueprint for pods, which specifies how many replica of a ped is needed.

- Database cannot have deployment and have replicas because of data inconsistencies. 

### StatefulSet
- Deployment for database/stateful apps.
- It is difficult to use this component.
- Database are often hosted outside k8s. 

## Processes
### Running on every node:
**Docker**
**kubelet**
- interacts with docker and node. 
- Starts the pod 
- Assign resources such as ram, etc 
**kubeproxy**

### Running on master node
#### Api Server
The client (command line, etc) will communicate with it, the only entry point to the server. 

#### Scheduler
Schedule the requests from api server 

#### Controller Manager
detects a crash, etc from pods and tries to recover the pod 

#### etcd
a storage for k8s itself 

## Tools
### minikube
- To test k8s on local machine 
- A node cluster run on virtual box

### kubectl commands
```sh
kubectl get nodes 
kubectl get pod 
kubectl get services 
kubectl get deployment 
 
kubectl create deployment NAME --image=IMAGE 
kubectl delete deployment NAME 

# Show auto generated deployment file, and you can edit it for example the image 
kubectl edit deployment NAME 

kubectl logs [pod name] 
 
# Show for example the docker download image status
kubectl describe pod [pod name] 
 
kubectl exec –it [pod name] -- bin/bash 
 
kubectl apply –f config_filename 
```

### Config file

#### Deployment
```yaml
apiVersion: apps/v1 
kind: Deployment 
metadata: 
  name: my-app 
  # (1) explained below 
  labels: 
    app: my-app 
# spec of the deployment 
spec: 
  # number of replicas 
  replicas: 1 
  # (2) explained below 
  selector: 
    matchLabels: 
      app: my-app 
  # pods config
  template: 
    metadata: 
      labels: 
        # (3) explained below 
        app: my-app 
    # spec of pods 
    spec: 
      containers: 
        - name: my-app 
          # image of the pod to use 
          image: nginx:1.16 
          # bind that image to this port 
          ports: 
            - containerPort: 8080 
```

#### Internal Service
```yaml
apiVersion: v1 
kind: Service  
metadata:  
  name: my-service  
spec:  
  selector: 
    # (4) explained below:  
    app: my-app 
  ports: 
    - protocol: TCP 
      port: 80 
      targetPort: 8080 
```
#### Explanation of config file
- (3): A pod has a label `app: nginx` (it can be any other key-value pair), 
- (2): Create a connection between deployment (2) and pod (3) 
- (1): This is the deployment label and will be used by the service selector (4) 
- (4): connect to the deployment (1) also to the pod (3)  
#### Secret
```yaml
apiVersion: v1
kind: Secret 
metadata: 
  name: my-secret
type: Opaque 
data:
  username: BASE_64_VALUE
  password: BASE_64_VALUE
```

#### External Service
```yaml
apiVersion: v1 
kind: Service  
metadata:  
  name: my-service  
spec:  
  selector: 
    app: my-app 
  # !! Make it an external service
  type: LoadBalancer
  ports: 
    - protocol: TCP 
      port: 80 
      targetPort: 8080
      # !! Make it an external service
      # port in the browser to access the service
      # it has a range 30000-32000
      nodePort: 30000
```
- if the command `kubectl get service` you can see the external ip address
- In minikube this is not working and showing  pending
  - In minikube run `minikube service [service name]`

## [01 - Example](01-example/NOTE.md)

## Ingress
You want to have my-app.com website and not IP:PORT. 
So you change your external service into internal and then use ingress. and ingress will redirect to that internal service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: myapp.com
    http:
      paths:
        backend:
          serviceName: myapp-internal-service
          servicePort: 8080
```

The yaml file alone won't be enough and it needs ingress controller

### Ingress Controller
A pod or a set of pods that does evaluation and process of ingress rules.

It evaluates all the rules then manages the redirections.

You can choose between many third-party application for Ingress Controller. `K8s Nginx Ingress Controller` is from kubernetes itself. It depends on the cloud service provide. 

### Ingress in cloud service:

Not the only way but one of the common way: 
1. Requests first hit the cloud load balancer (ALS) 
1. Redirect to Ingress controller
1. Ingress rule needs to be defined

### [Example Ingress for example 01](01-example-ingress/NOTE.md)

### Ingress default backend
When page is not found will call it.

# Not important section

## Namespace
Organize your resources in a namespace

### Out of box namespaces
#### kube-system
For k8s system use
#### kube-public
publicly accessible data. It is a configmap that contains data without authentication.
#### kube-node-lease
contains heartbeat/availability of nodes information.
#### default
default if you haven't create a new namespace

### Create namespace
`kubectl create namespace [name]`

Or it can be created with config file:
```
metadata: 
  namespace: some-name
```

`kubectl get pod` equals to `kubectl get pod -n default`

## Helm

Package manager for k8s, to package/distribute yaml files.

### Helm charts

Bundles of YAML files

### Template engine

Example:
```yaml
values.yaml
name: my-app
pod.yaml
metadata:
  name: {{ .Values.name }}
```
