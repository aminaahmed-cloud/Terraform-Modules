# Terraform Modules 

## 1. Create a Module

- **Create a Folder Structure**:
  - Create a folder named `modules`.
  - Inside the `modules` folder, create subfolders for each module.
  - For example, create a subfolder named `subnet` for subnet configurations and another subfolder named `webserver` for web server configurations.

- **Define Subnet Module Configuration**:
  - Inside the `subnet` module subfolder, create the following files: `main.tf`, `variables.tf`, and `outputs.tf`.
 

<img src="https://i.imgur.com/AeDwLDQ.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>



### Subnet Module

- **main.tf**:
  ```hcl
  resource "aws_subnet" "myapp-subnet-1" {
    vpc_id            = var.vpc_id
    cidr_block        = var.subnet_cidr_block
    availability_zone = var.avail_zone

    tags = {
      Name = "${var.env_prefix}-subnet-1"
    }
  }

  resource "aws_internet_gateway" "myapp-igw" {
    vpc_id = var.vpc_id

    tags = {
      Name = "${var.env_prefix}-igw"
    }
  }

  resource "aws_default_route_table" "main-rtb" {
    default_route_table_id = var.default_route_table_id

    route {
      cidr_block = "0.0.0.0/0"
      gateway_id = aws_internet_gateway.myapp-igw.id
    }

    tags = {
      Name = "${var.env_prefix}-main-rtb"
    }
  }

- **variables.tf**:

```hcl
variable "subnet_cidr_block" {}
variable "avail_zone" {}
variable "env_prefix" {}
variable "vpc_id" {}
variable "default_route_table_id" {}
```


- **Update Main Configuration:**

- Add module declaration in the main Terraform configuration file (main.tf).

```
module "myapp_subnet" {
  source = "./modules/subnet"
  subnet_cidr_block = var.subnet_cidr_block
  avail_zone = var.avail_zone
  env_prefix = var.env_prefix
  vpc_id = aws_vpc.myapp-vpc.id
  default_route_table_id = aws_vpc.myapp-vpc.default_route_table_id
}
```

<img src="https://i.imgur.com/36W9GYI.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


## 2. Create Webserver Module

- **Create Configuration**:

- Create a new folder named webserver inside the modules directory.
Inside the webserver folder, create main.tf, variables.tf, and outputs.tf files.


### Webserver Module


- **main.tf:**

```

resource "aws_default_security_group" "default-sg" {
  vpc_id = var.vpc_id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "TCP"
    cidr_blocks = [var.my_ip]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "TCP"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    cidr_blocks     = ["0.0.0.0/0"]
    prefix_list_ids = []
  }

  tags = {
    Name = "${var.env_prefix}-default-sg"
  }
}

data "aws_ami" "latest-amazon-linux-image" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = [var.image_name]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_key_pair" "ssh-key" {
  key_name   = "server-key"
  public_key = file(var.public_key_location)
}

resource "aws_instance" "myapp-server" {
  ami                          = data.aws_ami.latest-amazon-linux-image.id
  instance_type                = var.instance_type
  subnet_id                    = var.subnet_id
  vpc_security_group_ids       = [aws_default_security_group.default-sg.id]
  availability_zone            = var.avail_zone
  associate_public_ip_address  = true
  key_name                     = aws_key_pair.ssh-key.key_name
  user_data                    = file("entry-script.sh")
  user_data_replace_on_change = true

  tags = {
    Name = "${var.env_prefix}-server"
  }
}

```

- **variables.tf:**

```
variable "vpc_id" {}
variable "my_ip" {}
variable "env_prefix" {}
variable "image_name" {}
variable "public_key_location" {}
variable "instance_type" {}
variable "subnet_id" {}
variable "default_sg_id" {}
variable "avail_zone" {}
```

- **outputs.tf:**

```
output "instance_type" {
  value = aws_instance.myapp-server
}
```

- **Update Main Configuration:**
  
- Add module declaration in the main Terraform configuration file ('main.tf').

  
```
module "myapp-server" {
  source = "./modules/webserver"
  vpc_id = aws_vpc.myapp-vpc.id
  my_ip = var.my_ip
  env_prefix = var.env_prefix
  image_name = var.image_name
  public_key_location = var.public_key_location
  instance_type = var.instance_type
  subnet_id = module.myapp-subnet.subnet.id
  avail_zone = var.avail_zone
}
```


### 3. Apply Configuration Changes

- **Initialize Terraform:**

Run terraform init in the terminal to initialize Terraform.


**Apply Configuration:**

- Execute terraform apply to apply the Terraform configuration changes.


<img src="https://i.imgur.com/AUCdyC5.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>
