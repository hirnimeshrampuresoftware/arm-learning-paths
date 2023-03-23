---
# User change
title: "Deploy Redis as a cache for MySQL on AWS Arm based Instance"

weight: 8 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Redis as a cache for MySQL on AWS Arm based Instance

## Prerequisites

* [An AWS account](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&src=default&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](https://developer.hashicorp.com/terraform/cli/install/apt)
* [Python](https://docs.python-guide.org/starting/install3/linux/#install3-linux)

## Install Redis on AWS Arm based Instance

To install Redis follow this [documentation](/content/learning-paths/server-and-cloud/redis/aws_deployment.md).

## Deploy Redis as a cache for MySQL on a single AWS Arm based instance

## Deploy AWS Arm based instance via Terraform

Before deploying AWS Arm based instance via Terraform, generate [Access keys](/content/learning-paths/server-and-cloud/mysql/ec2_deployment.md#generate-access-keys-access-key-id-and-secret-access-key) and [key-pair using ssh keygen](/content/learning-paths/server-and-cloud/mysql/ec2_deployment.md#generate-key-pairpublic-key-private-key-using-ssh-keygen).

After generating the public and private keys, we need to create an AWS Arm based instance. Then we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22`(ssh), `3306`(MySQL) and `6000`(Redis). Below is a Terraform file named **main.tf** which will do this for us.

```console
provider "aws" {
  region = "us-east-2"
  access_key  = "AXXXXXXXXXXXXXXXXXXX"
  secret_key   = "AAXXXXXXXXXXXXXXXXXXXXXXXXXXX"
}
resource "aws_instance" "aws-deployment" {
  ami = "ami-0bc02c3c09aaee8ea"
  instance_type = "t4g.small"
  key_name= "aws_key"
  vpc_security_group_ids = [aws_security_group.main.id]
}

resource "aws_security_group" "main" {
  name        = "main"
  description = "Allow TLS inbound traffic"

  ingress {
    description      = "Open redis connection port"
    from_port        = 6000
    to_port          = 6000
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  ingress {
    description      = "Open mysql connection port"
    from_port        = 3306
    to_port          = 3306
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
  }
  ingress {
    description      = "Allow ssh to instance"
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
}

resource "local_file" "inventory" {
    depends_on=[aws_instance.aws-deployment]
    filename = "inventory.txt"
    content = <<EOF
[all]
ansible-target1 ansible_connection=ssh ansible_host=${aws_instance.aws-deployment.public_dns} ansible_user=ubuntu
                EOF
}

resource "aws_key_pair" "deployer" {
        key_name   = "aws_key"
        public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCUZXm6T6JTQBuxw7aFaH6gmxDnjSOnHbrI59nf+YCHPqIHMlGaxWw0/xlaJiJynjOt67Zjeu1wNPifh2tzdN3UUD7eUFSGcLQaCFBDorDzfZpz4wLDguRuOngnXw+2Z3Iihy2rCH+5CIP2nCBZ+LuZuZ0oUd9rbGy6pb2gLmF89GYzs2RGG+bFaRR/3n3zR5ehgCYzJjFGzI8HrvyBlFFDgLqvI2KwcHwU2iHjjhAt54XzJ1oqevRGBiET/8RVsLNu+6UCHW6HE9r+T5yQZH50nYkSl/QKlxBj0tGHXAahhOBpk0ukwUlfbGcK6SVXmqtZaOuMNlNvssbocdg1KwOH ubuntu@ip-172-31-XXXX-XXXX"
}
```
 **NOTE:-** Replace `public_key`, `access_key`, `secret_key`, and `key_name` with respective values. In our example, we have used port number `6000`.

Now, use the below Terraform commands to deploy the **main.tf** file.

### Terraform Commands

To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/content/learning-paths/server-and-cloud/redis/aws_deployment.md#terraform-commands) to deploy the **main.tf** file.

## Install Redis and MySQL via Ansible

To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`. The following Playbook contains a collection of tasks to install MySQL database and Redis.

Here is the complete **deploy_mysql.yml** file of Ansible-Playbook
```console
---
- hosts: all
  become: true
  become_user: root
  remote_user: ubuntu

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
      become_user: ubuntu
    - name: Set Authentication password
      shell: redis-cli -p 6000 CONFIG SET requirepass "{password}"
      become_user: ubuntu
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
         login_password: {{Your_mysql_password}}
         login_host: localhost
         name: Local_user
         host: '%'
         password: {{Give_any_password}}
         priv: '*.*:ALL,GRANT'
         state: present
    - name: Create a new database with name 'arm_test'
      community.mysql.mysql_db:
        name: arm_test
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
**NOTE:-** Replace `{{Your_mysql_password}}` and `{{Give_any_password}}` with your password.

In our case, the inventory file will generate automatically after the `terraform apply` command.

### Ansible Commands
To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}`, `{your_inventory_file}` and `{path_to_private_key}` with your values.

Here is the output after the successful execution of the `ansible-playbook` commands.



## Deploy Redis as a cache for single MySQL database on a single AWS Arm based instance using Python
To deploy Redis as a cache for MySQL using Python, we need to create four files: **requirements.txt**, **values.json**, **gen_data.py** and **redis_mysql.py**.

Create a **requirements.txt** file with a list of all the necessary Python modules.
```console
redis==4.5.1
mysql-connector-python==8.0.32
Faker==17.0.0
beautifultable==1.1.0
requests==2.22.0
argparse==1.4.0
sqlvalidator==0.0.20
```

Install the necessary Python modules using `pip`:
```console
pip install -r requirements.txt
```

Create a **values.json** file to store credentials for MySQL and Redis:
```console
{
    "mysql": {
        {
            "host": "{host_ip}",
            "user": "{username}",
            "password": "{mysql_password}",
            "db": "{database_name}"
        }
    },
    "redis": {
        {
            "host": "{host_ip}",
            "port": {port_number},
            "password": "{redis_password}"
        }
    }
}
```
**NOTE:-** Get value of `{host_ip}` from **inventory.txt** file and replace `{username}`, `{mysql_password}`, `{database_name}`, `{port_number}` and `{redis_password}` with respective values.

Create **gen_data.py** file to create dummy table **products** having four columns: **Id**, **Name**, **Quantity** and **Cost**. The python file also fills dummy data into the four columns for testing purpose.
```console
#!/usr/bin/python3
from faker import Faker
import faker_commerce
import mysql.connector
import json


def connect_mysql(host, user, password, database):
    try:
        print("Attempting connection to MySQL database {} at {}@{}".format(database,user,host))
        conn = mysql.connector.connect(host=host, user=user, password=password, database=database)
        if conn.is_connected():
            print("Connected to MySQL database {} at {}@{}".format(database,user,host))
            return conn
    except mysql.connector.Error as e:
        print ("Error while connecting to MySQL database {} at {}@{}".format(database,user,host), e)


def gen_records(cnx, n):
    # Create a new Faker instance
    fake = Faker()
    fake.add_provider(faker_commerce.Provider)
    cursor = cnx.cursor()
    # Drop the table if it already exists
    query = "DROP TABLE IF EXISTS products;"
    cursor.execute(query)

    # Create Table products
    table_products: str = """
    create table products (Id int not null AUTO_INCREMENT primary key ,Name varchar(100),Quantity int, Cost int);
    """
    cursor.execute(table_products)

    # generate and insert fake data into the products table
    for i in range(n):
        name = fake.ecommerce_name()
        qty = fake.random_int(0, 5000)
        cost = round(fake.random_int(1000,50000),-2)
        query = "INSERT INTO products (Name, Quantity, Cost) VALUES (%s, %s, %s)"
        cursor.execute(query, (name, qty, cost))

    cnx.commit()
    # close the database connection
    cnx.close()

if __name__=='__main__':
    with open("values.json", "r") as f:
        val = json.load(f)
    mydb = connect_mysql(val['mysql']['host'], val['mysql']['user'], val['mysql']['password'], val['mysql']['db'])
    N = int(input("Enter number of records to generate: "))
    if mydb:
        gen_records(mydb, N)
```

To execute the above script, run the following command:
```console
python3 {script_name}
```

Create **redis_mysql.py** file to implement Redis as cache for MySQL
```console
#!/usr/bin/python3
import redis
import mysql.connector
from hashlib import shake_256
from beautifultable import BeautifulTable
import argparse
import json
import sqlvalidator
import sys


def get_args():
    parser = argparse.ArgumentParser()
    optional = parser._action_groups.pop()
    required = parser.add_argument_group('Required arguments')
    parser._action_groups.append(optional)
    required.add_argument('-q', "--query", help='Provide SQL query to be executed', required=True)
    return parser.parse_args()


def connect_redis(host, port, password):
    try:
        print("Attempting connection to Redis at {}:{}".format(host, port))
        redis_db = redis.Redis(host=host, port=port, password=password, db=0, decode_responses=True, retry_on_timeout=True, retry=3)
        if redis_db.ping():
            print("Connected to Redis at {}:{}".format(host, port))
            return redis_db
    except redis.RedisError as e:
        print ("Error while connecting to Redis at {}:{} ".format(host, port), e)


def connect_mysql(host, user, password, database):
    try:
        print("Attempting connection to MySQL database {} at {}@{}".format(database, user, host))
        conn = mysql.connector.connect(host=host, user=user, password=password, database=database)
        if conn.is_connected():
            print("Connected to MySQL database {} at {}@{}".format(database, user, host))
            return conn
    except mysql.connector.Error as e:
        print ("Error while connecting to MySQL database {} at {}@{}".format(database, user, host), e)


def get_data_from_redis(redis_db, key):
    # Check if the data is already in the cache
    cache_data = redis_db.hgetall(key)
    if cache_data:
        print("Fetching data from Redis cache having key: {}.".format(key))
        return cache_data
    return None


def get_data_from_mysql(mydb, key, query):
    # If data is not in the cache, retrieve it from MySQL database
    print("Data is not present in Redis cache for key: {}. Fetching data from MySQL database with query: {}".format(key,query))
    cursor = mydb.cursor()
    cursor.execute(query)
    headers = [i[0] for i in cursor.description]
    data = cursor.fetchall()

    # Store the data in the cache
    dt_d = {}
    for i in range(len(headers)):
        dt_d[headers[i]] = []
    for i in range(len(data)):
        for j in range(len(headers)):
            dt_d[headers[j]].append(data[i][j])
    redis_db.hset(key, mapping=dt_d)

    return headers, data

def display_table(headers, row):
     table=BeautifulTable()
     table.columns.header = headers
     for i in range(len(row)):
         table.rows.append(row[i])
     print(table)

if __name__=='__main__':
    # Get values of arguments into variable args
    args = get_args()
    
    sql_query = sqlvalidator.parse(args.query)

    if not sql_query.is_valid():
        print("Invalid SQL query:", sql_query.errors)
        sys.exit(0)

    # Load values from values.json file
    with open("values.json","r") as f:
        val = json.load(f)

    # Connect to MySQL database
    mydb = connect_mysql(val['mysql']['host'], val['mysql']['user'], val['mysql']['password'], val['mysql']['db'])
    
    # Connect to Redis server
    redis_db = connect_redis(val['redis']['host'], val['redis']['port'], val['redis']['password'])

    # Generate a unique hash key for SQL query
    K = 'product#'+shake_256(args.query.encode('utf-8')).hexdigest(8)
    print("Generated hash key for the SQL query {} is {}".format(args.query, K))
    
    # Fetch data from redis
    redis_data = get_data_from_redis(redis_db, K)
    if redis_data:
        # Display data fetched from Redis
        display_table(list(redis_data.keys()), list(redis_data.values()))
    else:
        headers, row = get_data_from_mysql(mydb, K, args.query)
        # Display data fetched from MySQL
        display_table(headers, row)
```

To execute the above script, run the following command:
```console
python3 {script_name} -q {SQL_query}
```
**NOTE:-** Replace `{SQL_query}` with its respective value.


## Deploy Redis as a cache for MySQL on two AWS Arm based instances

### Deploy two AWS Arm based instances via terraform
After generating the keys, we need to create two MySQL AWS Arm based instances. Then we will push our public key to the **authorized_keys** folder in `~/.ssh`. We will also create a security group that opens inbound ports `22` (ssh) and `3306` (MySQL). Below is a Terraform file called **main.tf** that will do this for us. 
    
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

## Install Redis and MySQL via Ansible
To run Ansible, we have to create a `.yml` file, which is also known as `Ansible-Playbook`.he following Playbook contains a collection of tasks to install MySQL database and Redis.

Here is the complete **deploy_mysql.yml** file of Ansible-Playbook
```console
---
- hosts: mysql1, mysql2
  remote_user: root
  become_user: root
  remote_user: ubuntu

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
      become_user: ubuntu
    - name: Set Authentication password
      shell: redis-cli -p 6000 CONFIG SET requirepass "{password}"
      become_user: ubuntu       
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

### Ansible Commands
To run a Playbook, we need to use the `ansible-playbook` command.
```console
ansible-playbook {your_yml_file} -i {your_inventory_file} --key-file {path_to_private_key}
```
**NOTE:-** Replace `{your_yml_file}`, `{your_inventory_file}` and `{path_to_private_key}` with your values.

Here is the output after the successful execution of the `ansible-playbook` commands.


## Deploy Redis as a cache for two independent MySQL database on two AWS Arm based instance using Python
To deploy Redis as a cache for MySQL using Python, we need to create four files: **requirements.txt**, **values.json**, **gen_data.py** and **redis_mysql.py**.

To create **requirements.txt** file and install necessary Python modules follow this [documentation](#deploy-redis-as-a-cache-for-single-mysql-database-on-a-single-aws-arm-based-instance-using-python).

Create a **values.json** file to store credentials for two MySQL database and single Redis:
```console
{
    "mysql1": {
        {
            "host": "{mysql1_host_ip}",
            "user": "{mysql1_username}",
            "password": "{mysql1_password}",
            "db": "{mysql1_database_name}"
        }
    },
    "mysql2": {
        {
            "host": "{mysql2_host_ip}",
            "user": "{mysql2_username}",
            "password": "{mysql2_password}",
            "db": "{mysql1_database_name}"
        }
    },
    "redis": {
        {
            "host": "{host_ip}",
            "port": {port_number},
            "password": "{redis_password}"
        }
    }
}
```
**NOTE:-** Get value of `{mysql1_host_ip}` and `{mysql2_host_ip}` from **inventory.txt** file and replace `{mysql1_username}`, `{mysql1_password}`, `{mysql1_database_name}`, `{mysql2_username}`, `{mysql2_password}`, `{mysql2_database_name}`, `{port_number}` and `{redis_password}` with respective values.

Create **gen_data.py** file to create dummy table **products** having four columns: **Id**, **Name**, **Quantity** and **Cost**. The python file also fills dummy data into the four columns for testing purpose.
```console
#!/usr/bin/python3
from faker import Faker
import faker_commerce
import mysql.connector
import json


def connect_mysql(host, user, password, database):
    try:
        print("Attempting connection to MySQL database {} at {}@{}".format(database,user,host))
        conn = mysql.connector.connect(host=host, user=user, password=password, database=database)
        if conn.is_connected():
            print("Connected to MySQL database {} at {}@{}".format(database,user,host))
            return conn
    except mysql.connector.Error as e:
        print ("Error while connecting to MySQL database {} at {}@{}".format(database,user,host), e)


def gen_records(cnx, n, tname):
    # Create a new Faker instance
    fake = Faker()
    fake.add_provider(faker_commerce.Provider)
    cursor = cnx.cursor()
    # Drop the table if it already exists
    query = "DROP TABLE IF EXISTS {};".format(tname)
    cursor.execute(query)

    # generate and insert fake data into the products table
    if tname=="products":
        table_products: str = """
        create table products (Id int not null AUTO_INCREMENT primary key ,Name varchar(100),Quantity int, Cost int);
        """
        cursor.execute(table_products)
        for i in range(n):
            name = fake.ecommerce_name()
            qty = fake.random_int(0, 5000)
            cost = round(fake.random_int(1000,50000),-2)
            query = "INSERT INTO products (Name, Quantity, Cost) VALUES (%s, %s, %s)"
            cursor.execute(query, (name, qty, cost))
    elif tname=="companies":
        table_companies: str = """
        create table companies (Id int not null AUTO_INCREMENT primary key ,Name varchar(100),Email varchar(100), Phone_no varchar(20));
        """
        cursor.execute(table_companies)
        for i in range(n):
            name = fake.company()
            email = fake.email()
            phno = fake.phone_number()
            query = "INSERT INTO companies (Name, Email, Phone_no) VALUES (%s, %s, %s)"
            cursor.execute(query, (name, email, phno))
    else:
        print("Invalid Table Name")
        return None

    cnx.commit()
    # close the database connection
    cnx.close()

if __name__=='__main__':
    with open("values.json", "r") as f:
        val = json.load(f)
    mydb1 = connect_mysql(val['mysql1']['host'], val['mysql1']['user'], val['mysql1']['password'], val['mysql1']['db'])
    mydb2 = connect_mysql(val['mysql2']['host'], val['mysql2']['user'], val['mysql2']['password'], val['mysql2']['db'])
    N = int(input("Enter number of records to generate: "))
    if mydb1:
        gen_records(mydb1, N, "products")
    if mydb2:
        gen_records(mydb2, N, "companies")
```

To execute the script, run the following command:
```console
python3 {filename.py}
```

Create **redis_mysql.py** file to implement Redis as cache for MySQL
```console
#!/usr/bin/python3
import redis
import mysql.connector
from hashlib import shake_256
from beautifultable import BeautifulTable
import argparse
import json
import sqlvalidator
import sys


def get_args():
    parser = argparse.ArgumentParser()
    optional = parser._action_groups.pop()
    required = parser.add_argument_group('Required arguments')
    parser._action_groups.append(optional)
    required.add_argument('-t', "--table", help='Name of Table to fetch data from', required=True)
    required.add_argument('-q', "--query", help='Provide SQL query to be executed', required=True)
    return parser.parse_args()


def connect_redis(host,port):
    try:
        print("Attempting connection to Redis at {}:{}".format(host, port))
        redis_db = redis.Redis(host=host, port=port, db=0, decode_responses=True, retry_on_timeout=True, retry=3)
        if redis_db.ping():
            print("Connected to Redis at {}:{}".format(host, port))
            return redis_db
    except redis.RedisError as e:
        print ("Error while connecting to Redis at {}:{} ".format(host, port), e)


def connect_mysql(host, user, password, database):
    try:
        print("Attempting connection to MySQL database {} at {}@{}".format(database, user, host))
        conn = mysql.connector.connect(host=host, user=user, password=password, database=database)
        if conn.is_connected():
            print("Connected to MySQL database {} at {}@{}".format(database, user, host))
            return conn
    except mysql.connector.Error as e:
        print ("Error while connecting to MySQL database {} at {}@{}".format(database, user, host), e)


def get_data_from_redis(redis_db, key):
    # Check if the data is already in the cache
    cache_data = redis_db.hgetall(key)
    if cache_data:
        print("Fetching data from Redis cache having key: {}.".format(key))
        return cache_data
    return None


def get_data_from_mysql(mydb, key, query):
    # If data is not in the cache, retrieve it from MySQL database
    print("Data is not present in Redis cache for key: {}. Fetching data from MySQL database with query: {}".format(key,query))
    cursor = mydb.cursor()
    try:
        cursor.execute(query)
    except Exception as e:
        print("Error executing the query {}\n{}".format(query,e))
    headers = [i[0] for i in cursor.description]
    data = cursor.fetchall()

    # Store the data in the cache
    dt_d = {}
    for i in range(len(headers)):
        dt_d[headers[i]] = []
    for i in range(len(data)):
        for j in range(len(headers)):
            dt_d[headers[j]].append(data[i][j])
    redis_db.hset(key, mapping=dt_d)

    return headers, data

def display_table(headers, row):
     table=BeautifulTable()
     table.columns.header = headers
     for i in range(len(row)):
         table.rows.append(row[i])
     print(table)

if __name__=='__main__':
    # Get values of arguments into variable args
    args = get_args()
    
    if args.table not in ["products","companies"]:
        print("Table {} does not exist in any database.".format(args.table))
        sys.exit(0)

    sql_query = sqlvalidator.parse(args.query)

    if not sql_query.is_valid():
        print("Invalid SQL query:", sql_query.errors)
        sys.exit(0)

    # Load values from values.json file
    with open("values.json","r") as f:
        val = json.load(f)

    # Connect to MySQL database 1
    mydb1 = connect_mysql(val['mysql1']['host'], val['mysql1']['user'], val['mysql1']['password'], val['mysql1']['db'])

    # Connect to MySQL database 2
    mydb2 = connect_mysql(val['mysql2']['host'], val['mysql2']['user'], val['mysql2']['password'], val['mysql2']['db'])
    
    # Connect to Redis server
    redis_db = connect_redis(val['mysql']['redis'], val['redis']['port'])

    # Generate a unique hash key for SQL query
    K = 'product#'+shake_256(args.query.encode('utf-8')).hexdigest(8)
    print("Generated hash key for the SQL query {} is {}".format(args.query, K))
    
    # Fetch data from redis
    redis_data = get_data_from_redis(redis_db, K)
    if redis_data:
        # Display data fetched from Redis
        display_table(list(redis_data.keys()), list(redis_data.values()))
    else:
        if args.table=="products":
            headers, row = get_data_from_mysql(mydb1, K, args.query)
            display_table(headers, row)
        elif args.table=="companies":
            headers, row = get_data_from_mysql(mydb2, K, args.query)
            display_table(headers, row)
        else:
            print("Error occurred while running this program")
            sys.exit(0)
```
