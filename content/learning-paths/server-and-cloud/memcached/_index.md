---
title: Run memcached on Arm servers

description: Learn how to install memcached on Arm servers

minutes_to_complete: 40

who_is_this_for: This is an introductory topic for software developers who want to use memcached as their in-memory key-value store for mobile, web, gaming or e-Commerce applications.

learning_objectives:
    - Install and run memcached on your Arm-based cloud server
    - Use an open-source benchmark to test memcached performance
    - Deploy memcached as a cache for MySQL

prerequisites:
    - An Amazon Web Services (AWS) [account](https://aws.amazon.com/)
    - An Azure portal [account](https://azure.microsoft.com/en-in/get-started/azure-portal)
    - A Google Cloud [account](https://console.cloud.google.com/)
    - A machine with [Terraform](/install-guides/terraform/), [AWS CLI](/install-guides/aws-cli), [Google Cloud CLI](/install-guides/gcloud), [Azure CLI](/install-guides/azure-cli), [AWS IAM authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html), and [Ansible](/install-guides/ansible/) installed

author_primary: Pareena Verma

### Tags
skilllevels: Advanced
subjects: Web
armips:
    - Neoverse
tools:
softwares:
    - Memcached
operatingsystems:
    - Linux

### Test
test_images:
- ubuntu:latest
test_link: https://github.com/armflorentlebeau/arm-learning-paths/actions/runs/4312122327
test_maintenance: true
test_status:
- passed

### FIXED, DO NOT MODIFY
# ================================================================================
weight: 1                       # _index.md always has weight of 1 to order correctly
layout: "learningpathall"       # All files under learning paths have this same wrapper
learning_path_main_page: "yes"  # This should be surfaced when looking for related content. Only set for _index.md of learning path content.
---
