# 12 - Infrastructure as Code with Terraform

Terraform tool for infrastructure provisioning. declarative & open source

## 1 - Introduction to Terraform

**Terraform & Ansible**

- Both Infrastructure as code (IaC)
- Both automate (provisioning, configuring & managing the infrastructure)
- Terraform is mainly (better) infrastructure provisioning tool
- Ansible is mainly (better) configuration tool (configure, deploy apps, install & update software)

**Terraform Architecture**

2 main components

1) Terraform Core: takes 2 input sources (TF-Config & State), perform Create, Update, Destroy on infrastructure to match config & state 

2) Terraform Providers: providers for different Technologies
  1) AWS, AZure (IaaS)
  2) Kubernetes (PaaS)
  3) Fastly (SaaS)

Through providers you get access to resources. Terraform has over 100 providers to over 1000 resources.

Core create execution plan based on config & state then use providers to execute the plan

**Sample config file**

```hcl
# 1. Specify the AWS Provider
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# 2. Configure the AWS provider
provider "aws" {
  region = "us-east-1" # Change to your preferred target region
}

# 3. Create S3 Bucket for AWS Config Logs
resource "aws_s3_bucket" "config_bucket" {
  bucket = "my-unique-aws-config-bucket-2026" # Must be globally unique
  force_destroy = true
}
```

**Terraform Commands**

- **`refresh`**: query infrastructure provider to get current state
- **`plan`**: create an execution plan
- **`apply`**: execute the plan
- **`destroy`**: destroy the resources/infrastructure

## 2 - Install Terraform & Setup Terraform Project

The official Terraform installation documentation can be found here: https://developer.hashicorp.com/terraform/downloads

```sh
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

## 3 - Providers in Terraform

create "terraform/main.tf" file with following

```terraform
provider "aws" {
  region = "eu-central-1"
  access_key = ""
  secret_key = ""
}
```

providers needs to be installed. you can either add them in "terraform/main.tf" or create "terraform/versions.tf" file with following

```tf
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 6.0"
    }
    linode = {
      source = "linode/linode"
      version = "3.1.1"
    }
  }
}
```

run `terraform init` to install the providers

`terraform init` will create the following in the current directory

- **.terraform**: directory containing the installed providers
- **.terraform.lock.hcl**: file contains the version of the installed providers

official terraform providers documentation can be found here: https://registry.terraform.io/browse/providers

official providers can be used without mentioning them in "required_providers" block, but non official providers needs to be mentioned in "required_providers" block

## 4 - Resources & Data Sources

providers give access to resources & data sources. Resources are used to create infrastructure, while data sources are used to query existing infrastructure.

resource names can be found in the official terraform providers documentation here: https://registry.terraform.io/browse/providers

resource names follow format: `<provider>_<resource_type>` for example: `aws_s3_bucket` is a resource type for AWS S3 Bucket

```tf
provider "aws" {
  region = "eu-central-1"
  access_key = ""
  secret_key = ""
}

# development-vpc is resource name, aws_vpc is resource type
resource "aws_vpc" "development-vpc" {
  cidr_block = "10.0.0.0/16"
}

# aws_vpc.development-vpc.id is a reference to the id of the development-vpc resource created above
resource "aws_subnet" "dev-subnet-1" {
  vpc_id     = aws_vpc.development-vpc.id
  cidr_block = "10.0.10.0/24"
  availability_zone = "eu-central-1a"
}
```

navigate to the "terraform" directory and run `terraform init` to initialize the terraform project, then run `terraform plan` to see the execution plan, and finally run `terraform apply` to create the resources defined in the configuration files.

Data Sources allow data to be fetched for use in TF configuration. Data sources are read-only and cannot be used to create or modify infrastructure.

```tf
provider "aws" {
  region = "eu-central-1"
  access_key = ""
  secret_key = ""
}

# development-vpc is resource name, aws_vpc is resource type
resource "aws_vpc" "development-vpc" {
  cidr_block = "10.0.0.0/16"
}

# aws_vpc.development-vpc.id is a reference to the id of the development-vpc resource created above
resource "aws_subnet" "dev-subnet-1" {
  vpc_id     = aws_vpc.development-vpc.id
  cidr_block = "10.0.10.0/24"
  availability_zone = "eu-central-1a"
}

data "aws_vpc" "existing-vpc" {
  default = true
}

