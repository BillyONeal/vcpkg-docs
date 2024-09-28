---
title: "Tutorial: Set up a vcpkg binary cache using Azure Storage, Azure DevOps, and Workload Identity Federation"
description: This tutorial shows how to set up a vcpkg binary cache using an Azure Storage account authenticated using Workload Identity Federation.
author: bion
ms.author: bion
ms.topic: tutorial
ms.date: 9/26/2024
zone_pivot_group_filename: zone-pivot-groups.json
zone_pivot_groups: shell-selections
---

# Tutorial: Set up a vcpkg binary cache using Azure Storage, Azure DevOps, and Workload Identity Federation

vcpkg supports using Azure Storage containers to upload and restore binary packages. However,
configuring pre-shared long-lived SAS tokens for authentication purposes requires error-prone
manual secrets rotation. [Workload Identity Federation](/entra/workload-id/workload-identity-federation)
offers a mechanism where Azure DevOps takes care of secrets rotation and management for you
that can be used with vcpkg.

In this tutorial, you'll learn how to:

> [!div class="checklist"]
>
> * [Create an Entra Managed Identity](#1---create-an-entra-managed-identity)
> * [Set up Workload Identity Federation between Azure Storage and Azure DevOps](#1---set-up-a-nuget-feed)
> * [Generate a SAS token to authenticate with Azure Storage](#2---add-a-nuget-source)
> * [Configure vcpkg to Azure Storage with the SAS token](#3---configure-vcpkg-to-use-your-nuget-feed)

## Prerequisites 

::: zone pivot="shell-powershell"

* A terminal with [Azure PowerShell](/powershell/azure/install-azps-windows) installed
* [vcpkg](../get_started/get-started.md#1---set-up-vcpkg)
* An Azure Storage subscription and container where you wish binaries to be stored

::: zone-end

::: zone pivot="shell-cmd, shell-bash"

* A terminal with [Azure CLI](/cli/azure/install-azure-cli-windows) installed
* [vcpkg](../get_started/get-started.md#1---set-up-vcpkg)
* An Azure Storage subscription and container where you wish binaries to be stored

::: zone-end

## 1 - Create an Entra Managed Identity

Skip this step if you already have configured a
[Managed Identity](/entra/identity/managed-identities-azure-resources/overview) you wish to use.

In the Azure Portal, go to the page for a Resource Group in which you wish for the managed identity
to be created, and press "+ Create". In the search box, enter "User Assigned Managed Identity".
Choose "User Assigned Managed Identity" from "Microsoft" and choose "Create ->
User Assigned Managed Identity". Name the identity, press "Review and Create", and "Create". For
purposes of this documentation, the identity was named `vcpkg-docs-identity`.

## 2 - Set Up Workload Identity Federation with Azure DevOps

In the Azure DevOps portal for the project in which you wish to run Pipelines, select
"Project Settings" in the lower left corner. Then select "Service Connections" on the left.
Then choose "New Service Connection" on the right. Choose the "Azure Resource Manager" radio button,
and press Next. Select "Workload identity federation (manual)" and press next. Name the
service connection and press next. For purposes of this documentation, the service connection was
named `vcpkg-docs-identity-connection`. At this point, Azure DevOps should be showing an issuer
and subject identifier.

In another tab, to the Azure Portal navigate to the managed identity created in step 1. On the left
select Settings/Federated Credentials, and select 'Add Credential'. In the drop down, select
'Other'. Copy the 'Issuer URL' and 'Subject Identifer' from Azure DevOps into the form. Give the
federated credential a name, and press 'Add'. For purposes of this documentation, the name used was
`azure-devops-credential`.

## 3 - Assign Managed Identity Permissons to the Storage Account Container

Skip this step if you have already granted your managed identity permissions to your Storage Account
Container.

In the Azure Portal, go to the page for the Storage Account to use. On the left, select
Data Storage/Containers, and choose the container you wish to use. This should open the Azure Portal
to the properties for that specific container. On the left select "Access Control (IAM)", and choose
"Add -> Add Role Assignment". Select "Azure Blob Data Reader" for read only access, or
"Azure Blob Data Contributor" for read/write access, and press Next. Then, select the
"Managed Identity" radio button, and click + Select Members. Select the managed identity you created
in step 1. Then select 'Review and Assign' twice.

Then, return to the page for the storage account container, and again on the left select
"Access Control (IAM)", and choose "Add -> Add Role Assignment". Select "Storage Blob Delegator" and
press Next. Then, select the "Managed Identity" radio button, and click + Select Members. Select the
managed identity you create in step 1. Then select 'Review and Assign' twice.


## 2 - Add a NuGet source

::: zone pivot="shell-bash"
> [!NOTE]
> On Linux you need `mono` to execute `nuget.exe`. You can install `mono` using your distribution's
> system package manager.
::: zone-end

vcpkg acquires its own copy of the `nuget.exe` executable that it uses during binary caching
operations. This tutorial uses the vcpkg-acquired `nuget.exe`. The `vcpkg fetch nuget` command
outputs the location of the vcpkg-acquired `nuget.exe`, downloading the executable if necessary.

Run the following command to add your NuGet feed as a source, replace `<feed name>` with any name of
your choosing and `<feed url>` with the URL to your NuGet feed.

::: zone pivot="shell-powershell"

```PowerShell
.$(vcpkg fetch nuget) sources add -Name <feed name> -Source <feed url>
```

::: zone-end
::: zone pivot="shell-cmd"

Execute the command below to fetch the path to the NuGet executable:

```console
vcpkg fetch nuget
```

This will provide an output that looks something like `C:\path\to\nuget.exe`. Make a note of this path.
Using the path obtained from the previous step, run the following command:

```console
C:\path\to\nuget.exe sources add -Name <feed name> -Source <feed url>
```

::: zone-end
::: zone pivot="shell-bash"

```bash
mono `vcpkg fetch nuget | tail -n 1` sources add -Name <feed name> -Source <feed url>
```

::: zone-end

### Provide an API key

Some providers require that you push your NuGet packages to the feed using an API key. For example,
GitHub Packages requires a GitHub PAT (Personal Access Token) as the API key; if you're using Azure
Artifacts the API key is `AzureDevOps` instead.

Use the following command to set the API key for all the packages pushed to your NuGet feed, replace
`<apiKey>` with your feed's API key.

::: zone pivot="shell-powershell"

```PowerShell
.$(vcpkg fetch nuget) setapikey <apikey> -Source <feed url>
```

::: zone-end
::: zone pivot="shell-cmd"

Execute the command below to fetch the path to the NuGet executable:

```console
vcpkg fetch nuget
```

This will provide an output that looks something like `C:\path\to\nuget.exe`. Make a note of this path.
Using the path obtained from the previous step, run the following command:

```console
C:\path\to\nuget.exe setapikey <apikey> -Source <feed url>
```

::: zone-end
::: zone pivot="shell-bash"

```bash
mono `vcpkg fetch nuget | tail -n 1` sources setapikey <apiKey> -Source <feed url>
```

::: zone-end

### Provide authentication credentials

Your NuGet feed may require authentication to let you download and upload packages. If that's the
case you can provide credentials by adding them as parameters to the `nuget sources add` command.

For example:

```Console
nuget sources add -Name my-packages -Source https://my.nuget.feed/vcpkg-cache/index.json -UserName myusername -Password mypassword -StorePasswordInClearText
```

Some providers like Azure Artifacts may require different authentication methods, read the 
[Authenticate to private NuGet feeds](../reference/binarycaching.md#nuget-credentials) article to learn
more.

### Use a `nuget.config` file

Alternatively, you can use a `nuget.config` file to configure your NuGet sources, following the
template below:

`nuget.config`

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <config>
    <add key="defaultPushSource" value="<feed url>" />
  </config>
  <apiKeys>
    <add key="<feed url>" value="<apikey>" />
  </apiKeys>
  <packageSources>
    <clear />
    <add  key="<feed name>" value="<feed url>" />
  </packageSources>
  <packageSourcesCredentials>
    <<feed name>>
      <add key="Username" value="<username>" />
      <add key="Password" value="<password>" />
    </<feed name>>
  </packageSourcesCredentials>
</configuration>
```

Example `nuget.config` file :

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <config>
    <add key="defaultPushSource" value="https://contoso.org/packages/" />
  </config>
  <apikeys>
    <add key="https://contoso.org/packages/" value="encrypted_api_key" />
  </apikeys>
  <packageSources>
    <clear />
    <add key="Contoso" value="https://contoso.org/packages/" />
  </packageSources>
  <packageSourcesCredentials>
    <Contoso>
      <add key="Username" value="user" />
      <add key="Password" value="..." />
    </Contoso>
  </packageSourcesCredentials>
</configuration>
```

vcpkg requires that you set a `defaultPushSource` in your `nuget.config` file, use your NuGet feed's
URL as the default source to push binary packages. 

If you're uploading your packages to an Azure Artifacts NuGet feed, use `AzureDevOps` as your
source's API Key by running `nuget setApiKey AzureDevOps -Source <feed url> -ConfigFile <path to nuget.config>`.
Otherwise, replace the value with your feed's proper API Key if you have one.

Add the `<clear />` source to ignore other previously configured values. If you want, you can define multiple
sources in this file, use a `<add key="<feed name>" value="<feed url>" />` entry for each source.

Run the following command to add a NuGet source using a `nuget.config` file, replace 
`<path to nuget.config>` with the path to your `nuget.config` file:

::: zone pivot="shell-powershell"

```PowerShell
.$(vcpkg fetch nuget) sources add -ConfigFile <path to nuget.config>
```

::: zone-end
::: zone pivot="shell-cmd"

Execute the command below to fetch the path to the NuGet executable:

```console
vcpkg fetch nuget
```
This will provide an output that looks something like `C:\path\to\nuget.exe`. Make a note of this path.
Using the path obtained from the previous step, run the following command:

```console
C:\path\to\nuget.exe sources add -ConfigFile <path to nuget.config>
```

::: zone-end
::: zone pivot="shell-bash"

```bash
mono `vcpkg fetch nuget | tail -n 1` sources add -ConfigFile <path to nuget.config>
```

::: zone-end

## 3 - Configure vcpkg to use your NuGet feed

Set the `VCPKG_BINARY_SOURCES` environment variable as follows:

::: zone pivot="shell-powershell"

```PowerShell
$env:VCPKG_BINARY_SOURCES="clear;x-azblob,<feed url>,readwrite"
```

::: zone-end
::: zone pivot="shell-cmd"

```console
set "VCPKG_BINARY_SOURCES=clear;nuget,<feed url>,readwrite"
```

If you're using a `nuget.config` file, instead do:

```console
set "VCPKG_BINARY_SOURCES=clear;nugetconfig,<path to nuget.config>"
```

::: zone-end
::: zone pivot="shell-bash"

> [!NOTE]
> Setting `VCPKG_BINARY_SOURCES` using the `export` command will only affect the current shell
> session. To make this change permanent across sessions, you'll need to add the `export` command to
> your shell's profile script (e.g., `~/.bashrc` or `~/.zshrc`).

```bash
export VCPKG_BINARY_SOURCES="clear;nuget,<feed url>,readwrite"
```

If you're using a `nuget.config` file, instead do:

```bash
export VCPKG_BINARY_SOURCES="clear;nugetconfig,<path to nuget.config>"
```

::: zone-end

And that's it! vcpkg will now upload or restore packages from your NuGet feed.

## Next steps

Here are other tasks to try next:

* [Change the default binary cache location](binary-caching-default.md)
* [Set up a local binary cache](binary-caching-local.md)
