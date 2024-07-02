#

## License

```terraform
provider "aws" {
  region = "" // enter region
  access_key = "" // Enter access key
  secret_key = "" // Enter secret key
}



# RSA key of size 4096 bits
resource "tls_private_key" "rsa_4096" {
    algorithm = "RSA"
    rsa_bits  = 4096
}

# Taking input dynamically
variable "key_name" {}



#Creating key pair for aws.
resource "aws_key_pair" "key_pair" {
    key_name   = var.key_name
    public_key = tls_private_key.rsa_4096.public_key_openssh
}



resource "local_file" "private_key" {
    content = tls_private_key.rsa_4096.private_key_pem
    filename =var.key_name
}



// Create instance
resource "aws_instance" "demo_server" {
  ami           = "ami-0f06718ac552afe18"
  key_name      = "Linux-sec-key"
  instance_type = "t2.micro"

  subnet_id= aws_subnet.demo_subnet.id
  vpc_security_group_ids=[aws_security_group.demo_vpc_sg.id]
}


// Create VPC
resource "aws_vpc" "demo_vpc" {
  cidr_block = "10.10.0.0/16"
}


// Create Subnet
resource "aws_subnet" "demo_subnet" {
  vpc_id     = aws_vpc.demo_vpc.id
  cidr_block = "10.10.1.0/24"

  tags = {
    Name = "demo_subnet"
  }
}


// Create Internet Gateway
resource "aws_internet_gateway" "demo_igw" {
  vpc_id = aws_vpc.demo_vpc.id

  tags = {
    Name = "demo_igw"
  }
}


// Create Route Table
resource "aws_route_table" "demo_rt" {
  vpc_id = aws_vpc.demo_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.demo_igw.id
  }


  tags = {
    Name = "demo_rt"
  }
}


// Associate subnet with route table.
resource "aws_route_table_association" "demo_rt_assocation" {
  subnet_id      = aws_subnet.demo_subnet.id
  route_table_id = aws_route_table.demo_rt.id
}


// Create security group
resource "aws_security_group" "demo_vpc_sg" {
  name        = "demo_vpc_sg"
  description = "Allow TLS inbound traffic and all outbound traffic"
  vpc_id      = aws_vpc.demo_vpc.id

  tags = {
    Name = "demo_vpc_sg"
  }
}


// Part of Security Group for Ingress (Inbound Rule)
resource "aws_vpc_security_group_ingress_rule" "demo_vpc_sg_ipv4" {
  security_group_id = aws_security_group.demo_vpc_sg.id
  cidr_ipv4         = aws_vpc.demo_vpc.cidr_block
  from_port         = 443
  ip_protocol       = "tcp"
  to_port           = 443
}


// Part of Security Group for Egress (Egress Rule)
resource "aws_vpc_security_group_egress_rule" "allow_all_traffic_ipv4" {
  security_group_id = aws_security_group.demo_vpc_sg.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1" # semantically equivalent to all ports
}


```