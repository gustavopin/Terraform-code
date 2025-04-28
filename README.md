# Documentation

This document provides functionalities for a Terraform code along with its explanation.

## Description of the Original Terraform File

### Provider

The "provider" command connects the program to AWS. The **region = "us-east-1"** argument declares which AWS data center will be used, in this case, the default one located on the US East Coast.

```
provider "aws" {
  region = "us-east-1"
}
```

### Variable

The *variable* command declares the variables to be used. In this project, their names are "projeto" and "candidato".

```
variable "projeto" {
  description = "Project name"
  type        = string
  default     = "nome"
}

variable "candidato" {
  description = "Candidate name"
  type        = string
  default     = "SeuNome"
}
```

Within the variable block, there are 3 arguments:

  - *description*: the purpose of this variable and what is expected in it;
  - *type*: the type of variable (string, number, bool, map, among others). For both variables declared in the code, the *string* type is used, a text variable type;
  - *default*: uses this value if the user does not input anything ("company" for the "projeto" variable and "SeuNome" for the "candidato" variable).

### Resource

Each *resource* block describes one (or more) infrastructure components, such as compute instances, virtual networks, etc. The *resource* block is basically composed of 4 fundamental arguments:

  - The declaration of *resource*;
  - The type of *resource* to be used;
  - The name of this *resource*;
  - And within the braces {}, all the arguments to be used.

```
resource "resource_type" "resource_name" {
    statement1 = "instance"
    statement2 = "bits"
}
```

  - Key for EC2:

```
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}
```

In the provided main.tf file, the first resource generates an encrypted access key named "ec2_key" using the TLS provider and the RSA algorithm (one of the most widely used encryption algorithms in the world) with a size of 2048 bits (described by NITS as secure until at least 2030). Note that this block creates two keys, one public and one private, the latter being exclusively for user access.

 - Registering the key for EC2:

```
resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
```

This way, we can create the key pair within AWS EC2 named "ec2_key_pair". The *key_name* argument ensures that this key pair has a name referring to the **projeto** and **candidato** variables. The *public_key* argument sends the key to AWS so that EC2 recognizes the private key during a connection.

- VPC (Virtual Private Cloud):

```
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16" 
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
```

In this block, we see 5 main parts:

  - The creation of the VPC through the *aws_vpc* command, which makes Terraform use the AWS provider to create the VPC for the user named "aws_main";
  - The definition of the CIDR block using a 16-bit block (approximately 65,000 IP variations, from 10.0.0.0 to 10.0.255.255);
  - With the *enable_dns_support* argument, we enable EC2 to communicate within the VPC using domain names (e.g., servidorexemplo.ec2.internal). Without it, we would have to use the IPs generated from the CIDR block directly (e.g., 10.0.240.237);
  - *enable_dns_hostnames* ensures that the AWS provider uses the public IP (created with the private IP that will only be used internally) and assigns it a "DNS" name: ec2.publicip.compute-1.amazonaws.com. Here, ec2 indicates that we are using an Elastic Compute Cloud (cloud server), *publicip* is also defined during the creation of the IPs within the VPC along with the private IP, *compute-1* refers to the region of the data center used (in this case, *us-east-1*), and *amazonaws.com* is AWS's domain;
  - Finally, we use our previously established variables as *tags* for our VPC, making it more organized.

Thus, we can proceed to the subnet.

- For the *subnet*:

The next step in the code is to generate a subnet, a segment of the VPC to improve management within a single IP. In the main.tf code, the subnet is created with the argument (within a *resource*) called *aws_subnet* and given the name "main_subnet". To identify the VPC for the creation of the subnet, the *vpc_id = aws_vpc.main_vpc.id* argument is used, identifying the "vpc_main". The *cidr_block* is also defined (as default but can be modified) as 10.0.1.0/24 (a 24-bit IP), and the availability zone (AZ / *availability zone*) within the chosen data center region is specified. The last character "a" in "us-east-1a" defines a more specific zone within the region.

```
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
```

Finally, *tags* are also used for the identification and organization of the subnet. These *tags* will be used throughout the *resources* for organization.

- *Gateway* (Internet Gateway/IGW)

This resource enables the VPC to communicate with the internet through the argument *vpc_id = aws_vpc.main_vpc.id* (an argument that identifies the VPC within main.tf). This *gateway* is called "main_igw" and is created using the *aws_internet_gateway* argument.

```
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
```

- *Route*

