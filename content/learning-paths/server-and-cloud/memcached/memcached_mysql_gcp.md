---
# User change
title: "Deploy Memcached as a cache for MySQL on a Google Cloud Arm based Instance"

weight: 5 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Memcached as a cache for MySQL on a Google Cloud Arm based Instance

You can deploy Memcached as a cache for MySQL on Google Cloud using Terraform and Ansible. 

In this section, you will deploy Memcached as a cache for MySQL on a Google Cloud instance.

If you are new to Terraform, you should look at [Automate GCP instance creation using Terraform](/learning-paths/server-and-cloud/gcp/terraform/) before starting this Learning Path.

## Before you begin

You should have the prerequisite tools installed before starting the Learning Path. 

Any computer which has the required tools installed can be used for this section. The computer can be your desktop or laptop computer or a virtual machine with the required tools. 

You will need a [Google Cloud account](https://console.cloud.google.com/?hl=en-au) to complete this Learning Path. Create an account if you don't have one.

Before you begin you will also need:
- Login to the Google Cloud CLI 
- An SSH key pair

The instructions to login to the Google Cloud CLI and create the keys are below.

### Acquire user credentials

To obtain user access credentials, follow the [steps from the Terraform Learning Path](/learning-paths/server-and-cloud/gcp/terraform#acquire-user-credentials).

### Generate an SSH key-pair

Generate an SSH key-pair (public key, private key) using `ssh-keygen` to use for Google Cloud access. 

```console
ssh-keygen -f gcp_key -t rsa -b 2048 -P ""
```

You should now have your SSH keys in the current directory.

## Create GCP instances using Terraform

Using a text editor, save the code below in a file called `main.tf`. Here we are creating 2 instances.
    
```console
provider "google" {
  project = "{project_id}"
  region = "us-central1"
  zone = "us-central1-a"
}

resource "google_compute_instance" "MYSQL_TEST" {
  name         = "mysqltest-${count.index+1}"
  count        = "2"
  machine_type = "t2a-standard-1"

  boot_disk {
    initialize_params {
      image = "ubuntu-os-cloud/ubuntu-2204-lts-arm64"
    }
  }

  network_interface {
    network = "default"
    access_config {
      // Ephemeral public IP
    }
  }
  metadata = {
     ssh-keys = "ubuntu:${file("public_key_location")}"
  }  
}
resource "google_compute_firewall" "default" {
  name    = "test-firewall"
  network = google_compute_network.default.name

  allow {
    protocol = "icmp"
  }

  allow {
    protocol = "tcp"
    ports    = ["22", "3306"]
  }

  source_tags = ["web"]
}

resource "google_compute_network" "default" {
  name = "test-network1"
}
resource "local_file" "inventory" {
    depends_on=[google_compute_instance.MYSQL_TEST]
    filename = "(your_current_directory)/hosts"
    content = <<EOF
[mysql1]
${google_compute_instance.MYSQL_TEST[0].network_interface.0.access_config.0.nat_ip}
[mysql2]
${google_compute_instance.MYSQL_TEST[1].network_interface.0.access_config.0.nat_ip}
[all:vars]
ansible_connection=ssh
ansible_user=ubuntu
                EOF
}
```
Make the changes listed below in `main.tf` to match your account settings.

1. In the `provider` section, update the `project_id` with your value.

2. In the `google_compute_project_metadata_item` section, change the `public_key_location` value to match your SSH key.

3. In the `local_file` section, change the `filename` to be the path to your current directory.

The hosts file is automatically generated and does not need to be changed, change the path to the location of the hosts file.

## Terraform Commands

Use Terraform to deploy the `main.tf` file.

### Initialize Terraform

Run `terraform init` to initialize the Terraform deployment. This command downloads the dependencies required for GCP.

```console
terraform init
```
    
The output should be similar to:

```output
Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/google...
- Finding latest version of hashicorp/local...
- Installing hashicorp/google v4.57.0...
- Installed hashicorp/google v4.57.0 (signed by HashiCorp)
- Installing hashicorp/local v2.4.0...
- Installed hashicorp/local v2.4.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

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

Run `terraform apply` to apply the execution plan and create all GCP resources. 

```console
terraform apply
```      

Answer `yes` to the prompt to confirm you want to create GCP resources. 

The output should be similar to:

```output
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

```

## Configure MySQL through Ansible

Install MySQL and the required dependencies on both the instances. 

You can use the same `playbook.yaml` file used in the topic, [Deploy Memcached as a cache for MySQL on an AWS Arm based Instance](/learning-paths/server-and-cloud/memcached/memcached_mysql_aws#configure-mysql-through-ansible).

### Ansible Commands

Substitute your private key name, and run the playbook using the `ansible-playbook` command:

```console
ansible-playbook playbook.yaml -i hosts --key-file gcp_key
```

Answer `yes` when prompted for the SSH connection. 

Deployment may take a few minutes. 

The output should be similar to:

```output
ubuntu@ip-172-31-38-39:~/gcp-mysql$ ansible-playbook playbook.yaml -i hosts --key-file gcp_key

PLAY [mysql1, mysql2] ********************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************
The authenticity of host '34.28.237.71 (34.28.237.71)' can't be established.
ED25519 key fingerprint is SHA256:xOr4xr3TvaRdPxX4QlxhYpjf9mykgmhAtWElxkhqK3w.
This key is not known by any other names
The authenticity of host '35.222.119.249 (35.222.119.249)' can't be established.
ED25519 key fingerprint is SHA256:gHsDuIJ9IVFrOeeYUZXMEvFOu5tXL0ZB78aHwZjooTI.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
yes
ok: [34.28.237.71]
ok: [35.222.119.249]

TASK [Update the Machine and Install dependencies] ***************************************************************************************************************
changed: [35.222.119.249]
changed: [34.28.237.71]

TASK [start and enable mysql service] ****************************************************************************************************************************
ok: [34.28.237.71]
ok: [35.222.119.249]

TASK [Change Root Password] **************************************************************************************************************************************
changed: [34.28.237.71]
changed: [35.222.119.249]

TASK [Create database user with password and all database privileges and 'WITH GRANT OPTION'] ********************************************************************
changed: [34.28.237.71]
changed: [35.222.119.249]

TASK [Create a new database with name 'arm_test1'] ***************************************************************************************************************
skipping: [35.222.119.249]
changed: [34.28.237.71]

TASK [Create a new database with name 'arm_test2'] ***************************************************************************************************************
skipping: [34.28.237.71]
changed: [35.222.119.249]

TASK [MySQL secure installation] *********************************************************************************************************************************
changed: [35.222.119.249]
changed: [34.28.237.71]

TASK [Enable remote login by changing bind-address] **************************************************************************************************************
changed: [34.28.237.71]
changed: [35.222.119.249]

RUNNING HANDLER [Restart mysql] **********************************************************************************************************************************
changed: [35.222.119.249]
changed: [34.28.237.71]

PLAY RECAP *******************************************************************************************************************************************************
34.28.237.71               : ok=9    changed=7    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
35.222.119.249             : ok=9    changed=7    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
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
ubuntu@ip-172-31-38-39:~/gcp-mysql$ mysql -h 34.28.253.43 -u Local_user -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.32-0ubuntu0.22.04.2 (Ubuntu)

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

You will create two `.py` files on the host machine to deploy Memcached as a MySQL cache using Python: `values.py` and `memcached.py`.  

`values.py` to store the IP addresses of the instances and the databases created in them.
```console
MYSQL_TEST=[["{{public_ip of MYSQL_TEST[0]}}", "arm_test1"],
["{{public_ip of MYSQL_TEST[1]}}", "arm_test2"]]
```
Replace `{{public_ip of MYSQL_TEST[0]}}` & `{{public_ip of MYSQL_TEST[1]}}` with the public IPs generated in the `hosts` file after running the Terraform commands.       
`memcached.py` to access data from Memcached and, if not present, store it in the Memcached.       
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
python3 memcached.py -db {database_name} -k {key} -q {query}
```
Replace `{database_name}` with the database you want to access, `{query}` with the query you want to run in the database and `{key}` with a variable to store the result of the query in Memcached.

When the script is executed for the first time, the data is loaded from the MySQL database and stored on the Memcached server.

The output will be:
```output
ubuntu@ip-172-31-38-39:~/gcp-mysql$ python3 memcached.py -db arm_test1 -k AA -q "select * from book limit 3"
Updated memcached with MySQL data
('Abook', '10')
('Bbook', '20')
('Cbook', '20')
```
```output
ubuntu@ip-172-31-38-39:~/gcp-mysql$ python3 memcached.py -db arm_test2 -k BB -q "select * from movie limit 3"
Updated memcached with MySQL data
('Amovie', '1')
('Bmovie', '2')
('Cmovie', '3')
```

When executed after that, it loads the data from Memcached. In the example above, the information stored in Memcached is in the form of rows from a Python DB cursor. When accessing the information (within the 120-second expiry time), the data is loaded from Memcached and dumped.

The output will be:
```output
ubuntu@ip-172-31-38-39:~/gcp-mysql$ python3 memcached.py -db arm_test1 -k AA -q "select * from book limit 3"
Loaded data from memcached
Abook,10
Bbook,20
Cbook,20
```
```output
ubuntu@ip-172-31-38-39:~/gcp-mysql$ python3 memcached.py -db arm_test2 -k BB -q "select * from movie limit 3"
Loaded data from memcached
Amovie,1
Bmovie,2
Cmovie,3
```

### Memcached Telnet Commands

Execute the steps below to verify that the MySQL query is getting stored in Memcached.
1. Connect to the Memcached server with Telnet and start a session.
```console
telnet localhost 11211
```
2. Retrieve data from Memcached through Telnet.
```console
get <key>
```
**NOTE:-** Key is the variable in which we store the data. In the above command, we are storing the data from the tables `book` and `movie` in `AA` and `BB` respectively.

The output will be:

```output
ubuntu@ip-172-31-38-39:~/gcp-mysql$ telnet localhost 11211
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
get AA
VALUE AA 0 51
(('Abook', '10'), ('Bbook', '20'), ('Cbook', '20'))
END
get BB
END
```
You have successfully deployed Memcached as a cache for MySQL on a Google Cloud Arm based Instance.

### Clean up resources

Run `terraform destroy` to delete all resources created.

```console
terraform destroy
```