resource "aws_subnet" "dev-subnet-2" {
  vpc_id     = data.aws_vpc.existing-vpc.id
  cidr_block = "172.31.48.0/20"
  availability_zone = "eu-central-1a"
}
```

run `terraform apply` to create the resources defined in the configuration files. The `dev-subnet-2` resource will be created in the default VPC of the AWS account, as specified by the data source `aws_vpc.existing-vpc`.

AWS user needs to have the required permissions to create the resources defined in the configuration files

terraform is **idempotent**, meaning that running `terraform apply` multiple times will not create duplicate resources. It will only create or modify resources as necessary to match the desired state defined in the configuration files.

## 5 - Change & Destroy Terraform Resources

To change the configuration of a resource, simply modify the configuration file and run `terraform apply` to check the plan & apply the changes.

```tf
provider "aws" {
  region = "eu-central-1"
  access_key = ""
  secret_key = ""
}

# development-vpc is resource name, aws_vpc is resource type
resource "aws_vpc" "development-vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name: "development",
    vpc_env: "dev"
  }
}

# aws_vpc.development-vpc.id is a reference to the id of the development-vpc resource created above
resource "aws_subnet" "dev-subnet-1" {
  vpc_id     = aws_vpc.development-vpc.id
  cidr_block = "10.0.10.0/24"
  availability_zone = "eu-central-1a"
  tags = {
    Name: "subnet-1-dev"
  }
}

data "aws_vpc" "existing-vpc" {
  default = true
}

resource "aws_subnet" "dev-subnet-2" {
  vpc_id     = data.aws_vpc.existing-vpc.id
  cidr_block = "172.31.48.0/20"
  availability_zone = "eu-central-1a"
  tags = {
    Name: "subnet-2-default"
  }
}
```

To destroy the resources created by Terraform, either remove the resource from configuration & run `terraform apply` or run `terraform destroy`.

```sh
# This will remove all resources defined in the configuration files.
terraform destroy

# remove single resource 
terraform destroy -target=aws_subnet.dev-subnet-2
```

## 6 - Terraform commands

```sh
# Initialize the Terraform project
terraform init

# Show the execution plan or check difference between the current state and the desired state
terraform plan

# Apply the changes required to reach the desired state
terraform apply

# apply changes without confirmation prompt
terraform apply -auto-approve

# Destroy all resources, terraform will figure out the order of destruction
terraform destroy
```

## 7 - Terraform State

when you run `terraform apply`, terraform creates 2 json files in the current directory

- **terraform.tfstate**: This file contains the current state of the infrastructure managed by terraform.
- **terraform.tfstate.backup**: This file contains the previous state of the infrastructure managed by terraform.

We can either read the json files directly or use terraform commands to read the state of the infrastructure.

```sh
# show the current state of the infrastructure & check sub-commands
terraform state

# list all resources in the current state
terraform state list

# show the current state of a specific resource or check resource attributes instead of using aws console
terraform state show dev-subnet-1
```

## 8 - Output Values

terraform output values are used to extract information from the state file and display it to the user. Output values can be defined in the configuration files and can be used to display information about the resources created by terraform.

```tf
output "vpc_id" {
  value = aws_vpc.development-vpc.id
}
output "subnet_1_id" {
  value = aws_subnet.dev-subnet-1.id
}
```

## 9 - Variables in Terraform

terraform variables are used to parameterize the configuration files and make them more flexible. Variables can be defined in the configuration files and can be used to pass values to the resources created by terraform.

```tf
provider "aws" {
  region = "eu-central-1"
  access_key = ""
  secret_key = ""
}

# define a variable for the subnet CIDR block
variable "subnets_cidr_block" {
  description = "The CIDR block for the subnet"
  type        = list(string)
  default     = ["10.0.10.0/24", "10.0.20.0/24"]
}

