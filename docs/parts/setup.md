<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 1: Setup

> **_Note:_**  This workshop is intended to detail the manual installation process and the tools available to run and manage EKS clusters on Equinix Metal. However, a **Terraform module** is available to automate the entire setup process. If you're more interested in that approach, you can follow the [terraform installation instructions](https://github.com/equinix-labs/terraform-equinix-metal-eks-anywhere#using-terraform) and skip parts 1-3 from this workshop.


To run this workshop you will need access to an Equinix Metal Account or create a new one following step 1 below.

> **_Note:_** You are responsible for the cost of resources created in your Equinix Metal account while running this workshop.

## Pre-requisites

The following tools will be needed on your local development environment where you will be running most of the commands in this guide.

* A Unix-like environment (Linux, OSX, [Windows WSL](https://docs.microsoft.com/en-us/windows/wsl/install))
* [jq](https://stedolan.github.io/jq/download/)
* [metal-cli](https://github.com/equinix/metal-cli) (v0.9.0+)
  
## Steps

### 1. Create an Equinix Metal API key (token)

If you have never used Equinix Metal before, please follow the [Create an Equinix Metal account](https://equinix-labs.github.io/terraform-on-equinix-workshop/parts/install/#1-create-an-equinix-metal-account) 1,2 and 4 steps or you can watch our [Getting Started with Equinix Metal](https://www.youtube.com/watch?v=5Ax6fKBeg2U&t=153s) video.

### 2. Install and Configure Metal CLI

Once you are familiar with the console you may feel more comfortable managing your Equinix Metal resources with the command-line interface tool [Metal-cli](https://github.com/equinix/metal-cli).

Once installed take [API Key](https://console.equinix.com/users/-/api-keys) from step 1 and register it:

Execute `metal init` and provide the requested information:

```shell
$ metal init

Equinix Metal API Tokens can be obtained through the portal at https://console.equinix.com/.
See https://metal.equinix.com/developers/docs/accounts/users/ for more details.

Token (hidden):
Organization ID [27703148-e7bf-4a2f-95cf-46e7dddb4bb8]:
Project ID []:

Writing /Users/You/.config/equinix/metal.yaml
```

### 3. Verify

```shell
$ metal organization get

+--------------------------------------+-------------------------------------+----------------------+
|                  ID                  |                NAME                 |       CREATED        |
+--------------------------------------+-------------------------------------+----------------------+
| 27703148-e7bf-4a2f-95cf-46e7dddb4bb8 | Equinix Terraform                   | 2023-01-01T00:00:00Z |
+--------------------------------------+-------------------------------------+----------------------+
```

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* Is there any automated deployment option?
