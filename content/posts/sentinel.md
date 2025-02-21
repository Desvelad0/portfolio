---
title: "Getting started with Sentinel and automating with Terraform"
tags: [azure,terraform]
date: 2025-02-18T21:58:32+02:00
draft: false
---

This past weekend I went to Disobey, which if you are reading this and have not gone, I implore you to do so next year, it was a great experience for anyone interested in cyber security and any aspiring sticker collector.

At the event, I spoke with several people regarding employment in the field, and many mentioned how important it is to be comfortable using Microsoft Sentinel, as it is often used as the collection point for any logs an environment produces and configured to generate alerts depending on the needs of a company.  I have previously used Sentinel to monitor and act on security alerts, but thought it would be a good exercise to set up the tool from scratch in my own environment, and while doing so, learn more about how to use Terraform for better cloud automation.

## Prerequisites

As I'm currently on Windows, I like to use `winget` to install and update any software that I can. To install Terraform I need to run `winget install terraform` which prompts me with a list of options, of which I choose `Hashicorp.Terraform`. 

I also need to install the Azure CLI tools which after a quick [googlin](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?pivots=winget)' can also be installed with winget:

``winget install --exact --id Microsoft.AzureCLI`` 

Then (after closing and re-opening my PowerShell window) I need to login to my Azure account by running `az login` and apparently need to append `--tenant <tenantid>` to get to the proper login screen for my tenant and have access to that tenant's resources:

![loginscreen.png](/azure/loginscreen.png)

After that, I had to pick the subscription I wanted to use for this project and that's it.

