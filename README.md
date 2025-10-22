# test4
// provider

provider "aws" {
  region = "us-east-2"
}

// Create VPC

resource "aws_vpc" "ec2-vpc" {
  cidr_block       = "10.10.0.0/16"
  

  tags = {
    Name = "ec2-vpc"
  }
}

// create subnet

resource "aws_subnet" "ec2-subnet" {
  vpc_id     = aws_vpc.ec2-vpc.id
  cidr_block = "10.10.1.0/24"

  tags = {
    Name = "ec2-subnet"
  }
}

// create internet gateway

resource "aws_internet_gateway" "ec2-igw" {
  vpc_id = aws_vpc.ec2-vpc.id

  tags = {
    Name = "ec2-igw"
  }
}

// create route table

resource "aws_route_table" "ec2-rt" {
  vpc_id = aws_vpc.ec2-vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.ec2-igw.id
  }

  tags = {
    Name = "ec2-rt"
  }
}

// associate subnet with routetable

resource "aws_route_table_association" "ec2-route-associateion" {
  subnet_id      = aws_subnet.ec2-subnet.id
  route_table_id = aws_route_table.ec2-rt.id
}

// create security group

resource "aws_security_group" "ec2-sg" {
  name        = "ec2-sg"
  vpc_id      = aws_vpc.ec2-vpc.id

  ingress {
    description      = "ssh"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  
  ingress {
    description      = "apache"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ec2-sg"
  }
}

// To Generate Private Key
resource "tls_private_key" "rsa_4096" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

variable "key_name" {}

// Create Key Pair for Connecting EC2 via SSH
resource "aws_key_pair" "key_pair" {
  key_name   = var.key_name
  public_key = tls_private_key.rsa_4096.public_key_openssh
}

// Save PEM file locally
resource "local_file" "private_key" {
  content  = tls_private_key.rsa_4096.private_key_pem
  filename = var.key_name
}

// create ec2

resource "aws_instance" "ec2-test" {
  ami           = "ami-048e636f368eb3006"
  instance_type = "t2.micro"
  subnet_id = aws_subnet.ec2-subnet.id
  vpc_security_group_ids = [aws_security_group.ec2-sg.id]
  associate_public_ip_address = "true"
  key_name = aws_key_pair.key_pair.key_name
  user_data = "${file("apache.sh")}"
  

  tags = {
    Name = "ec2-test"
  }
}
