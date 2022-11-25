# Documentation
## EKS

- AWS manages master nodes
- AWS installs all apps in master nodes
- EKS can run:
  - EC2
  - Fargate: serverless containers

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
- It is free, [pricing](https://aws.amazon.com/vpc/pricing/)

### NAT gateway
Internet Gateway (IGW) allows instances with public IPs to access the internet. NAT Gateway (NGW) allows instances with no public IPs to access the internet.
- It is not free


## Load balancer

[comparison](https://aws.amazon.com/elasticloadbalancing/features/)

### Classic
The default load balancer is classic load balancer. You can create it using private `node groups` with:
```yml
apiVersion: v1
kind: Service
metadata:
  name: clb-usermgmt-restapp
  labels:
    app: usermgmt-restapp
spec:
  # just this line is required
  type: LoadBalancer
  selector:
    app: usermgmt-restapp     
  ports:
  - port: 80
    targetPort: 8095
```

### NLB
### ALB


## Stateful services

### EBS CSI driver

- Manages the lifecycle of the EBS volumes for persistent volume in k8s (create/resize/delete volumes)
- It requires some AMI policy for the nodes
[Docs](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
- EBS is limited to an availability zones (not on a region).
- Example: [Mysql using EBS csi driver](https://github.com/stacksimplify/aws-eks-kubernetes-masterclass/tree/master/04-EKS-Storage-with-EBS-ElasticBlockStore/04-02-SC-PVC-ConfigMap-MySQL)

### RDS
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

## Autoscaller
### Horizontal Pod Autoscaller
Automatically increase the number of pods

[docs](https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html)

### Vertical Pod Autoscaller
Launch a new pod with increased/decreased cpu/memory

[docs](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)

### Cluster autoscaller

Autoscale the EC2, on these conditions
- Pods fails to run due to insufficient resources
- Pods that are underutilized and the pod can be placed on the other node.

[docs](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html)

# Terraform
## Examples
`outdated?` EKS service example: [01-terraform-module-example](01-terraform-module-example)

## VPC
```terraform
resource "aws_vpc" "NAME" {
  cidr_block           = "10.0.0.0/16"
  # Optional: instances receive public DNS hostnames
  enable_dns_hostnames = "true"
}
```
### Subnet
#### Private subnet
```terraform
resource "aws_subnet" "NAME" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "eu-west-1a"
  tags                    = {
    # Is required by kubernetes to discover subnets where private load balancer will be created
    "kubernetes.io/role/internal-elb" = "1"
    # must tag it with cluster name (ie. demo below)
    # owned if its only used by kubernetes or shared if its shared with other services or other eks cluster
    "kubernetes.io/cluster/demo"      = "owned"
  }
}
```
#### Public subnet
```terraform
resource "aws_subnet" "NAME" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "eu-west-1a"
  map_public_ip_on_launch = true
  tags                    = {
    "kubernetes.io/role/elb"     = "1"
    "kubernetes.io/cluster/demo" = "owned"
  }
}
```
### Internet gateway / igw
```terraform
resource "aws_internet_gateway" "NAME" {
  vpc_id = aws_vpc.main.id
}
```
### NAT gateway
```terraform
# Elastic IP address, static ip address
resource "aws_eip" "nat" {
  vpc = true
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  # Needs to be placed in a public subnet
  subnet_id     = aws_subnet.public-us-east-1a.id
  # To ensure proper ordering, it is recommended to add an explicit dependency on the Internet Gateway for the VPC.
  depends_on = [aws_internet_gateway.igw]
}
```

```
### Route table
### Public route table
```terraform
resource "aws_route_table" "NAME" {
  vpc_id = aws_vpc.main.id

  route {
    # all ip addresses
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.NAME.id
  }
}

resource "aws_route_table_association" "NAME" {
  subnet_id      = aws_subnet.NAME.id
  route_table_id = aws_route_table.NAME.id
}
```
### Private route table
```terraform
resource "aws_route_table" "NAME" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.NAME.id
  }
}

resource "aws_route_table_association" "NAME" {
  subnet_id      = aws_subnet.NAME.id
  route_table_id = aws_route_table.NAME.id
}
```

# eksctl
<details><summary>more info</summary>

- a tool to simplify creating eks.
- It's not from aws, but from [weaveworks](https://github.com/weaveworks/eksctl)

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

                       # If you want to create the node groups in private subnets
                       # --node-private-networking 
```
</details>
</br>