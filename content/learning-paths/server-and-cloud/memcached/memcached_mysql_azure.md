---
# User change
title: "Deploy Memcached as a cache for MySQL on an Azure Arm based Instance"

weight: 4 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Memcached as a cache for MySQL on an Azure Arm based Instance

You can deploy Memcached as a cache for MySQL on Azure using Terraform and Ansible. 

In this section, you will deploy Memcached as a cache for MySQL on an Azure instance. 

If you are new to Terraform, you should look at [Automate Azure instance creation using Terraform](/learning-paths/server-and-cloud/azure/terraform/) before starting this Learning Path.

## Before you begin

You should have the prerequisite tools installed before starting the Learning Path. 

Any computer which has the required tools installed can be used for this section. The computer can be your desktop or laptop computer or a virtual machine with the required tools. 

You will need an [an Azure portal account](https://azure.microsoft.com/en-in/get-started/azure-portal) to complete this Learning Path. Create an account if you don't have one.

Before you begin you will also need:
- Login to Azure CLI
- An SSH key pair

The instructions to login to Azure CLI and to create the keys are below.

### Azure authentication

The installation of Terraform on your Desktop/Laptop needs to communicate with Azure. Thus, Terraform needs to be authenticated.

For authentication, follow the [steps from the Terraform Learning Path](/learning-paths/server-and-cloud/azure/terraform#azure-authentication).

### Generate an SSH key-pair

Generate an SSH key-pair (public key, private key) using `ssh-keygen` to use for Azure access. 

```console
ssh-keygen -f azure_key -t rsa -b 2048 -P ""
```

You should now have your SSH keys in the current directory.

## Create Azure instances using Terraform

For Azure Arm based instance deployment, the Terraform configuration is broken into three files: `providers.tf`, `variables.tf` and `main.tf`. Here we are creating 2 instances.

Add the following code in `providers.tf` file to configure Terraform to communicate with Azure.
    
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
Create a `variables.tf` file for describing the variables referenced in the other files with their type and a default value.

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

Add the resources required to create a virtual machine in `main.tf`.

Scroll down to see the information you need to change in `main.tf`.

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
  address_space = ["10.1.0.0/16"]
  location = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create subnet
resource "azurerm_subnet" "my_terraform_subnet" {
  name = "mySubnet"
  resource_group_name = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.my_terraform_network.name
  address_prefixes = ["10.1.1.0/24"]
}

# Create Public IPs
resource "azurerm_public_ip" "my_terraform_public_ip" {
  name = "myPublicIP${format("%02d", count.index)}-test"
  count= 2
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
    name= "MYSQL"
    priority= 1002
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
  count= 2
  name= "NIC-${format("%02d", count.index)}-test"
  location= azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name= "my_nic_configuration"
    subnet_id = azurerm_subnet.my_terraform_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id= azurerm_public_ip.my_terraform_public_ip.*.id[count.index]
  }
}

# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
  count= 2
  network_interface_id= azurerm_network_interface.my_terraform_nic.*.id[count.index]
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
resource "azurerm_linux_virtual_machine" "MYSQL_TEST" {
  name= "MYSQL_TEST${format("%02d", count.index + 1)}"
  count= 2
  location= azurerm_resource_group.rg.location
  resource_group_name= azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.my_terraform_nic.*.id[count.index]]
  size= "Standard_D2ps_v5"

  os_disk {
    name = "myOsDisk${format("%02d", count.index + 1)}"
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
  admin_username= "ubuntu"
  disable_password_authentication = true

  admin_ssh_key {
    username= "ubuntu"
    public_key = file("/path/to/public_key/azure_key.pub")
  }

  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.my_storage_account.primary_blob_endpoint
  }
}
resource "local_file" "inventory" {
    depends_on=[azurerm_linux_virtual_machine.MYSQL_TEST]
    filename = "(your_current_directory)/hosts"
    content = <<EOF
[mysql1]
${azurerm_linux_virtual_machine.MYSQL_TEST[0].public_ip_address}
[mysql2]
${azurerm_linux_virtual_machine.MYSQL_TEST[1].public_ip_address}
[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}
```
Make the changes listed below in `main.tf` to match your account settings.

1. In the `admin_ssh_key` section, change the `public_key` value to match your SSH key.

2. In the `local_file` section, change the `filename` to be the path to your current directory.

The hosts file is automatically generated and does not need to be changed, change the path to the location of the hosts file.

## Terraform Commands

Use Terraform to deploy the `main.tf` file.

### Initialize Terraform

Run `terraform init` to initialize the Terraform deployment. This command downloads the dependencies required for Azure.

```console
terraform init
```
    
The output should be similar to:

```output
Initializing the backend...

Initializing provider plugins...
- Reusing previous version of hashicorp/local from the dependency lock file
- Reusing previous version of hashicorp/tls from the dependency lock file
- Reusing previous version of hashicorp/azurerm from the dependency lock file
- Reusing previous version of hashicorp/random from the dependency lock file
- Using previously-installed hashicorp/local v2.4.0
- Using previously-installed hashicorp/tls v4.0.4
- Using previously-installed hashicorp/azurerm v2.99.0
- Using previously-installed hashicorp/random v3.4.3

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

### Create a Terraform execution plan

Run `terraform plan` to create an execution plan.

```console
terraform plan
```

A long output of resources to be created will be printed. 

### Apply a Terraform execution plan

Run `terraform apply` to apply the execution plan and create all Azure resources. 

```console
terraform apply
```      

Answer `yes` to the prompt to confirm you want to create Azure resources. 

The output should be similar to:

```output
Apply complete! Resources: 16 added, 0 changed, 0 destroyed.
```

## Configure MySQL through Ansible

Install MySQL and the required dependencies on both the inastances. 

You can use the same `playbook.yaml` file used in the topic, [Deploy Memcached as a cache for MySQL on an AWS Arm based Instance](/learning-paths/server-and-cloud/memcached/memcached_mysql_aws#configure-mysql-through-ansible).

### Ansible Commands

Substitute your private key name, and run the playbook using the  `ansible-playbook` command:

```console
ansible-playbook playbook.yaml -i hosts --key-file azure_key
```

Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to:

```output
ubuntu@ip-172-31-38-39:~/azure-mysql$ ansible-playbook playbook.yaml -i hosts --key-file azure_key

PLAY [mysql1, mysql2] ********************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************
The authenticity of host '20.114.166.37 (20.114.166.37)' can't be established.
ED25519 key fingerprint is SHA256:AFYNvATg2zT/PQJ8I5Qr0JKO+3Jwq6lGYSex4/2gDKs.
This key is not known by any other names
The authenticity of host '20.114.166.67 (20.114.166.67)' can't be established.
ED25519 key fingerprint is SHA256:0gpYKMdRIHpFxaXePtYFxs+k6JSioKcVEN6CMrCJUBA.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
yes
ok: [20.114.166.37]
ok: [20.114.166.67]

TASK [Update the Machine and Install dependencies] ***************************************************************************************************************
changed: [20.114.166.37]
changed: [20.114.166.67]

TASK [start and enable mysql service] ****************************************************************************************************************************
ok: [20.114.166.67]
ok: [20.114.166.37]

TASK [Change Root Password] **************************************************************************************************************************************
changed: [20.114.166.37]
changed: [20.114.166.67]

TASK [Create database user with password and all database privileges and 'WITH GRANT OPTION'] ********************************************************************
changed: [20.114.166.37]
changed: [20.114.166.67]

TASK [Create a new database with name 'arm_test1'] ***************************************************************************************************************
skipping: [20.114.166.67]
changed: [20.114.166.37]

TASK [Create a new database with name 'arm_test2'] ***************************************************************************************************************
skipping: [20.114.166.37]
changed: [20.114.166.67]

TASK [MySQL secure installation] *********************************************************************************************************************************
changed: [20.114.166.67]
changed: [20.114.166.37]

TASK [Enable remote login by changing bind-address] **************************************************************************************************************
changed: [20.114.166.67]
changed: [20.114.166.37]

RUNNING HANDLER [Restart mysql] **********************************************************************************************************************************
changed: [20.114.166.67]
changed: [20.114.166.37]

PLAY RECAP *******************************************************************************************************************************************************
20.114.166.37              : ok=9    changed=7    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
20.114.166.67              : ok=9    changed=7    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

## Connect to Database from local machine

To connect to the database, you need the `public-ip` of the instance where MySQL is deployed. You also need to use the MySQL Client to interact with the MySQL database.

```console
apt install mysql-client
```

```console
mysql -h {public_ip of instance where Mysql deployed} -P3306 -u {user of database} -p{password of database}
```
Replace `{public_ip of instance where Mysql deployed}`, `{user of database}` and `{password of database}` with your values. In this example, `user`= `Local_user`, which is getting created in the `playbook.yaml` file. 

The output will be:
```output
ubuntu@ip-172-31-38-39:~/azure-mysql$ mysql -h 20.114.166.37 -P3306 -u Local_user -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 8.0.32-0ubuntu0.20.04.2 (Ubuntu)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
### Access Database and Create Table

1. You can access your database by using the below commands.

```console
show databases;
```

```console
use {your_database};
```

The output will be:

```output
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| arm_test1          |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
mysql> use arm_test1;
Database changed
```

2. Use the below commands to create a table and insert values into it.

```console
create table book(name char(10),id varchar(10));
```
```console
insert into book(name,id) values ('Abook','10'),('Bbook','20'),('Cbook','20'),('Dbook','30'),('Ebook','45'),('Fbook','40'),('Gbook
','69');
```

The output will be:

```output
mysql> create table book(name char(10),id varchar(10));
Query OK, 0 rows affected (0.03 sec)

mysql> insert into book(name,id) values ('Abook','10'),('Bbook','20'),('Cbook','20'),('Dbook','30'),('Ebook','45'),('Fbook','40'),('Gbook
    '> ','69');
Query OK, 7 rows affected (0.01 sec)
Records: 7  Duplicates: 0  Warnings: 0

```

3. Use the below command to access the content of the table.

```console
select * from {{your_table_name}};
```

The output will be:

```output
mysql> select * from book;
+--------+------+
| name   | id   |
+--------+------+
| Abook  | 10   |
| Bbook  | 20   |
| Cbook  | 20   |
| Dbook  | 30   |
| Ebook  | 45   |
| Fbook  | 40   |
| Gbook
 | 69   |
+--------+------+
7 rows in set (0.00 sec)
```

4. Now connect to the second instance and repeat the above steps with a different data as shown below.
The output will be:

```output
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| arm_test2          |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use arm_test2;
Database changed
mysql> create table movie(name char(10),id varchar(10));
Query OK, 0 rows affected (0.02 sec)

mysql> insert into movie(name,id) values ('Amovie','1'), ('Bmovie','2'), ('Cmovie','3'), ('Dmovie','4'), ('Emovie','5'), ('Fmovie','6'), ('Gmovie','7');
Query OK, 7 rows affected (0.01 sec)
Records: 7  Duplicates: 0  Warnings: 0

mysql> select * from movie;
+--------+------+
| name   | id   |
+--------+------+
| Amovie | 1    |
| Bmovie | 2    |
| Cmovie | 3    |
| Dmovie | 4    |
| Emovie | 5    |
| Fmovie | 6    |
| Gmovie | 7    |
+--------+------+
7 rows in set (0.00 sec)
```

## Deploy Memcached as a cache for MySQL using Python

You will create two `.py` files on the host machine to deploy Memcached as a MySQL cache using Python: `values.py` and `mem.py`.  

`values.py` to store the IP addresses of the instances and the databases created in them.
```console
MYSQL_TEST=[["{{public_ip of MYSQL_TEST[0]}}", "arm_test1"],
["{{public_ip of MYSQL_TEST[1]}}", "arm_test2"]]
```
Replace `{{public_ip of MYSQL_TEST[0]}}` & `{{public_ip of MYSQL_TEST[1]}}` with the public IPs generated in the `hosts` file after running the Terraform commands.       
`mem.py` to access data from Memcached and, if not present, store it in the Memcached.       
```console
import sys
import MySQLdb
import pymemcache
from values import *
from ast import literal_eval
import argparse
parser = argparse.ArgumentParser()

parser.add_argument("-db", "--database", help="Database")
parser.add_argument("-k", "--key", help="Key")
parser.add_argument("-q", "--query", help="Query")
args = parser.parse_args()

memc = pymemcache.Client("127.0.0.1:11211");

for i in range(0,2):
    if (MYSQL_TEST[i][1]==args.database):
        try:
            conn = MySQLdb.connect (host = MYSQL_TEST[i][0],
                                    user = "{{Your_database_user}}",
                                    passwd = "{{Your_database_password}}",
                                    db = MYSQL_TEST[i][1])
        except MySQLdb.Error as e:
             print ("Error %d: %s" % (e.args[0], e.args[1]))
             sys.exit (1)

        sqldata = memc.get(args.key)

        if not sqldata:
            cursor = conn.cursor()
            cursor.execute(args.query)
            rows = cursor.fetchall()
            memc.set(args.key,rows,120)
            print ("Updated memcached with MySQL data")
            for x in rows:
                print(x)
        else:
            print ("Loaded data from memcached")
            data = tuple(literal_eval(sqldata.decode("utf-8")))
            for row in data:
                print (f"{row[0]},{row[1]}")
        break
else:
    print("this database doesn't exist")            
```
Replace `{{Your_database_user}}` & `{{Your_database_password}}` with the database user and password created through Ansible-Playbook. Also change the `range` in `for loop` according to the number of instances created.

To execute the script, run the following command:
```console
python3 mem.py -db {database_name} -k {key} -q {query}
```
Replace `{database_name}` with the database you want to access, `{query}` with the query you want to run in the database and `{key}` with a variable to store the result of the query in Memcached.

When the script is executed for the first time, the data is loaded from the MySQL database and stored on the Memcached server.

The output will be:
```output
ubuntu@ip-172-31-38-39:~/azure-mysql$ python3 memcached.py -db arm_test1 -k AA -q "select * from book limit 3"
Updated memcached with MySQL data
('Abook', '10')
('Bbook', '20')
('Cbook', '20')
```
```output
ubuntu@ip-172-31-38-39:~/azure-mysql$ python3 memcached.py -db arm_test2 -k BB -q "select * from movie limit 3"
Updated memcached with MySQL data
('Amovie', '1')
('Bmovie', '2')
('Cmovie', '3')
```

When executed after that, it loads the data from Memcached. In the example above, the information stored in Memcached is in the form of rows from a Python DB cursor. When accessing the information (within the 120 second expiry time), the data is loaded from Memcached and dumped.

The output will be:
```output
ubuntu@ip-172-31-38-39:~/azure-mysql$ python3 memcached.py -db arm_test1 -k AA -q "select * from book limit 3"
Loaded data from memcached
Abook,10
Bbook,20
Cbook,20
```

```output
ubuntu@ip-172-31-38-39:~/azure-mysql$ python3 memcached.py -db arm_test2 -k BB -q "select * from movie limit 3"
Loaded data from memcached
Amovie,1
Bmovie,2
Cmovie,3
```

### Memcached Telnet Commands

Execute the steps below to verify that the MySQL query is getting stored in Memcached
1. Connect to the Memcached server with Telnet and start a session:
```console
telnet localhost 11211
```
2. Retrieve data from Memcached through Telnet:
```console
get <key>
```
**NOTE:-** Key is the variable in which we store the data. In the above command, we are storing the data from the tables `book` and `movie` in `AA` and `BB` respectively.

The output will be:

```output
ubuntu@ip-172-31-38-39:~/azure-mysql$ telnet localhost 11211
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
get AA
VALUE AA 0 51
(('Abook', '10'), ('Bbook', '20'), ('Cbook', '20'))
END
get BB
VALUE BB 0 51
(('Amovie', '1'), ('Bmovie', '2'), ('Cmovie', '3'))
END
```

You have successfully deployed Memcached as a cache for MySQL on an Azure Arm based Instance.

### Clean up resources

Run `terraform destroy` to delete all resources created.

```console
terraform destroy
```

Continue the Learning Path to deploy Memcached as a cache for MySQL on a GCP Arm based Instance.
