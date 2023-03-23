---
# User change
title: "Deploy Redis as a cache for MySQL on Azure Arm based Instance"

weight: 9 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Redis as a cache for MySQL on Azure Arm based Instance

## Prerequisites

* An [Azure portal account](https://azure.microsoft.com/en-in/get-started/azure-portal)
* [Azure CLI](/content/install-tools/azure-cli.md)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](https://developer.hashicorp.com/terraform/cli/install/apt)
* [Python](https://docs.python-guide.org/starting/install3/linux/#install3-linux)

## Install Redis on Azure Arm based Instance

To install Redis follow this [documentation](/content/learning-paths/server-and-cloud/redis/azure_deployment.md).

## Deploy Redis as a cache for MySQL on a single Azure Arm based instance

## Deploy Azure Arm based instance via Terraform

Before deploying Azure Arm based instance via Terraform, generate [Access keys](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#generate-access-keys-access-key-id-and-secret-access-key) and [key-pair using ssh keygen](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#generate-key-pairpublic-key-private-key-using-ssh-keygen).

For Azure Arm based instance deployment, the Terraform configuration is broken into four files: **providers.tf**, **variables.tf**, **main.tf**, and **outputs.tf**.

Add the following code in **providers.tf** file to configure Terraform to communicate with Azure:

```console
terraform {
  required_version = ">=0.12"

  required_providers {
    azurerm = {
      source= "hashicorp/azurerm"
      version = "~>2.0"
    }
    random = {
      source= "hashicorp/random"
      version = "~>3.0"
    }
    tls = {
    source = "hashicorp/tls"
    version = "~>4.0"
    }
  }
}

provider "azurerm" {
  features {}
}
``` 

Create a **variables.tf** file for describing the variables referenced in the other files with their type and a default value.

```console
variable "resource_group_location" {
  default = "eastus2"
  description = "Location of the resource group."
}

variable "resource_group_name_prefix" {
  default = "rg"
  description = "Prefix of the resource group name that's combined with a random ID so name is unique in your Azure subscription."
}
```

Add the resources required to create a virtual machine in **main.tf**.

```console
resource "random_pet" "rg_name" {
  prefix = var.resource_group_name_prefix
}

resource "azurerm_resource_group" "rg" {
  location = var.resource_group_location
  name = random_pet.rg_name.id
}

# Create virtual network
resource "azurerm_virtual_network" "my_terraform_network" {
  name = "myVnet"
  address_space = ["10.0.0.0/16"]
  location = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create subnet
resource "azurerm_subnet" "my_terraform_subnet" {
  name = "mySubnet"
  resource_group_name = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.my_terraform_network.name
  address_prefixes = ["10.0.1.0/24"]
}

# Create Public IPs
resource "azurerm_public_ip" "my_terraform_public_ip" {
  name = "myPublicIP"
  location = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method = "Dynamic"
}

# Create Network Security Group and rule
resource "azurerm_network_security_group" "my_terraform_nsg" {
  name= "myNetworkSecurityGroup"
  location= azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name= "SSH"
    priority= 1001
    direction= "Inbound"
    access = "Allow"
    protocol= "Tcp"
    source_port_range= "*"
    destination_port_range = "22"
    source_address_prefix= "*"
    destination_address_prefix = "*"
  }
  security_rule {
    name= "Redis-port"
    priority= 1002
    direction= "Inbound"
    access = "Allow"
    protocol= "Tcp"
    source_port_range= "*"
    destination_port_range = "6000"
    source_address_prefix= "*"
    destination_address_prefix = "*"
  }
  security_rule {
    name= "MySQL-port"
    priority= 1003
    direction= "Inbound"
    access = "Allow"
    protocol= "Tcp"
    source_port_range= "*"
    destination_port_range = "3306"
    source_address_prefix= "*"
    destination_address_prefix = "*"
  }
}

# Create network interface
resource "azurerm_network_interface" "my_terraform_nic" {
  name= "myNIC"
  location= azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name= "my_nic_configuration"
    subnet_id = azurerm_subnet.my_terraform_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id= azurerm_public_ip.my_terraform_public_ip.id
  }
}

# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
  network_interface_id= azurerm_network_interface.my_terraform_nic.id
  network_security_group_id = azurerm_network_security_group.my_terraform_nsg.id
}

# Generate random text for a unique storage account name
resource "random_id" "random_id" {
  keepers = {
    # Generate a new ID only when a new resource group is defined
    resource_group = azurerm_resource_group.rg.name
  }

  byte_length = 8
}

# Create storage account for boot diagnostics
resource "azurerm_storage_account" "my_storage_account" {
  name = "diag${random_id.random_id.hex}"
  location = azurerm_resource_group.rg.location
  resource_group_name= azurerm_resource_group.rg.name
  account_tier = "Standard"
  account_replication_type = "LRS"
}

# Create virtual machine
resource "azurerm_linux_virtual_machine" "my_terraform_vm" {
  name= "myVM"
  location= azurerm_resource_group.rg.location
  resource_group_name= azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.my_terraform_nic.id]
  size= "Standard_D2ps_v5"

  os_disk {
    name = "myOsDisk"
    caching= "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer = "0001-com-ubuntu-server-focal"
    sku= "20_04-lts-arm64"
    version= "20.04.202209200"
  }

  computer_name= "myvm"
  admin_username= "azureuser"
  disable_password_authentication = true

  admin_ssh_key {
    username= "azureuser"
    public_key = file("/home/ubuntu/.ssh/azure_key.pub")
  }

  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.my_storage_account.primary_blob_endpoint
  }
}

resource "local_file" "inventory" {
    depends_on=[azurerm_linux_virtual_machine.my_terraform_vm]
    filename = "inventory.txt"
    content = <<EOF
[all]
ansible-target1 ansible_connection=ssh ansible_host=${azurerm_linux_virtual_machine.my_terraform_vm.public_ip_address} ansible_user=azureuser
                EOF
}
```
**NOTE:-** Replace the **path** of **public_key** with its respective value.

Add the below code in **outputs.tf** to get **Resource group** name and **Public IP**:

```console
output "resource_group_name" {
  value = azurerm_resource_group.rg.name
}

output "public_ip_address" {
  value = azurerm_linux_virtual_machine.my_terraform_vm.public_ip_address
}
```

## Terraform commands

To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/content/learning-paths/server-and-cloud/redis/ec2_deployment.md#terraform-commands) to deploy the **main.tf** file.

## Install Redis and MySQL via Ansible

To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`. The following Playbook contains a collection of tasks to install MySQL database and Redis.

Here is the complete YML file of Ansible-Playbook
```console
---
- hosts: all
  become: true
  become_user: root
  remote_user: azureuser

  tasks:
    - name: Update the Machine
      shell: apt update -y
    - name: Download redis gpg key
      shell: curl -fsSL "https://packages.redis.io/gpg" | gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
      args:
        warn: false
    - name: Add redis gpg key
      shell: echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" |  tee /etc/apt/sources.list.d/redis.list
    - name: Update the apt sources
      shell: apt update
    - name: Install redis
      shell: apt install -y redis-tools redis
    - name: Start redis server
      shell: redis-server --port 6000 --daemonize yes
      become_user: azureuser
    - name: Set Authentication password
      shell: redis-cli -p 6000 CONFIG SET requirepass "{password}"
      become_user: azureuser
    - name: Installing Mysql-Server
      shell: apt -y install mysql-server
    - name: start and enable mysql service
      service:
        name: mysql
        state: started
        enabled: yes
    - name: Change Root Password
      shell: sudo mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '{{Your_mysql_password}}'"
    - name: Create database user with password and all database privileges and 'WITH GRANT OPTION'
      mysql_user:
         login_user: root
         login_password: {{root_password}}
         login_host: localhost
         name: local_user
         host: '%'
         password: {{local_user_password}}
         priv: '*.*:ALL,GRANT'
         state: present
    - name: Create a new database with name 'arm_test'
      community.mysql.mysql_db:
        name: arm_test
        login_user: local_user
        login_password: {{local_user_password}}
        login_host: localhost
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
    - name: MySQL secure installation
      become: yes
      expect:
        command: mysql_secure_installation
        responses:
           'Enter current password for root': '{{Your_mysql_password}}'
           'Set root password': 'n'
           'Remove anonymous users': 'y'
           'Disallow root login remotely': 'n'
           'Remove test database': 'y'
           'Reload privilege tables now': 'y'
        timeout: 1
      register: secure_mysql
      failed_when: "'... Failed!' in secure_mysql.stdout_lines"
    - name: Enable remote login by changing bind-address
      lineinfile:
         path: /etc/mysql/mysql.conf.d/mysqld.cnf
         regexp: '^bind-address'
         line: 'bind-address = 0.0.0.0'
         backup: yes
      notify:
         - Restart mysql
  handlers:
    - name: Restart mysql
      service:
        name: mysql
        state: restarted
```
**NOTE:-** Replace `{{Your_mysql_password}}` and `{{Give_any_password}}` with your password.

In our case, the inventory file will generate automatically after the `terraform apply` command.

### Ansible Commands
To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}`, `{your_inventory_file}` and `{path_to_private_key}` with your values.

Here is the output after the successful execution of the `ansible-playbook` commands.



## Deploy Redis as a cache for single MySQL database on a single Azure Arm based instance using Python

To deploy Redis as a cache for MySQL using Python follow this [documentation](/content/learning-paths/server-and-cloud/redis/redis_cache_aws.md#deploy-redis-as-a-cache-for-single-mysql-database-on-a-single-aws-arm-based-instance-using-python)

## Deploy Redis as a cache for MySQL on two AWS Arm based instances

### 
After generating the keys, we have to create the MySQL instances. Then we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22` (ssh) and `3306` (MySQL). Below is a Terraform file called **main.tf** that will do this for us. Here we are creating 2 instances.
    
```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
  resource "aws_instance" "MYSQL_TEST" {
  count         = "2"
  ami           = "ami-064593a301006939b"
  instance_type = "t4g.small"
  security_groups= [aws_security_group.Terraformsecurity1.name]
  key_name = "mysql_h"
  tags = {
    Name = "MYSQL_TEST"
  }

resource "aws_default_vpc" "main" {
  tags = {
    Name = "main"
  }
}
resource "aws_security_group" "Terraformsecurity1" {
  name        = "Terraformsecurity1"
  description = "Allow TLS inbound traffic"
  vpc_id      = aws_default_vpc.main.id

  ingress {
    description      = "TLS from VPC"
    from_port        = 3306
    to_port          = 3306
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
}
  ingress {
    description      = "TLS from VPC"
    from_port        = 22
    to_port          = 22
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
    Name = "Terraformsecurity1"
  }

 }
resource "local_file" "inventory" {
    depends_on=[aws_instance.MYSQL_TEST]
    filename = "inventory.txt"
    content = <<EOF
[mysql1]
${aws_instance.MYSQL_TEST[0].public_ip}
[mysql2]
${aws_instance.MYSQL_TEST[1].public_ip}
[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "mysql_h"
        public_key = "ssh-rsaxxxxxxxxxxxxxxx"
} 
    
```
**NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with respective values. The Terraform commands will automatically generate the **inventory.txt** file in the path provided in the filename. Specify the path accordingly.

### Terraform Commands
To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](https://github.com/zachlas/arm-software-developers-ads/blob/main/content/learning-paths/server-and-cloud/mysql/ec2_deployment.md#terraform-commands) to deploy the **main.tf** file.

## Configure MySQL through Ansible
To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`. Playbook contains a collection of tasks.       
Here is the complete YML file for Ansible-Playbook for both instances.      

```console
---
- hosts: mysql1, mysql2
  remote_user: root
  become: true

  tasks:
    - name: Update the Machine
      shell: apt-get update -y
    - name: Installing Mysql-Server
      shell: apt-get -y install mysql-server
    - name: Installing PIP for enabling MySQL Modules
      shell: apt -y install python3-pip
    - name: Installing Mysql dependencies
      shell: pip3 install PyMySQL
    - name: start and enable mysql service
      service:
        name: mysql
        state: started
        enabled: yes
    - name: Change Root Password
      shell: sudo mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '{{Your_mysql_password}}'"
    - name: Create database user with password and all database privileges and 'WITH GRANT OPTION'
      mysql_user:
         login_user: root
         login_password: {{Your_mysql_password}}
         login_host: localhost
         name: Local_user
         host: '%'
         password: {{Give_any_password}}
         priv: '*.*:ALL,GRANT'
         state: present
    - name: Create a new database with name 'arm_test1'
      when: "'mysql1' in group_names"
      community.mysql.mysql_db:
        name: arm_test1
        login_user: root
        login_password: {{Your_mysql_password}}
        login_host: localhost
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock
    - name: Create a new database with name 'arm_test2'
      when: "'mysql2' in group_names"
      community.mysql.mysql_db:
        name: arm_test2
        login_user: root
        login_password: {{Your_mysql_password}}
        login_host: localhost
        state: present
        login_unix_socket: /run/mysqld/mysqld.sock      
    - name: MySQL secure installation
      become: yes
      expect:
        command: mysql_secure_installation
        responses:
           'Enter current password for root': '{{Your_mysql_password}}'
           'Set root password': 'n'
           'Remove anonymous users': 'y'
           'Disallow root login remotely': 'n'
           'Remove test database': 'y'
           'Reload privilege tables now': 'y'
        timeout: 1
      register: secure_mariadb
      failed_when: "'... Failed!' in secure_mariadb.stdout_lines"
    - name: Enable remote login by changing bind-address
      lineinfile:
         path: /etc/mysql/mysql.conf.d/mysqld.cnf
         regexp: '^bind-address'
         line: 'bind-address = 0.0.0.0'
         backup: yes
      notify:
         - Restart mysql
  handlers:
    - name: Restart mysql
      service:
        name: mysql
        state: restarted
```

### Ansible Commands
To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}`, `{your_inventory_file}` and `{path_to_private_key}` with your values.

Here is the output after the successful execution of the `ansible-playbook` commands.

![ansible-end-final](https://user-images.githubusercontent.com/71631645/217766981-3e00b3f6-6ba8-47eb-9c8d-fefcd1685e36.jpg)

## Deploy Redis as a cache for two independent MySQL database on two AWS Arm based instance using Python

To deploy Redis as a cache for two independent MySQL database on two AWS Arm based instance using Python follow this [documentation](/content/learning-paths/server-and-cloud/redis/redis_cache_aws.md#deploy-redis-as-a-cache-for-two-independent-mysql-database-on-two-aws-arm-based-instance-using-python)