I then found a good [freeCodeCamp video](https://www.youtube.com/watch?v=V53AHWun17s) tutorial on on the topic which I will be following to learn about the use of Terraform.

First, I have to make a `main.tf` file, which will look something like this:

```
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "=3.0.1"
    }
  }
}

provider "azurerm" {
  features {}
}

```

 Apparently freezing the version makes it easier to deal with any possible errors in newer versions; if one version is known to work, sticking to that might be useful in case a newer version breaks things.

Running Terraform inside VSCodium, I can handily run `terraform fmt` from a terminal to properly format the file I'm currently working on, assuming I have opened the working directory where that file is located inside the editor.

I can add a `resource` to make things:

```
resource "azurerm_resource_group" "sentinel" {
  name     = "sentinel"
  location = "West Europe"
  tags = {
    environment = "dev"
  }
}
```

Then I can run `terraform plan` to see what the current code would do:

```
PS C:\Users\hackerman\Documents\VScodium\terraform> terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_resource_group.sentinel will be created
  + resource "azurerm_resource_group" "sentinel" {
      + id       = (known after apply)
      + location = "westeurope"
      + name     = "sentinel"
      + tags     = {
          + "environment" = "dev"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── 

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
```

To then actually perform the actions that the code would do, I can run `terraform apply`:

```
PS C:\Users\hackerman\Documents\VScodium\terraform> terraform apply   

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # azurerm_resource_group.sentinel will be created
  + resource "azurerm_resource_group" "sentinel" {
      + id       = (known after apply)
      + location = "westeurope"
      + name     = "sentinel"
      + tags     = {
          + "environment" = "dev"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

azurerm_resource_group.sentinel: Creating...
azurerm_resource_group.sentinel: Still creating... [10s elapsed]
azurerm_resource_group.sentinel: Creation complete after 11s [id=/subscriptions/idwashere/resourceGroups/sentinel]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Success! 

![sentinel](/azure/terraform.png)

## Onboarding Windows desktop PC to Sentinel

Now like it often seems to be the case with my projects, this is when it started getting tricky. My original plan was to get my gaming PC's and pfSense's logs ingested into Sentinel, which in theory should be simple but requires some research if you haven't done it before. 

In order to get Sentinel working to begin with, you have to have a log analytics workspace. Then, you need to connect Sentinel to a workspace, after which you can start adding data connectors that actually allow you start ingesting data to be dealt with in Sentinel. For Windows devices, it seems like there are multiple ways of having my logs be ingested into Sentinel, and to me the easiest way seemed like to install Azure Arc:

```Connect-AzConnectedMachine
Connect-AzConnectedMachine -ResourceGroupName sentinel-demo -Name <computername> -Location "West Europe"
```

Then I needed to get the Azure Monitor Agent installed:

```
New-AzConnectedMachineExtension -Name AMAWindows -ExtensionType AzureMonitorWindowsAgent -Publisher Microsoft.Azure.Monitor -ResourceGroupName sentinel-demo -MachineName <computername> -Location "West Europe"
```

At this stage, my PC is connected to Azure, and browsing my computer in the GUI, I begin to see some Heartbeat logs for it, but nothing else yet. I needed to make data collection rules.

I made a Data Collection Rule (DCR) where I had to add my computer as a resource, and then had to add "Windows Event Logs" as a data source. These are among the only logs that can be ingested into Sentinel without a Data Collection Endpoint as it turns out.

After making the rule, slowly but surely logs started to flow into Sentinel:

![sentinelpclogs](/azure/sentinelpclogs.png)

From these I made a few rules such as one for Login Failures:

``` 
Event
| where EventID == 4625
| extend FailedAccount = extract(@"Account For Which Logon Failed:\s+Security ID:\s+\S+\s+Account Name:\s+(\S+)", 1, RenderedDescription)
| project TimeGenerated, FailedAccount, Computer
| sort by TimeGenerated desc
```

This makes some neat data for local log in errors to my computer. Based on this rule I also made a simple automation to send an email if there's three failed logins in rapid succession and then assigns the alert to security personnel:

![logicapp](/azure/logicapp.png)

![automation](/azure/automation.png)

I wanted to make the alert email all neat and beautiful, but turns out the provider where I hosted this email address (for free) doesn't really like it when you send a lot of emails in rapid succession. I was going to set up SendGrid for this purpose since they apparently have a decent free tier, but at the time of writing, their sign up page was down. Oh well, something to continue with later.

## Automate it all

Since I had everything set up the way I wanted it, I wanted a way to quickly deploy and delete it all, just to get a feel for Terraform but also to make everything a lot faster. 

Deleting everything: 

```
terraform destroy
```

After agreeing to the destroying of all things, some things still remained:

```
PS C:\Users\hackerman\Documents\VScodium\terraform> terraform state list
azurerm_log_analytics_workspace.sentinel_workspace
azurerm_resource_group.sentinel-rg
```

These were causing some issues in re-deploying, therefore they can be deleted by doing ``terraform state rm``. I even had to manually still delete my existing log analytics work space (which deletes everything within):

``az monitor log-analytics workspace delete --resource-group sentinel-demo --workspace-name sentinel-demo-ws`` 

And even then, still something is left behind:

```
│ Error: A resource with the ID "/subscriptions/subid/resourceGroups/sentinel-demo/providers/Microsoft.OperationalInsights/workspaces/sentinel-demo-ws/providers/Microsoft.SecurityInsights/onboardingStates/default" already exists - to be managed via Terraform this resource needs to be imported into the State. Please see the resource documentation for "azurerm_sentinel_log_analytics_workspace_onboarding" for more information.
```

```
az resource show --ids "/subscriptions/subid/resourceGroups/sentinel-demo/providers/Microsoft.OperationalInsights/workspaces/sentinel-demo-ws/providers/Microsoft.SecurityInsights/onboardingStates/default"
```

Getting rid of it for good:

```
az resource delete --ids "/subscriptions/subid/resourceGroups/sentinel-demo/providers/Microsoft.OperationalInsights/workspaces/sentinel-demo-ws/providers/Microsoft.SecurityInsights/onboardingStates/default"
```

... I'll spare you the rest. I essentially kept having an issue where despite destroying everything from the environment, remnants would be left over, and additionally the analytics rules would keep referencing some log analytics workspace that doesn't even exist, either due to me being impatient or the version of terraform having a bug; for whatever reason, when I switched to using version 4.19.0, I stopped having these issues, and can get a simple environment going with the following setup (with the help of ChatGPT, the aforementioned freeCodeCamp video and Terraform documentation):

https://github.com/Desvelad0/scripts/blob/main/azure/sentineldemo.tf

The only thing left to do manually with this script is to enroll my Windows PC with Azure Arc and Azure Monitor Agent, and after that add the computer into the data collection rule (in Configuration > Resources) made by the script.

## Wrapping up

As is often the case, I started off with an ambitious goal for my project here, wanting to not only ingest logs from Windows but also pfSense, but as it turns out, the latter is not as simple as I might have thought. In addition to the issues I was having with destroying and re-deploying my infrastructure, I thought it best to wrap up here with a proof of concept and continue later on with more additions; I will definitely be making more rules, adding more data connectors and ingesting data from more sources, and definitely getting pfSense logs into Sentinel. 

