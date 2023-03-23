---
# User change
title: "Deploy Redis as a cache for MySQL on GCP Arm based Instance"

weight: 10 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"
---

##  Deploy Redis as a cache for MySQL on GCP Arm based Instance

## Prerequisites

* A [Google cloud account](https://console.cloud.google.com/?hl=en-au)
* [Google Cloud CLI](https://cloud.google.com/sdk/docs/install-sdk#deb)
* [Ansible](https://www.cyberciti.biz/faq/how-to-install-and-configure-latest-version-of-ansible-on-ubuntu-linux/)
* [Terraform](https://developer.hashicorp.com/terraform/cli/install/apt)
* [Python](https://docs.python-guide.org/starting/install3/linux/#install3-linux)

## Install Redis on GCP Arm based instance

To install Redis follow this [documentation](/content/learning-paths/server-and-cloud/redis/ec2_deployment.md).

## Deploy Redis as a cache for MySQL on a single AWS Arm based instance

## Deploy AWS Arm based instance via Terraform

Before deploying AWS Arm based instance via Terraform, generate [Access keys](/content/learning-paths/server-and-cloud/mysql/ec2_deployment.md#generate-access-keys-access-key-id-and-secret-access-key) and [key-pair using ssh keygen](/content/learning-paths/server-and-cloud/mysql/ec2_deployment.md#generate-key-pairpublic-key-private-key-using-ssh-keygen).

Add resources required to create a virtual machine in **main.tf** file.
```
provider "google" {
  project = "{project_id}"
  region = "us-central1"
  zone = "us-central1-a"
}

resource "google_compute_project_metadata_item" "ssh-keys" {
  key   = "ssh-keys"
  value = "ubuntu:${file("{public_key_location}")}"
}

resource "google_compute_firewall" "rules" {
  project     = "{project_id}"
  name        = "my-firewall-rule"
  network     = "default"
  description = "Open Redis connection port"
  source_ranges = ["0.0.0.0/0"]

  allow {
    protocol  = "tcp"
    ports     = ["6000"]
  }
}

resource "google_compute_instance" "vm_instance" {
  name         = "vm_name"
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

resource "local_file" "inventory" {
    depends_on=[google_compute_instance.vm_instance]
    filename = "inventory.txt"
    content = <<EOF
[all]
ansible-target1 ansible_connection=ssh ansible_host=${google_compute_instance.vm_instance.network_interface.0.access_config.0.nat_ip} ansible_user=ubuntu
                EOF
}
```
**NOTE:-** Replace `{project_id}` and `{public_key_location}` with respective values.

## Terraform commands

To deploy the instances, we need to initialize Terraform, generate an execution plan and apply the execution plan to our cloud infrastructure. Follow this [documentation](/content/learning-paths/server-and-cloud/redis/ec2_deployment.md#terraform-commands) to deploy the **main.tf** file.

## Install Redis and MySQL through Ansible

## Deploy Redis as a cache for single MySQL database on a single AWS Arm based instance using Python

## Deploy Redis as a cache for MySQL on two AWS Arm based instances