However, even with a *VPC*, *subnet*, and a *gateway*, our system cannot access the internet. For this, we use the *resource* that creates a *route* (command within our *resource*) named "main_route_table" using the *aws_route_table* argument. The *route* command also has an argument for its ID called *gateway_id* with the command "aws_internet_gateway.main_igw.id", connecting to our *gateway* and also has a CIDR block with all possible IPs (defined by 0.0.0.0/0). This way, our VPC has access to the public internet.

```
resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}
```

- *Table Association*

*aws_route_table_association* is a command that links our previously created subnet with the route table using the arguments *subnet_id* and *route_table_id* (both identifying the previously created resources). This way, our *subnet* knows which route to use for traffic.

```
resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}
```

>[!IMPORTANT]
>Tags are not allowed in *table associations* and will be removed in code modifications.

- Security for EC2

Using the *aws_security_group* command (named *main_sg*), our code introduces a *Firewall* for the EC2, providing security for the information and deciding what/who can enter (rules defined using the *ingress* command) and what/who can exit (rules defined using the *egress* command). This command includes a name that will use the data from our variables "projeto" and "candidato" along with the suffix "-sg" (*security group*), like this: company-SeuNome-sg, and a connection to our VPC through the *vpc_id* argument:

```
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Allow SSH from anywhere and all outbound traffic"
  vpc_id      = aws_vpc.main_vpc.id
```

Within the *security group* block, the ingress and egress permissions are defined:
  - *Ingress*

  This command defines what/who will have access to our EC2, provided through the code block:

  ```
  ingress {
    description      = "Allow SSH from anywhere"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  ```

  Here, it is manually defined who will have access to the EC2 through the *from_port* and *to_port* arguments, which, in our case, are declared as port 22 (restricted access for administrators via Security Shell (SSH)). The *protocol* defines the type of traffic to be used for access (TCP is used for security). Finally, the CIDR blocks for IPV4 ("0.0.0.0/0") and IPV6 ("::/0") are defined.

  - *Egress*

  The *egress* block controls EC2 traffic. In the main.tf file, we have:

  ```
    egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  ```

  In this code, the outbound rules allow any traffic to leave the EC2. The values "0" for *from_port* and *to_port* allow all types of traffic (SSH, HTTP/HTTPS), along with *protocol* being set to "-1", allowing all protocols to be used (TCP, UDP, etc.). Again, the CIDR blocks for IPV4 and IPV6 allow any IP in this communication.

- AMI (Amazon Machine Image)

The following block searches AWS for the most recent AMI (*most_recent = true*) based on established filters (*filter{}*):

```
data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}
```

The first filter looks for AMIs with the name "debian-12-amd64-*", where "*" at the end of this name acts as a wildcard: the AMI must start with "debian-12-amd64-" and the last value can be anything. Example: debian-12-amd64-20250305-1234. Note that Debian 12 is a Linux distribution, with 12 being its version (Bookworm).

The second filter defines the virtualization type as **HVM** (hardware virtualization), which is more modern and uses hardware acceleration for better performance.

- EC2 Creation

This block defines the creation of the EC2 instance as well as its disk configurations:

