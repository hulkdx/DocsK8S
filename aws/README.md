

- Aws manages master nodes
- Install all the apps on them
- Scale / backup when needed

## Core objects

### EKS Control Plane

It's the master node, managed by aws

### Worker nodes & node groups

Group of EC2 instances.

EC2 instances needs to have the same instance type, ami and IAM role.

### Fargate Profiles (serverless)

Instead of EC2, we run our apps on serverless.

- It provides on-demand, right size compute capacity for containers

## eksctl

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
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access 
```
