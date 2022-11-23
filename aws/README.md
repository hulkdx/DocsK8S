# What is EKS

- AWS manages master nodes
- AWS installs all apps in master nodes
- EKS can run:
  - EC2
  - Fargate: serverless containers
  
# How to create EKS

## using eksctl

TODO: review this section if it is `outdated` or not

- a tool to simplify creating eks.
- It's not from aws, but from [weaveworks](https://github.com/weaveworks/eksctl)

### Commands

Create cluster:
```sh
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup              
```

Create & Associate IAM OIDC Provider for our EKS Cluster:
```sh
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve
```

Create node-group:
```sh
eksctl create nodegroup --cluster=eksdemo1 \
                       --region=us-east-1 \
                       --name=eksdemo1-ng-public1 \
                       --node-type=t2.micro \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \ #20gb of size
                       --ssh-access \
                       --ssh-public-key=kube-demo \
                       # managed means aws will take care of patching, auto-upgrading, etc
                       --managed \
                       # Giving some access: 
                       # TODO: what are these access?
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access 
```


## Terraform

TODO: review this section if it is `outdated` or not

### Terraform module

Terraform module is to simplify creating amazon services.

TODO: this example is probably `outdated`
EKS service example: [01-terraform-module-example](01-terraform-module-example)

# Other aws services related to EKS

## VPC

- VPC is your isolated network within a region and it contains all `availability zone`
- Needs a IP range

### Subnets

- Is network inside one of the `availability zone`
- Needs a subset of VPC's IP address range

### Internet gateway / igw
- Allow to communicate between your VPC and the internet.
- Internet accesss can be inbound or outbound
- Even ssh to the instance is not possible without igw

# AWS stateful k8s

## EBS CSI driver

- Manages the lifecycle of the EBS volumes for persistent volume in k8s (create/resize/delete volumes)
- It requires some AMI policy for the nodes
[Docs](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- EBS is limited to an availability zones (not on a region).


## Examples

- [Mysql using EBS csi driver](https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/04-EKS-Storage-with-EBS-ElasticBlockStore/04-02-SC-PVC-ConfigMap-MySQL)

## RDS
Mysql is difficult to implement properly in aws and we can leverage RDS service.

```yml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: ExternalName
  externalName: usermgmtdb.c7hldelt9xfp.us-east-1.rds.amazonaws.com
```

# Autoscaller
## Horizontal Pod Autoscaller
Automatically increase the number of pods

[docs](https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html)

## Vertical Pod Autoscaller
Launch a new pod with increased/decreased cpu/memory

[docs](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)

# Cluster autoscaller

Autoscale the EC2, on these conditions
- Pods fails to run due to insufficient resources
- Pods that are underutilized and the pod can be placed on the other node.

[docs](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html)