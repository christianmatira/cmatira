---
layout: single
title:  Create an EC2 instance with Terraform
date:   2023-09-26 
categories: 
  - Cloud
  - Terraform
  - AWS
tags:
  - Cloud
  - Terraform
  - AWS
toc: true
toc_sticky: true
header: 
  teaser: "/assets/images/terraform_ec2.svg"
---

AWS EC2 provides a flexible and scalable solution for running virtual servers in the cloud. It offers a wide range of instance types, auto-scaling capabilities, and seamless integration with other AWS services, making it a powerful tool for businesses and developers seeking to deploy and manage their applications in a reliable and cost-effective manner.

Terraform is an open-source infrastructure as code (IaC) tool developed by HashiCorp. It allows users to define and provision infrastructure resources across various cloud providers and on-premises environments in a declarative and consistent manner.

**NOTE:** This tutorial assumes that you have a working terraform installation and terraform user with enough credentials in its policies. Please use an IAM user. Never use the root user to create resources in AWS. 
{: .notice--info}

### Simple VPC

Amazon Virtual Private Cloud (VPC) is a service that lets you launch AWS resources in a logically isolated virtual network. We will create our EC2 instance inside a VPC. Create a tfvars file to save the generated keys in a variable file readably by terraform.

```
$ cat tf_vars.auto.tfvars 
access_key = "my_access_key"
secret_key = "my_secret_key"
```

create a simple terraform script to create a VPC. 

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
variable "access_key" {}
variable "secret_key" {}

provider "aws" {
  region     = "ap-northeast-1"
  access_key = var.access_key
  secret_key = var.secret_key
}

