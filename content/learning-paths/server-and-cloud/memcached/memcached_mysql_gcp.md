---
# User change
title: "Deploy Memcached as a cache for MySQL on a Google Cloud Arm based Instance"

weight: 5 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Memcached as a cache for MySQL on a Google Cloud Arm based Instance

You can deploy Memcached as a cache for MySQL on Google Cloud using Terraform and Ansible. 

In this topic, you will deploy Memcached as a cache for MySQL on Google Cloud instance.

If you are new to Terraform, you should look at [Automate GCP instance creation using Terraform](/learning-paths/server-and-cloud/gcp/terraform/) before starting this Learning Path.

## Before you begin

You should have the prerequisite tools installed before starting the Learning Path. 

Any computer which has the required tools installed can be used for this section. The computer can be your desktop or laptop computer or a virtual machine with the required tools. 

You will need an [Google Cloud account](https://console.cloud.google.com/?hl=en-au) to complete this Learning Path. Create an account if you don't have one.

Before you begin you will also need:
- Login to Google Cloud CLI 
- An SSH key pair

The instructions to login to Google Cloud CLI and to create the keys are below.

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

Scroll down to see the information you need to change in `main.tf`.
    
```console
provider "google" {
  project = "{project_id}"
  region = "us-central1"
  zone = "us-central1-a"
}

resource "google_compute_project_metadata_item" "ssh-keys" {
  key   = "ssh-keys"
  value = "ubuntu:${file("public_key_location")}"
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

Run `terraform init` to initialize the Terraform deployment. This command downloads the dependencies required for Google Cloud.

```console
terraform init
```
    
The output should be similar to:

```console
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

```console

```

## Configure MySQL through Ansible
An Ansible Playbook installs & enables MySQL in the instances and creates databases & tables inside them. To configure MySQL through Ansible and run the Playbook, follow this [documentation](/learning-paths/server-and-cloud/memcached/memcached_mysql_aws#configure-mysql-through-ansible).

## Deploy Memcached as a cache for MySQL using Python
To deploy Memcached as a cache for MySQL using Python, follow this [documentation](/learning-paths/server-and-cloud/memcached/memcached_mysql_aws#deploy-memcached-as-a-cache-for-mysql-using-python).