```
resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
```

  - *ami = data.aws_ami.debian12.id*: defines the AMI to be used for the EC2 instance, based on the previously created debian12 data source;
  - *instance_type = "t2.micro"*: specifies the type of instance to be used, *t2.micro* refers to the "T2" instance type with "micro" size (7 variations from nano to 2xlarge, micro is the second smallest) that uses a scalable processor up to 3.3GHz ([AWS](https://aws.amazon.com/ec2/instance-types/));
  - *subnet_id = aws_subnet.main_subnet.id*: associates the EC2 instance with the *subnet* (main_subnet) created earlier in the VPC;
  - *key_name = aws_key_pair.ec2_key_pair.key_name*: defines the keys for accessing the EC2 instance;
  - *security_groups = [aws_security_group.main_sg.name]*: includes the EC2 instance in the previously created *security group* (*main_sg*);
  - *associate_public_ip_address = true*: ensures a public IP is assigned to the EC2 instance.

For the *root_block_device* (disk):

  - *volume_size = 20*: sets the disk volume size (in gigabytes);
  - *volume_type = "gp2"*: specifies the disk type as "gp2" (general-purpose SSD);
  - *delete_on_termination = true*: allows the generated volume to be deleted when the instance is **terminated**.

*user_data* Block

Executes an update script for the Debian instance:

  - *#!/bin/bash*: indicates the start of the script in bash;
  - *apt-get update -y*: fetches updates for necessary packages;
  - *apt-get upgrade -y*: upgrades the packages.
  - The "EOF" argument marks the beginning and end of a script within Terraform.

### *Outputs*

The *outputs* are blocks to display information about the infrastructure:

```
output "private_key" {
  description = "Private key to access the EC2 instance"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.debian_ec2.public_ip
}
```

In the first *output* (named "private_key"):

  - *value = tls_private_key.ec2_key.private_key_pem*: defines the value of the *output*, in this case, the private key created earlier by the *private_key* resource;
  - *sensitive = true*: marks this output as "sensitive," meaning it is information that should not be publicly displayed and therefore will not appear in the terminal.

For the second *output* ("ec2_public_ip"):
  - *value = aws_instance.debian_ec2.public_ip*: defines the value as the public IP address of the created debian_ec2 instance.

### NGINX

To install and run NGINX after creating the EC2 instance, the following function is executed along with the package update script in the *user_data* subcommand (internal command in the *resource "aws_instance"*):

```
  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install nginx -y
              systemctl enable nginx
              systemctl start nginx
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
```

With this:
  - *apt-get install nginx -y*: installs NGINX;
  - *systemctl enable nginx*: enables NGINX to run;
  - *systemctl start nginx*: starts NGINX.

This way, the infrastructure now includes a more responsive *web server* that can also serve as a *proxy* (acting as an entry barrier for communication with the EC2 instance).

>[!IMPORTANT]
>It is highlighted that: for the use of NGINX, the rules in the *ingress* command will undergo a change: the arguments for the ports will be modified to the value 80, as NGINX communication only occurs through HTTP. As shown below:

```
  ingress {
    description      = "Allows HTTP from anywhere"
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
```

However, this creates a security issue.

### Modifications and Security

Opening the VPC ports for all HTTP traffic is not a good plan, but we can modify the infrastructure to include new *ingress* rules within *main_sg*:

```
  #SSH Ingress (Port 22)
  ingress {
    description = "Port for administrator via SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.ips_members
  }

  #HTTP Ingress (Port 80)
  ingress {
    description = "Port for HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = var.ips_members
  }
```

This way, we have two *ingress* blocks that will allow ports 22 (SSH) and 80 (HTTP) to be used. However, this alone would not make the infrastructure more complex, so we add, within the *cidr_blocks*, a variable called *ips_member*. This variable will store specific IPs of team members so they can access the infrastructure, thus enabling co-working within the team. This variable is defined through the block (added at the beginning of the code):

```
variable "ips_members" {
  description = "List of IPs that can access this infrastructure"
  type        = list(string)
  default     = ["example/32"] # Other IPs can be added later
}
```

The same idea can be applied to *egress*. Allowing any protocol and any IP to access this zone is not secure, as many attacks occur through *egress*. Therefore, the following structure can be applied:

```
# Outbound Rules
egress {
  description = "Traffic for HTTP"
  from_port   = 80   
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"] # Public internet
}

egress {
  description = "Traffic for HTTPS"
  from_port   = 443  # HTTPS
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"] # Public internet
}

egress {
  description = "DNS"
  from_port   = 53
  to_port     = 53
  protocol    = "udp"
  cidr_blocks = ["0.0.0.0/0"]
}

egress {
  description = "Access port for NGINX"
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
  security_groups = [aws_security_group.main_sg.id] #access only within *main_sg*
}

egress {
  description = "Traffic for Debian update servers"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["1.2.3.4/24"] #insert the IP of Debian update servers
}

# Block to deny outbound traffic for any other IP or protocol
egress {
  description = "Block everything else"
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = []
}
```

These new *egress* blocks allow specific control over IPs and who has access to the EC2 instance, making the structure more secure. It is possible to decide which block the user wants to be useful. For example, if DNS is not necessary, add /* code */ to the block to mark it as a comment or delete it entirely.

The *cidr_blocks* can also be configured for something specific if necessary.

If it is necessary to connect this EC2 instance to another server or add it to a *load balancer* in the future, the following output was added:

```
output "security_group_id" {
  description = "ID of the security group (main_sg)"
  value       = aws_security_group.main_sg.id
}
```

This way, we can retrieve the ID of *main_sg* more easily.

### Instructions

Commands to apply the infrastructure:

  - *terraform init*: downloads all *providers* and *plugins*;
  - *terraform validate*: checks if the code is valid;
  - *terraform plan*: shows what will be created or modified;
  - *terraform apply*: applies the changes and creates the infrastructure (confirm with "yes").