# Create a VPC
resource "aws_vpc" "example" {
  cidr_block = "10.0.0.0/16"
}
``` 
Let's break down the script:

The terraform block specifies the required providers for this script. In this case, it specifies that the AWS provider from HashiCorp should be used and expects a version approximately 5.0.

The variable blocks defines two variables: access_key and secret_key. These variables are placeholders that can be provided values when running the Terraform script, allowing for dynamic configuration. In this case, the script assumes that the access and secret keys will be provided as inputs.

The provider block configures the AWS provider and specifies the region (ap-northeast-1) to be used. It also references the access_key and secret_key variables to authenticate with AWS.

The resource block creates an AWS VPC using the aws_vpc resource type. The example identifier is assigned to this resource. The cidr_block parameter specifies the IP address range for the VPC, in this case, "10.0.0.0/16".

Run `terraform init` then run `terraform plan`

```
$ terraform plan 

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_vpc.example will be created
  + resource "aws_vpc" "example" {
      + arn                                  = (known after apply)
      + cidr_block                           = "10.0.0.0/16"
      + default_network_acl_id               = (known after apply)
      + default_route_table_id               = (known after apply)
      + default_security_group_id            = (known after apply)
      + dhcp_options_id                      = (known after apply)
      + enable_dns_hostnames                 = (known after apply)
      + enable_dns_support                   = true
      + enable_network_address_usage_metrics = (known after apply)
      + id                                   = (known after apply)
      + instance_tenancy                     = "default"
      + ipv6_association_id                  = (known after apply)
      + ipv6_cidr_block                      = (known after apply)
      + ipv6_cidr_block_network_border_group = (known after apply)
      + main_route_table_id                  = (known after apply)
      + owner_id                             = (known after apply)
      + tags_all                             = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

```

The output above shows what terraform plans to do. This is good to check before running `terraform apply`. 

Once we know that the script is good, we can run `terraform apply`

We now have successfully created a VPC, we can destroy the resources with `terraform destroy`. We will create the propper terraform file for creating the instance and other necessary resources. 

This is a good check to see if we have a working terraform setup. From here, we will improve the script to create the other necessary components for the EC2

### Complete the VPC

Now that we have a bare VPC working, We need declare the other necessary components of the VPC to create an EC2 instance. Specifically, the AWS internet gateway, public and private subnets.

The internet gateway is used to provide internet access to the VPC. This is attached to the public subnet of the VPC. The internet gateway logically provides the one-to-one NAT on behalf of your EC2 instance.

To read more about AWS the internet gateway, See the following [link](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) for more information.
{: .notice--info}

```
resource "aws_internet_gateway" "wp_gw" {
  vpc_id = aws_vpc.wp.id

  tags = {
    Name = "wp-igw"
  }
}

resource "aws_subnet" "wp_public" {
  vpc_id = aws_vpc.wp.id
  cidr_block = "192.168.1.0/24"
  availability_zone = "ap-northeast-1a"
  map_public_ip_on_launch = "true"

 tags = {
  Name = "public"
 } 
}
resource "aws_subnet" "wp_private" {
  vpc_id = aws_vpc.wp.id
  cidr_block = "192.168.10.0/24"
  availability_zone = "ap-northeast-1a"
  tags = {
    Name = "private"
    }
}

### Create Route table ### 
resource "aws_route_table" "wp_pub_rt" {
 vpc_id = aws_vpc.wp.id

 route {
  cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.wp_gw.id
 }
 
 tags = {
  Name = "pub_rt"
 }
}

## Add route
resource "aws_route_table_association" "wp_pub_rt_a" {
  subnet_id = aws_subnet.wp_public.id
  route_table_id = aws_route_table.wp_pub_rt.id
}
```

The names are renamed in this example to make more sense. The EC2 instance created will be a wordpress instance the resulting instance will later be installed with a wordpress installation. 

### Create an EC2 instance 

Before we can create our instance, we have to define the following:

1. AWS AMI (Amazon machine image) 
2. Instance Type (size)
2. SSH Keys
3. Security Group

#### AWS AMI

To get information needed for this data source, go to the EC2 page of the aws console > Images > AMI catalog. Search for the Image you want to use and copy the id (ami-*). 

Once that we have this information, we can go to AMIs page then search for the id under public images. The AMI name column has the information you need to filter for the AMI image. 

The resulting data source in terraform will have the following. 

```
data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-20230517"]
  }
}
```

#### EC2 instance type. 

This defines the CPU, Memory, and Storage of the instance we are creating. In this case, we will use t2.micro as it is covered by the free tier

```
data "aws_ec2_instance_type" "vm_size" {
  instance_type = "t2.micro"
}
```

#### SSH keys

Generate a new key with ssh-keygen or use the existing key that you have locally. 

Once generate, we add 2 variables to our tfvars file public_key and key_name

```
$ cat tf_vars.auto.tfvars 
access_key = "my_access_key"
secret_key = "my_secret_key"
public_key = "my_public_key"
key_name   = "my_key_name"
```

The public key is the .pub file generated by ssh-keygen. Paste the contents of `cat ~/.ssh/id_rsa.pub` to the public_key variable. The key_name is any name you want to use to reference this key in AWS.

This will create the key pairs into aws using terraform.

```
variable "public_key" {}
variable "key_name" {}