# define a variable for vpc CIDR block
variable "vpc_cidr_block" {
  description = "The CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

# development-vpc is resource name, aws_vpc is resource type
resource "aws_vpc" "development-vpc" {
  cidr_block = var.vpc_cidr_block
}

# aws_vpc.development-vpc.id is a reference to the id of the development-vpc resource created above
resource "aws_subnet" "dev-subnet-1" {
  vpc_id     = aws_vpc.development-vpc.id
  cidr_block = var.subnets_cidr_block[0]
  availability_zone = "eu-central-1a"
}
```

There are 3 ways to assign values to variables in terraform:

- **`terraform apply`**: terraform will prompt the user to enter values for the variables.
- **`terraform apply -var "subnets_cidr_block=[\"10.0.10.0/24\", \"10.0.20.0/24\"]"`**: terraform will use the specified value for the variable.
- **`terraform.tfvars`**: terraform will automatically load variable values from this file.

if the file name changed from "terraform.tfvars" to something else, then you can use `-var-file` option to specify the file name.

```sh
# specify the variable file name
terraform apply -var-file="myvars.tfvars"
``` 

Terraform also support object & list of objects as variable types. For example, you can define a variable of type object to represent a complex data structure, and then use that variable in your configuration files.

```tf
variable "subnets" {
  description = "List of subnets to create"
  type = list(object({
    name = string
    cidr_block = string
  }))
  default = [
    {
      name = "subnet-1"
      cidr_block = "10.0.10.0/24"
    }
  ]
}
```

## 10 - Environment Variables in Terraform

Terraform supports environment variables to set provider credentials, backend configuration, and other settings. Environment variables can be used to avoid hardcoding sensitive information in the configuration files.

For AWS provider, remove the "access_key" & "secret_key" from the configuration and set the following environment variables:

```sh
# set AWS access key
export AWS_ACCESS_KEY_ID="your_access_key"

# set AWS secret key
export AWS_SECRET_ACCESS_KEY="your_secret_key"
```

Alternatively, you can use the AWS CLI to configure your credentials and Terraform will automatically use them.

```sh
# configure AWS CLI, terraform will read ~/.aws/credentials file for credentials
aws configure
```

Check provider documentation for supported environment variables & authentication methods

We can also set own terraform environment variables with prefix "TF_VAR_". For example, to set the value of the variable "availability_zone", you can use the following command:

```sh
export TF_VAR_availability_zone="eu-central-1a"
```

Then in config, create a variable block for "availability_zone" and use it in the resource block.

```tf
variable "availability_zone" {
  description = "The availability zone for the subnet"
  type        = string
  default     = "eu-central-1a"
}

# reference the variable in the resource block
resource "aws_subnet" "dev-subnet-1" {
  vpc_id     = aws_vpc.development-vpc.id
  cidr_block = var.subnets_cidr_block[0]
  availability_zone = var.availability_zone
}
```

## 11 - Create Git Repository for local Terraform Project

## 12 - Automate Provisioning EC2 with Terraform - Part 1

In Terraform, create the following

- vpc
- subnet
- route table
- internet gateway
- security group

## 13 - Automate Provisioning EC2 with Terraform - Part 2

In Terraform, create the following

- data to get the latest Amazon Linux 2 AMI ID
- EC2 instance
- key pair
- output to get the public IP of the EC2 instance

## 14 - Automate Provisioning EC2 with Terraform - Part 3

## 15 - Provisioners in Terraform

Terraform provisioners are used to execute scripts or commands on the resources created by Terraform. Provisioners can be used to configure the resources after they have been created, such as installing software, copying files, or running custom scripts.

**Types of provisioners in Terraform:**

- **`local-exec`**: executes a command on the machine running Terraform
- **`remote-exec`**: executes a command on the remote resource created by Terraform
- **`file`**: copies files from the local machine to the remote resource created by Terraform

**Why Terraform provisioners are not recommended for production use:**

- Provisioners can introduce complexity and make the configuration harder to understand and maintain.
- Provisioners can create dependencies between resources, which can lead to unexpected behavior and make it harder to manage the infrastructure.
- Provisioners can make it harder to test and validate the configuration, as they can introduce side effects that are not easily reproducible.
- idempotency is not guaranteed with provisioners, as they can introduce changes to the resources that are not reflected in the Terraform state.

Terraform recommends using provisioners only as a **last resort**, and to use other methods such as configuration management tools **(e.g., Ansible, Chef, Puppet) or cloud-init scripts** to configure the resources after they have been created.

**Local Provider vs Local-Exec Provisioner**

- **Local Provider**: A Terraform provider that interacts with local resources on the machine running Terraform. It is used to manage local files, execute local commands, and interact with local system resources.
- **Local-Exec Provisioner**: A Terraform provisioner that executes a command on the machine running Terraform. It is used to run scripts or commands locally as part of the resource creation or modification process.

**Terraform Provisioner Failure:**

- If a provisioner fails, Terraform will stop the execution and mark the resource as tainted.
- You can use the `-ignore-errors` flag with the provisioner to continue execution even if the provisioner fails.
- You can also use the `when` argument to control when the provisioner runs, e.g., `when = create` or `when = destroy`.

```tf
resource "aws_instance" "myapp-server" {
  ami = data.aws_ami.latest-amazon-linux-image.id
  instance_type = var.instance_type

  subnet_id = aws_subnet.myapp-subnet-1.id
  vpc_security_group_ids = [aws_default_security_group.default-sg.id]
  availability_zone = var.avail_zone

  associate_public_ip_address = true
  key_name = aws_key_pair.ssh-key.key_name

  # user_data = file("entry-script.sh")

  user_data_replace_on_change = true

  # establish connection to the EC2 instance using SSH and run the provisioners
  connection {
    type = "ssh"
    host = self.public_ip
    user = "ec2-user"
    private_key = file(var.private_key_location)
  }

  # copy the entry-script.sh file from local machine to the EC2 instance
  provisioner "file" {
    source = "entry-script.sh"
    destination = "/home/ec2-user/entry-script-on-ec2.sh"
  }

  # run the entry-script-on-ec2.sh script on the EC2 instance
  provisioner "remote-exec" {
    inline = ["/home/ec2-user/entry-script-on-ec2.sh"]
  }

  # run a command on the local machine to save the public IP of the EC2 instance to a file
  provisioner "local-exec" {
    command = "echo ${self.public_ip} > output.txt"
  }

  tags = {
    Name: "${var.env_prefix}-server"
  }
}
```

## 16 - Modules in Terraform - Part 1

Terraform modules are a way to organize and reuse Terraform code. A module is a container for multiple resources that are used together. Modules can be used to create reusable components, such as VPCs, subnets, security groups, and EC2 instances.

You can create your own modules or use existing modules from the Terraform Registry. Modules can be used to create a consistent and repeatable infrastructure across multiple environments.

## 17 - Modules in Terraform - Part 2

Each module has its own directory and can contain multiple Terraform configuration files. The main configuration file for a module is `main.tf`, but you can also include other files such as `variables.tf`, `outputs.tf`, and `providers.tf`.

Create a directory structure for the module as follows:

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
└── modules/
    └── subnet/
        ├── main.tf
        ├── variables.tf
        ├── providers.tf
        └── outputs.tf
```

The root module can then call the subnet module as follows:

```tf
module "myapp-subnet" {
  source = "./modules/subnet"
  vpc_id = aws_vpc.development-vpc.id
  cidr_block = var.subnets_cidr_block[0]
  availability_zone = var.availability_zone
}
```

If the "subnet" module exported the subnet object in its "output.tf" file as below:

```tf
output "subnet" {
  value = aws_subnet.myapp-subnet-1
}
```

Then the root module can reference the module output using the following syntax:

```tf
output "subnet_1_id" {
  value = module.myapp-subnet.subnet.id
}
```

## 18 - Modules in Terraform - Part 3

Create a second module "webserver" to create an EC2 instance and call it from the root module as follows:

```tf
module "myapp-webserver" {
  source = "./modules/webserver"
  ami = data.aws_ami.latest-amazon-linux-image.id
  instance_type = var.instance_type
  subnet_id = module.myapp-subnet.subnet.id
  availability_zone = var.availability_zone
  private_key_location = var.private_key_location
}
```

## 19 - Automate Provisioning EKS cluster with Terraform - Part 1

To create an EKS cluster on AWS using Terraform, first we need to create a VPC, subnets, and security groups for the EKS cluster. The Terraform module "terraform-aws-modules/vpc/aws" can be used to create the VPC and subnets.

```tf
provider "aws" {
  region = "eu-central-1"
}

variable vpc_cidr_block {}
variable private_subnet_cidr_blocks {}
variable public_subnet_cidr_blocks {}

data "aws_availability_zones" "azs" {}

module "myapp-vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "6.0.1"

  name = "myapp-vpc"
  cidr = var.vpc_cidr_block
  private_subnets = var.private_subnet_cidr_blocks
  public_subnets = var.public_subnet_cidr_blocks
  azs = data.aws_availability_zones.azs.names

  enable_nat_gateway = true
  single_nat_gateway = true
  enable_dns_hostnames = true

  tags = {
    "kubernetes.io/cluster/myapp-eks-cluster" = "shared"
  }

  public_subnet_tags = {
    "kubernetes.io/cluster/myapp-eks-cluster" = "shared"
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/myapp-eks-cluster" = "shared"
    "kubernetes.io/role/internal-elb" = 1
  }
}
```

## 20 - Automate Provisioning EKS cluster with Terraform - Part 2

To create the EKS cluster, we can use the Terraform module "terraform-aws-modules/eks/aws". This module will create the EKS cluster and the necessary IAM roles and policies.

```tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "21.1.3"

  name = "myapp-eks-cluster"
  kubernetes_version = "1.33"
  endpoint_public_access  = true

  subnet_ids = module.myapp-vpc.private_subnets
  vpc_id = module.myapp-vpc.vpc_id

  enable_cluster_creator_admin_permissions = true  

  addons = {
    coredns                = {}
    eks-pod-identity-agent = {
      before_compute = true
    }
    kube-proxy             = {}
    vpc-cni                = {
      before_compute = true
    }
  }
  
  tags = {
    environment = "development"
    application = "myapp"
  }

  eks_managed_node_groups = {
    dev = {
    instance_types = ["t2.small"]
    ami_type       = "AL2023_x86_64_STANDARD"
    min_size       = 1
    max_size       = 3
    desired_size   = 3
    }
  }
}
```

## 21 - Automate Provisioning EKS cluster with Terraform - Part 3

Apply the terraform configuration to create the EKS cluster and the managed node group.

```sh
# Initialize the Terraform project
terraform init
# Show the execution plan
terraform plan
# Apply the changes required to reach the desired state
terraform apply -auto-approve
```

Once, created inspect the resources in AWS console, then connect to the cluster with the following commands

```sh
# Update kubeconfig to connect to the EKS cluster
aws eks update-kubeconfig --name myapp-eks-cluster --region eu-central-1
# Check the nodes in the EKS cluster
kubectl get nodes
```

Above requires AWS CLI, Kubectl, and AWS IAM Authenticator to be installed on the local machine. The official documentation for installing these tools can be found here:
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [AWS IAM Authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)

Once connected to the EKS cluster, you can deploy applications to the cluster using Kubernetes manifests or Helm charts. check "./kubernetes/nginx-config.yaml"

To clean up the resources created by Terraform, run the following command:

```sh
# Destroy all resources created by Terraform
terraform destroy -auto-approve

# to check if all resources are destroyed
terraform state list
```

## 22 - Complete CI/CD with Terraform - Part 1

## 23 - Complete CI/CD with Terraform - Part 2

The installation documentation for Terraform can be found here: https://developer.hashicorp.com/terraform/downloads

The project files used in this lecture can be found here:

Terraform Learn: https://gitlab.com/twn-devops-bootcamp/latest/12-terraform/terraform-learn/-/tree/feature/eks
Java-maven-app: https://gitlab.com/twn-devops-bootcamp/latest/12-terraform/java-maven-app

The installation documentation for Docker Compose standalone can be found here: https://docs.docker.com/compose/install/standalone/

steps to setup the project in Jenkins:

- create ssh key-pair
- install TF inside jenkins container
- configure TF to provision server
- adjust jenkinsfile

Create a droplet in digital ocean with Ubuntu 22.04 LTS, then SSH into the droplet and install docker & docker-compose

```bash
sudo apt update
sudo apt install docker.io -y
docker -v
```

Run jenkins as a docker container in the droplet with the following command:

```bash
docker run -d -p 8080:8080 -p 50000:50000 --name jenkins -v jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkins/jenkins
```

Install docker inside the jenkins container to run docker commands from jenkins jobs

```bash
# -u 0 to execute bash as root user
docker exec -u 0 -it jenkins bash

# fetch & install docker 
curl https://get.docker.com/ > dockerinstall && chmod 777 dockerinstall && ./dockerinstall
docker -v
```

Navigate to the browser "http://x.x.x.x:8080" and start jenkins, login with default password

```bash
docker volume ls

# to get volume details & mount point
docker volume inspect jenkins_home

# print the default password for login, username is admin
cat /var/lib/docker/volumes/jenkins_home/_data/secrets/initialAdminPassword
```

Install suggested plugins in jenkins:

- Under "Settings" -> "Plugins" -> "available plugins", install "stage view" plugin (useful to see the progress of each pipeline stage)
- Under "Settings" -> "Tools" we can configure maven, gradle & nodejs as build tools to be available during run time in the pipeline

Create key-pair in AWS console "myapp-key-pair", download the private key and save it in the "terraform" directory as "myapp-key.pem". 

In Jenkins, install "ssh agent" plugin & create multibranch pipeline job & add credentials "server-ssh-key" of type "SSH with Username & Private Key" with the private key "myapp-key-pair" and username "ec2-user".

SSH into jenkins container as root user to install terraform.

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | tee /etc/apt/sources.list.d/hashicorp.list
apt update && apt install terraform
terraform -version
```

Once the AWS server is provisioned, install docker-composer in the server via terraform 

```bash
sudo curl -SL "https://github.com/docker/compose/releases/download/v5.5.0/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose version
```

In Jenkins, Create credentials of type "secret text" for "jenkins_aws_access_key_id" and "jenkins-aws_secret_access_key" with the respective values from your AWS IAM user.

## 24 - Complete CI/CD with Terraform - Part 3

Modify the security group in terraform to allow inbound traffic on port 22 for SSH access from jenkins IP address.

We've to execute `docker login` on remote server in "deply" stage of the pipeline to pull the docker image from docker hub. check "server-cmds.sh" file for the commands to be executed on the remote server.

