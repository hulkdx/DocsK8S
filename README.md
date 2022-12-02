# Kubernetes documentation

## Components

### Node
a server (physical / virtual machine)

- Master node: is a node that manages worker nodes.

### Pod
- Contains a docker container
- Each pods get an IP, also get a new IP upon recreation

### Service
A permanent IP address, it is also a loadbalancer
- **External service**: a service that is accessible from an external source such as http://124.89.20.4:8080, it can be one of the following types:
- **Internal service**

### Ingress 
like external service but for https://my-app.com, does the forwarding to service

- `ingress` means entering the cloud
- `egress` means exiting the cloud

### ConfigMap
Just some configuration

### Secrets
config for secrets 

### Volumes
It’s a storage, if db pod get restarted, the data will be restarted as well, but with volumes this won't happen. 

<details><summary>Types of volumes</summary><br>

#### Persistent volume
- doesn't depends on the pod lifecycle
- available to all nodes
- must survive if all cluster crash
- Needs actual storage
  - Can come from any source
  - k8s doesn't create or backup the volume
- Cannot be namespaced

Admin role creates `persistent volume` and users create `persistent volume claim`

##### Persistent volume claim
Application needs to claim the pv so it can use them. Another k8s component.

Defines the size of the volume and access type and whatever volume matches this criteria, will be used.
In the pod you have to use that claim in the pod specification. Then the volume is mounted in the pod.

##### Storage Class
Provides `persistent volume` dynamically whenever `persistent volume claim` claims it.
- Cannot be namespaced

#### ConfigMap and Secrets
You can mount them into the pod

<br></details>

### Deployment
A blueprint for pods, which specifies how many replica of a ped is needed.

- Database cannot have deployment and have replicas because of data inconsistencies. 

### StatefulSet
- Deployment for database/stateful apps.
- It is difficult to use this component.
- Database are often hosted outside k8s. 

<details><summary>More information</summary>

- Cannot be created/deleted at the same time
- Cannot be randomly addressed
- Replica pods are not identical, they have their own additional identity

#### Pod Identity 
- Sticky id for each pod
- Created from the same spec but not interchangeable
- Each has a persistent id that maintains across rescheduling

Why it needs id? 
https://youtu.be/X48VuDVv0do?t=11054

</details>

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

<details><summary>kubectl commands</summary>

- [cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

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

# rollbacks
kubectl rollout history deployment/name
kubectl rollout undo    deployment/name
kubectl rollout undo    deployment/name --to-revision=3
```
</details>

### Config file

<details><summary>Deployment</summary>

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

</details>
<details><summary>Internal Service</summary>

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

</details>

#### label and selection explanation of example above:
- (3): A pod has a label `app: nginx` (it can be any other key-value pair), 
- (2): Create a connection between deployment (2) and pod (3) 
- (1): This is the deployment label and will be used by the service selector (4) 
- (4): connect to the deployment (1) also to the pod (3)  

<details><summary>Secret</summary>

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

</details>
<details><summary>External Service</summary>

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

</details>

## [01 - Example](k8s-examples/01-01-example/README.md)

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

### [Example Ingress](k8s-examples/01-02-example-ingress/README.md)

### Ingress default backend
When page is not found will call it.

## Others
<details><summary>Minikube</summary>

- To test k8s on local machine 
- A node cluster run on virtual box
- In minikube service running is not working and showing pending
  - `minikube service [service name]` to make it work

</details>
<details><summary>Namespace</summary>

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

</details>
<details><summary>Helm</summary>

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

</details>
<details><summary>Init containers</summary>

- Runs before app containers
- Run to completions
- Needs to complete successfully before next init container 
- Can have multiple init containers before app contaienrs runs
- Can contains setup scripts annd they can run before app container runs

```yml
kind: Deployment
...
  template:
    spec:
      initContainers:
        - name: init-db
          image: busybox:1.31
          command: ['sh', '-c', 'echo -e "Checking for the availability of MySQL Server deployment"; while ! nc -z mysql 3306; do sleep 1; printf "-"; done; echo -e "  >> MySQL DB Server has started";']
```

</details>
<details><summary>Probs</summary>

### Liveness probs

Restarts the app, if forexample a deadlock appears in the app.

### Readiness probs

- When pod can accept the traffic.
- If the pod is not ready it is removed from service load balancer.

### Startup probs

- To know when a contianer app has started.
- It disables liveness and readiness probes until startup probs is succeeded

### Options to create probs

- Using shell scripts
- Using http get request, e.g. GET health-status
- Using plain tcp socket connection 

### Examples:

Liveness Probe with Command:
```yml
kind: Deployment
...
containers:
  - ...
  livenessProbe:
    exec:
      command:
        - /bin/sh
        - -c
        - nc -z localhost 8095
    # The pod won't show ready until this time is passed
    initialDelaySeconds: 60
    # How often should call this prob
    periodSeconds: 10
```

Readiness Probe with HTTP GET:
```yml
readinessProbe:
  httpGet:
    path: /usermgmt/health-status
    port: 8095
  initialDelaySeconds: 60
  periodSeconds: 10     
```

</details>
<details><summary>Requests and limits</summary>

- Limits: as the `maximum` amount of a resource to be used by a container.
- Requests, on the other hand, are the `minimum` guaranteed amount of a resource that is reserved for a container
- These can be requested per namespaces as well with [LimitRange](https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/05-Kubernetes-Important-Concepts-for-Application-Deployments/05-05-Kubernetes-Namespaces/05-05-02-Namespaces-LimitRange-default) or [ResourceQuota](https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/05-Kubernetes-Important-Concepts-for-Application-Deployments/05-05-Kubernetes-Namespaces/05-05-03-Namespaces-ResourceQuota)
- It is a good practice to limit both cpu and memory

```yml
resources:
  requests:
    memory: "128Mi"
    cpu: "500m"
  limits:
    memory: "500Mi"
    cpu: "1000m"
```
</details>

# Future research
- Why is good practice to limit resources?
- What is Kustomize

