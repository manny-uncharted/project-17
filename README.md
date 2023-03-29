
# AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM PART 2

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Networking](#networking)
    - [Creating our Private Subnets](#creating-our-private-subnets)
    - [Introducing Tagging](#introducing-tagging)
    - [Creating our Internet Gateway](#creating-our-internet-gateway)
    - [Creating our NAT Gateway](#creating-our-nat-gateway)


## Introduction
This is the second part of the series on Infrastructure as Code using Terraform. In this part, we will be creating a VPC, Subnets, Internet Gateway, NAT gateway, Route Table, AutoScaling group, RDS, Security Group and EC2 instance. We will also be using the outputs from the previous part to create the VPC and Subnets.
To get up to speed with the previous part, you can read it [here](https://github.com/manny-uncharted/project-16)

Note: We are building according to this architecture
![Architecture](https://darey.io/wp-content/uploads/2021/07/tooling_project_15.png)

## Prerequisites
- [Terraform](https://www.terraform.io/downloads.html)
- [AWS Account](https://aws.amazon.com/)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)


## Networking
As a continuation to our previous part we would get started by

- Initializing Terraform
    ```bash
    terraform init
    ```

    result
    ![terraform init](img/terraform-init.png)

### Creating our Private Subnets
We would need to create 4 private subnets in our VPC. 
- In the `main.tf` file, we would create a resource for the private subnets
```terraform
resource "aws_subnet" "private" {
  count                   = var.preferred_number_of_private_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index + 2)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}
```
- In our `variables.tf` file, we would add the following variables
```terraform
variable "preferred_number_of_private_subnets" {
  # default = 4
  type        = number
  description = "Number of private subnets to create. If not specified, all available AZs will be used."
}
```

- In our 'terraform.tfvars' file, we would add the following
```terraform
preferred_number_of_private_subnets = 4
```

- We would then run `terraform plan` to see the changes that would be made
```bash
terraform plan
```

result:

![terraform plan](img/terraform-plan-subnet-1.png)
![terraform plan](img/terraform-plan-subnet-2.png)

### Introducing Tagging
We would need to tag our resources to make it easier to identify them. 

- We would need to add the following to our `main.tf` file to the public and private subnets
```terraform
tags = merge(
    var.tags,
    {
      Name = format("%s-Private-subnet-%s", var.name, count.index + 1)
    },
)
```

- In our `variables.tf` file, we would add the following variables
```terraform
variable "tags" {
  type        = map(string)
  description = "A map of tags to add to all resources."
  default     = {}
}
```

Note: The `merge` function is used to merge two maps together. In this case, we are merging the `var.tags` and the `Name` tag. The `var.tags` is a map of tags that we would pass in as a variable. The `Name` tag is the name of the subnet. The 'format' function is used to format the string. In this case, we are formatting the string to include our defined name, the type of subnet and the index of the subnet.

result:
![terraform tag](img/terraform-tag-subnet-1.png)
![terraform tag](img/terraform-tag-subnet-2.png)

- Let's run 'terraform plan' to see the changes that would be made
```bash
terraform plan
```

result:
![terraform plan](img/terraform-plan-tag-subnet-1.png)

### Creating our Internet Gateway
We would need to create an internet gateway to allow our instances to access the internet.
- Create an internet gateway in a separate file called `internet_gateway.tf`
```terraform
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Internet-Gateway", aws_vpc.main.id, "Internet-Gateway")
    },
  )
}
```

result:
![terraform ig](img/terraform-ig.png)

- Run `terraform plan` to see the changes that would be made
```bash
terraform plan
```

result:
![terraform plan](img/terraform-plan-ig.png)

### Creating our NAT Gateways
We would need to create a NAT gateway to allow our instances to access the internet without exposing them to the public internet. For the NAT gateway, we would need an elastic IP address.

- Create an elastic IP address in a separate file called `nat_gateway.tf`
```terraform
resource "aws_eip" "nat_eip" {
  vpc        = true
  depends_on = [aws_internet_gateway.ig]

  tags = merge(
    var.tags,
    {
      Name = format("%s-EIP-%s", var.name, var.environment)
    },
  )
}
```
Note: The `depends_on` is used to ensure that the elastic IP address is created after the internet gateway is created. This ensures that the elastic IP address does not get created before the internet gateway.

result:
![terraform eip](img/terraform-eip.png)

- Let's now move on to creating the NAT gateway
```terraform
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.ig]

  tags = merge(
    var.tags,
    {
      Name = format("%s-Nat-%s", var.name, var.environment)
    },
  )
}
```

Note: Don't forget to declare the variable `environment` in the `variables.tf` file. As this would help us identify the environment we are deploying to.

```terraform
variable "environment" {
  type        = string
  description = "The environment we are deploying to."
  default     = "dev"
}
```

result:
![terraform nat](img/terraform-nat.png)

- Let's run `terraform plan` to see the changes that would be made
```bash
terraform validate
terraform plan
```

Note: The `terraform validate` command is used to validate the syntax of the terraform files. It is a good practice to run this command before running `terraform plan` to ensure that there are no syntax errors in the terraform files.

result:
![terraform plan](img/terraform-plan-nat.png)