resource "aws_key_pair" "tf_key" {
  key_name   = var.key_name
  public_key = var.public_key
}
```

#### Security group

The following will create the security group for the wordpress instance. It will allow http, ssh, and icmp from all networks. The instance is able not restricted on its egress traffic. 

```
resource "aws_security_group" "tf_wordpress_sg" {
  name        = "tf_wordpress_sg"
  description = "allow http and ssh traffic"
  vpc_id  = aws_vpc.wp.id
  tags = {
    Name = "tf_wordpress_sg"
  }

  ingress {
    description = "allow http"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "allow ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "allow icmp"
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
}
```

### Create EC2 

Finally, we will piece together the parts we created in the early parts of the script. This will create the EC2 instance. 

```
resource "aws_instance" "wordpress" {
  tags = {
    Name = "tf_wordpress"
  }
  ami             = data.aws_ami.ubuntu.id
  instance_type   = data.aws_ec2_instance_type.vm_size.id
  vpc_security_group_ids = [aws_security_group.tf_wordpress_sg.id]
  key_name        = aws_key_pair.tf_key.key_name
  subnet_id = aws_subnet.wp_public.id
}
```

To know public IP upon creation, we can output the instance IP 

```
output "wordpress_ip" {
  value = ["${aws_instance.wordpress.public_ip}"]
}
``` 

### TLDR

With all that effort, The final script will look as follows. This would be a good enough as a starting file and we will expand or tweak the script for our requirement. 

**variable_file**

```
$ cat tf_vars.auto.tfvars 
access_key = "my_access_key"
secret_key = "my_secret_key"
public_key = "my_public_key"
key_name   = "my_key_name"
```

**Terraform file**
```
$ cat 00_infra.tf 
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
variable "access_key" {}
variable "secret_key" {}

provider "aws" {
  region     = "ap-northeast-1"
  access_key = var.access_key
  secret_key = var.secret_key
}

# Create a VPC
resource "aws_vpc" "wp" {
  cidr_block = "192.168.0.0/16"
  tags = {
    Name = "wordpress"
  }
}

resource "aws_internet_gateway" "wp_gw" {
  vpc_id = aws_vpc.wp.id

  tags = {
    Name = "wp-igw"
  }
}

resource "aws_subnet" "wp_public" {
  vpc_id = aws_vpc.wp.id
  cidr_block = "192.168.1.0/24"
  availability_zone = "ap-northeast-1a"
  map_public_ip_on_launch = "true"

 tags = {
  Name = "public"
 }
}
resource "aws_subnet" "wp_private" {
  vpc_id = aws_vpc.wp.id
  cidr_block = "192.168.10.0/24"
  availability_zone = "ap-northeast-1a"
  tags = {
    Name = "private"
    }
}

### Create Route table ###
resource "aws_route_table" "wp_pub_rt" {
 vpc_id = aws_vpc.wp.id

 route {
  cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.wp_gw.id
 }

 tags = {
  Name = "pub_rt"
 }
}

## Add route
resource "aws_route_table_association" "wp_pub_rt_a" {
  subnet_id = aws_subnet.wp_public.id
  route_table_id = aws_route_table.wp_pub_rt.id
}

data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-20230517"]
  }
}

# Set instance type
data "aws_ec2_instance_type" "vm_size" {
  instance_type = "t2.micro"
}

# Define and upload public key
variable "public_key" {}
variable "key_name" {}

resource "aws_key_pair" "tf_key" {
  key_name   = var.key_name
  public_key = var.public_key
}

