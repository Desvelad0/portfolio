---
title: "Setting up Sentinel with Terraform"
tags: [azure,terraform]
date: 2025-02-18T21:58:32+02:00
draft: true
---

This past weekend I went to Disobey, which if you are reading this and have not gone, I implore you to do so next year, it was a great experience for anyone interested in cyber security and any aspiring sticker collector.

At the event, I spoke with several people regarding employment in the field, and many mentioned how important it is to be comfortable using Microsoft Sentinel, as it is often used as the collection point for any logs an environment produces and configured to generate alerts depending on the needs of a company.  I have previously used Sentinel to monitor and act on security alerts, but thought it would be a good exercise to set up the tool from scratch in my own environment, and while doing so, learn more about how to use Terraform for better cloud automation.

## Prerequisites

As I'm currently on Windows, I like to use `winget` to install and update any software that I can. To install Terraform I need to run `winget install terraform` which prompts me with a list of options, of which I choose `Hashicorp.Terraform`. 

I also need to install the Azure CLI tools which after a quick [googlin](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?pivots=winget)' can also be installed with winget:

``winget install --exact --id Microsoft.AzureCLI`` 

Then (after opening and re-opening my PowerShell window) I need to login to my Azure account by running `az login`

I then found a good [freeCodeCamp video](https://www.youtube.com/watch?v=V53AHWun17s) tutorial on on the topic which I will be following to learn about the use of Terraform.

First, I have to make a `main.tf` file, which will look something like this:

```
terraform {
    requried_providers {
        azurerm = {
            source = "hashicorp/azurerm"
            version = "=3.0.1"
        }
    }
}

provider "azurerm" {
    feature{}
}
```

 
