# tets

#-----------------------------------------------
# Security Group for Private Node
#-----------------------------------------------
resource "aws_security_group" "private_node_sg" {
  name        = "${var.name}-sg"
  description = "Security group for EKS private node"
  vpc_id      = data.aws_vpc.existing.id

  # Allow SSH only from jump host
  ingress {
    description     = "SSH from Jump Host"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [data.aws_security_group.jump_host_sg.id]
  }

  # Allow all outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.tags, {
    Name = "${var.name}-sg"
  })
}

#-----------------------------------------------
# IAM Role for Private Node (to access EKS)
#-----------------------------------------------
resource "aws_iam_role" "private_node_role" {
  name = "${var.name}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })

  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  role       = aws_iam_role.private_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}

resource "aws_iam_role_policy_attachment" "ssm_policy" {
  role       = aws_iam_role.private_node_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "private_node_profile" {
  name = "${var.name}-profile"
  role = aws_iam_role.private_node_role.name
}

#-----------------------------------------------
# EC2 Private Node (Official Module)
#-----------------------------------------------
module "private_node" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "~> 6.4"

  name          = var.name
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  key_name      = var.key_name
  monitoring    = true

  subnet_id              = data.aws_subnet.private.id
  vpc_security_group_ids = [aws_security_group.private_node_sg.id]

  iam_instance_profile = aws_iam_instance_profile.private_node_profile.name

  # Install kubectl and AWS CLI on boot
  user_data = <<-EOF
    #!/bin/bash
    yum update -y

    # Install kubectl
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

    # Install AWS CLI v2
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    ./aws/install

    # Configure kubeconfig for EKS
    aws eks update-kubeconfig --name ${var.eks_cluster_name} --region ${var.aws_region}
  EOF

  tags = merge(var.tags, {
    Name = var.name
  })
}




# Fetch existing VPC
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = [var.vpc_name]
  }
}

# Fetch existing private subnet
data "aws_subnet" "private" {
  filter {
    name   = "tag:Name"
    values = [var.private_subnet_name]
  }
}

# Fetch existing jump host security group
data "aws_security_group" "jump_host_sg" {
  filter {
    name   = "tag:Name"
    values = [var.jump_host_sg_name]
  }
  vpc_id = data.aws_vpc.existing.id
}

# Fetch latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}