### Create Security Group ###
resource "aws_security_group" "tf_wordpress_sg" {
  name        = "tf_wordpress_sg"
  description = "allow http and ssh traffic"
  vpc_id  = aws_vpc.wp.id
  tags = {
    Name = "tf_wordpress_sg"
  }

  ingress {
    description = "allow http"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "allow ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "allow icmp"
    from_port   = -1
    to_port     = -1
    protocol    = "icmp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
}

### Create Instance ###

resource "aws_instance" "wordpress" {
  tags = {
    Name = "tf_wordpress"
  }
  ami             = data.aws_ami.ubuntu.id
  instance_type   = data.aws_ec2_instance_type.vm_size.id
  vpc_security_group_ids = [aws_security_group.tf_wordpress_sg.id]
  key_name        = aws_key_pair.tf_key.key_name
  subnet_id = aws_subnet.wp_public.id
}

output "wordpress_ip" {
  value = ["${aws_instance.wordpress.public_ip}"]
}
```

**Raw Output**

```
$ terraform plan
data.aws_ec2_instance_type.vm_size: Reading...
aws_key_pair.tf_key: Refreshing state... [id=workstation_key]
data.aws_ami.ubuntu: Reading...
aws_vpc.wp: Refreshing state... [id=vpc-07df79cba12992f25]
data.aws_ec2_instance_type.vm_size: Read complete after 1s [id=t2.micro]
data.aws_ami.ubuntu: Read complete after 1s [id=ami-0ed99df77a82560e6]
aws_subnet.wp_public: Refreshing state... [id=subnet-0ef2ee49bc8d09dc5]
aws_internet_gateway.wp_gw: Refreshing state... [id=igw-09f7dcd8f3ec7e1f4]
aws_subnet.wp_private: Refreshing state... [id=subnet-0bd6106ba590e06e3]
aws_security_group.tf_wordpress_sg: Refreshing state... [id=sg-06c62bbb800b9f444]
aws_route_table.wp_pub_rt: Refreshing state... [id=rtb-023d45626f4c6a1c1]
aws_route_table_association.wp_pub_rt_a: Refreshing state... [id=rtbassoc-0b4943e832d05b2d7]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated
with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.wordpress will be created
  + resource "aws_instance" "wordpress" {
      + ami                                  = "ami-0ed99df77a82560e6"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + disable_api_stop                     = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + get_password_data                    = false
      + host_id                              = (known after apply)
      + host_resource_group_arn              = (known after apply)
      + iam_instance_profile                 = (known after apply)
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_lifecycle                   = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t2.micro"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = "workstation_key"
      + monitoring                           = (known after apply)
      + outpost_arn                          = (known after apply)
      + password_data                        = (known after apply)
      + placement_group                      = (known after apply)
      + placement_partition_number           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = (known after apply)
      + source_dest_check                    = true
      + spot_instance_request_id             = (known after apply)
      + subnet_id                            = "subnet-0ef2ee49bc8d09dc5"
      + tags                                 = {
          + "Name" = "tf_wordpress"
        }
      + tags_all                             = {
          + "Name" = "tf_wordpress"
        }
      + tenancy                              = (known after apply)
      + user_data                            = (known after apply)
      + user_data_base64                     = (known after apply)
      + user_data_replace_on_change          = false
      + vpc_security_group_ids               = [
          + "sg-06c62bbb800b9f444",
        ]
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + wordpress_ip = [
      + (known after apply),
    ]

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these
actions if you run "terraform apply" now.

$ terraform apply 
data.aws_ec2_instance_type.vm_size: Reading...
aws_key_pair.tf_key: Refreshing state... [id=workstation_key]
data.aws_ami.ubuntu: Reading...
aws_vpc.wp: Refreshing state... [id=vpc-07df79cba12992f25]
data.aws_ec2_instance_type.vm_size: Read complete after 1s [id=t2.micro]
data.aws_ami.ubuntu: Read complete after 1s [id=ami-0ed99df77a82560e6]
aws_internet_gateway.wp_gw: Refreshing state... [id=igw-09f7dcd8f3ec7e1f4]
aws_subnet.wp_private: Refreshing state... [id=subnet-0bd6106ba590e06e3]
aws_subnet.wp_public: Refreshing state... [id=subnet-0ef2ee49bc8d09dc5]
aws_security_group.tf_wordpress_sg: Refreshing state... [id=sg-06c62bbb800b9f444]
aws_route_table.wp_pub_rt: Refreshing state... [id=rtb-023d45626f4c6a1c1]
aws_route_table_association.wp_pub_rt_a: Refreshing state... [id=rtbassoc-0b4943e832d05b2d7]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated
with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_instance.wordpress will be created
  + resource "aws_instance" "wordpress" {
      + ami                                  = "ami-0ed99df77a82560e6"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + disable_api_stop                     = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + get_password_data                    = false
      + host_id                              = (known after apply)
      + host_resource_group_arn              = (known after apply)
      + iam_instance_profile                 = (known after apply)
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_lifecycle                   = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t2.micro"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = "workstation_key"
      + monitoring                           = (known after apply)
      + outpost_arn                          = (known after apply)
      + password_data                        = (known after apply)
      + placement_group                      = (known after apply)
      + placement_partition_number           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = (known after apply)
      + source_dest_check                    = true
      + spot_instance_request_id             = (known after apply)
      + subnet_id                            = "subnet-0ef2ee49bc8d09dc5"
      + tags                                 = {
          + "Name" = "tf_wordpress"
        }
      + tags_all                             = {
          + "Name" = "tf_wordpress"
        }
      + tenancy                              = (known after apply)
      + user_data                            = (known after apply)
      + user_data_base64                     = (known after apply)
      + user_data_replace_on_change          = false
      + vpc_security_group_ids               = [
          + "sg-06c62bbb800b9f444",
        ]
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + wordpress_ip = [
      + (known after apply),
    ]

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_instance.wordpress: Creating...
aws_instance.wordpress: Still creating... [10s elapsed]
aws_instance.wordpress: Still creating... [20s elapsed]
aws_instance.wordpress: Still creating... [30s elapsed]
aws_instance.wordpress: Creation complete after 34s [id=i-083a4612a805e6a02]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

wordpress_ip = [
  "3.112.235.175",
] 
```

Thank you for reading this long post. You should now have a working instance. That you can access via ssh. `ssh ubuntu@<IP>` This is helpful for creating simple environments that we can easily destroy with `terraform destroy` once we are done with the instance. 