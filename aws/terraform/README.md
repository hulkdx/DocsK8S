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
resource "aws_iam_role" "demo" {
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

resource "aws_iam_role_policy_attachment" "demo-AmazonEKSClusterPolicy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.demo.name
}
```
