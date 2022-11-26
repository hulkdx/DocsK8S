# Terraform

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
