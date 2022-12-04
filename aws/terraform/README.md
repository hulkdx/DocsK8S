# Terraform

## VPC
```terraform
resource "aws_vpc" "NAME" {
  cidr_block           = "10.0.0.0/16"
  # EC2 instances receive public dns hostnames
  # It is required by aws to enable dns hostnames
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
### Route table
### Public route table
```terraform
resource "aws_route_table" "NAME" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.NAME.id
  }
}

resource "aws_route_table_association" "NAME" {
  route_table_id = aws_route_table.NAME.id
  subnet_id      = aws_subnet.NAME.id
}
```
### Private route table
```terraform
resource "aws_route_table" "NAME" {
  vpc_id = aws_vpc.main.id
  route {
    nat_gateway_id = aws_nat_gateway.NAME.id
    cidr_block = "0.0.0.0/0"
  }
}

resource "aws_route_table_association" "NAME" {
  route_table_id = aws_route_table.NAME.id
  subnet_id      = aws_subnet.NAME.id
}
```
## EKS
```terraform
resource "aws_eks_cluster" "NAME" {
  name     = var.cluster-name
  role_arn = aws_iam_role.demo-cluster.arn
  version  = "1.24"

  vpc_config {
    security_group_ids = [aws_security_group.demo-cluster.id]
    subnet_ids = module.vpc.public_subnets
  }

  depends_on = [
    aws_iam_role_policy_attachment.demo-cluster-AmazonEKSClusterPolicy,
  ]
}
```
### Role and policy
EKS will call other aws services on your behalf (e.g. autoscaling group) so it needs a role and policy, [more info](https://docs.aws.amazon.com/eks/latest/userguide/security-iam-awsmanpol.html#security-iam-awsmanpol-AmazonEKSClusterPolicy)
```terraform
resource "aws_iam_role" "NAME" {
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "NAME" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.demo.name
}
```
## EKS worker nodes
```terraform
resource "aws_eks_node_group" "NAME" {
  cluster_name    = aws_eks_cluster.demo.name
  node_role_arn   = aws_iam_role.nodes.arn

  subnet_ids = [
    aws_subnet.private-us-east-1a.id,
    aws_subnet.private-us-east-1b.id
  ]

  capacity_type  = "ON_DEMAND"
  instance_types = ["t3.small"]
  
  scaling_config {
    desired_size = 1
    max_size     = 5
    min_size     = 0
  }

  update_config {
    # The maximum number of nodes unavailable at once during a version update. 
    max_unavailable = 1
  }

  depends_on = [
    aws_iam_role_policy_attachment.node_policy_AmazonEKSWorkerNodePolicy,
    aws_iam_role_policy_attachment.node_policy_AmazonEC2ContainerRegistryReadOnly,
    aws_iam_role_policy_attachment.node_policy_AmazonEKS_CNI_Policy,
  ]
}
```
### Role
```terraform
resource "aws_iam_role" "NAME" {
  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
    Version = "2012-10-17"
  })
}
resource "aws_iam_role_policy_attachment" "NAME" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.nodes.name
}
resource "aws_iam_role_policy_attachment" "node_policy_AmazonEC2ContainerRegistryReadOnly" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.main_eks_node_group_role.name
}
resource "aws_iam_role_policy_attachment" "node_policy_AmazonEKS_CNI_Policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.main_eks_node_group_role.name
}
```
### oidc
Create service account for the pods:
```terraform
data "tls_certificate" "eks" {
  url = aws_eks_cluster.demo.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.demo.identity[0].oidc[0].issuer
}
```
#### To test oidc is working
Define policies for accessing `s3:ListAllMyBuckets`:
```terraform
resource "aws_iam_role" "test_oidc" {
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:default:aws-test"
        }
      }
      Effect = "Allow"
      Principal = {
        Federated = "arn:aws:iam::115563711365:oidc-provider/oidc.eks.eu-north-1.amazonaws.com/id/4708E9AA3B9C8348F43DDB013B3A4DCF"
      }
      Sid = ""
    }]
  })
}
resource "aws_iam_policy" "test-policy" {
  name = "test-policy"

  policy = jsonencode({
    Statement = [{
      Action = [
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ]
      Effect   = "Allow"
      Resource = "arn:aws:s3:::*"
    }]
    Version = "2012-10-17"
  })
}
resource "aws_iam_role_policy_attachment" "test_attach" {
  role       = aws_iam_role.test_oidc.name
  policy_arn = aws_iam_policy.test-policy.arn
}
output "oidc_arn" {
  value = aws_iam_openid_connect_provider.eks.arn
}
```
Then deploy
```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  # Name should be the same as defined above
  name: aws-test
  namespace: default
---
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: NAME
spec:
  serviceAccountName: aws-test
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: [ "/bin/bash", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
  tolerations:
  - operator: Exists
    effect: NoSchedule
```
execute command `kubectl exec aws-cli -- aws s3api list-buckets` should work.
### worker node's security group
```terraform
resource "aws_eks_node_group" "NAME" {
  ...
  launch_template {
    name = aws_launch_template.node_group.name
    version = aws_launch_template.node_group.latest_version
  }
}
resource "aws_launch_template" "NAME" {
  vpc_security_group_ids = [
    aws_eks_cluster.NAME.vpc_config[0].cluster_security_group_id,
    aws_security_group.NAME.id,
  ]
}
resource "aws_security_group" "NAME" {
  vpc_id = aws_vpc.NAME.id
}
```

# kubectl
```sh
aws eks --region us-east-1 update-kubeconfig --name demo
```
