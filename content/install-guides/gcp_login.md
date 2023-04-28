---
### Title the install tools article with the name of the tool to be installed
### Include vendor name where appropriate
title: Google Cloud Credentials

### Optional additional search terms (one per line) to assist in finding the article
additional_search_terms:

### Estimated completion time in minutes (please use integer multiple of 5)
minutes_to_complete: 10

author_primary: Jason Andrews

### Combined with gcloud install. This install guide to be deleted.

### Link to official documentation
official_docs: https://cloud.google.com/sdk/gcloud/reference/auth/application-default/login

### PAGE SETUP
weight: 1                       # Defines page ordering. Must be 1 for first (or only) page.
tool_install: true              # Set to true to be listed in main selection page, else false
multi_install: false            # Set to true if first page of multi-page article, else false
multitool_install_part: false   # Set to true if a sub-page of a multi-page article, else false
layout: installtoolsall         # DO NOT MODIFY. Always true for tool install articles
---

In this section you will learn how to obtain user access credentials for Google Cloud using a web flow. You will put the credentials in a well-known location for Application Default Credentials (`ADC`).

## Before you begin

Install the `Google Cloud CLI` on your machine using the [install guide](/install-guides/gcloud/).

## Acquire user credentials

Run the following command to obtain user access credentials::
```console
gcloud auth application-default login
```

A URL is generated as the output of the command:

![image #center](https://user-images.githubusercontent.com/67620689/204504640-c49c0b0d-6a59-4915-ac3a-f03614783d98.PNG)

Open the URL in the browser and copy the authentication code.

![image #center](https://user-images.githubusercontent.com/67620689/204244780-6c0542ab-4240-4be3-8272-fb1e6e38ec08.PNG)

Now paste the authentication code as shown below:

![image #center](https://user-images.githubusercontent.com/67620689/204242841-58e30570-1f88-4755-b3d2-32d7052a9b5d.PNG)

After a successful log in, you will be able to use the `Google Cloud CLI` and automation tools like [Terraform](../terraform) from the terminal.
