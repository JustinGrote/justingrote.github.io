---
title: Using Azure Artifacts As a Private Powershell Gallery
classes: wide
toc: true
---

Azure DevOps (formerly VSTS) has a component called Azure Artifacts (formerly VSTS Packages) that lets you publish NuGet feeds. While typically used for .NET assemblies, Powershell Gallery (based on PackageManagement) uses NuGet feeds in the background, and it turns out these are compatible

Why would you want to do this? Maybe you have a company-proprietary module that you don't want to share on Powershell Gallery, or perhaps you want to keep a curated subset of modules from Powershell Gallery that you know are tested and stable so you can distribute them among systems or peers.

Thankfully, it's pretty easy!

Note: Much of this process comes from [this excellent article](https://roadtoalm.com/2017/05/02/using-vsts-package-management-as-a-private-powershell-gallery/). I've just simplified it a bit.

## Getting Started

I assume you already have an Azure Devops Account.

1. First [Create the Feed](https://docs.microsoft.com/en-us/azure/devops/artifacts/get-started-nuget?view=vsts&tabs=new-nav#create-a-feed) 
2. Get your repository URL from step 3 of [Publish a Package](https://docs.microsoft.com/en-us/azure/devops/artifacts/nuget/nuget-exe?view=vsts&tabs=new-nav#add-a-feed-to-nuget-2). It should be something like: `https://pkgs.dev.azure.com/YOURCOMPANY/_packaging/MYPROJECT/nuget/v3/index.json`
3. Powershell Package Management as of October 2018 still only supports version 2 nuget feeds, so [change the "/v3/index.json" in the URL to just "/v2/"](https://docs.microsoft.com/en-us/azure/devops/artifacts/nuget/nuget-exe?view=vsts&tabs=new-nav#add-a-feed-to-nuget-2): `https://pkgs.dev.azure.com/YOURCOMPANY/_packaging/MYPROJECT/nuget/v2/`

## Connecting to Your Feed
Now you have a feed to work with! There are a few ways you can authenticate to your feed and start publishing packages.

### Recommended Way - Azure Artifacts Credential Provider
The [Azure Artifacts Credential Provider for NuGet](https://github.com/Microsoft/artifacts-credprovider) automates the process of fetching a [Personal Access Token (PAT)](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=vsts) in a friendly way, and supports two-factor authentication. 

#### Install Azure Artifacts Credential Provider
To install automatically, run the following powershell deployment script (provided by the Azure DevOps team). *Always Review Any Script Before You Run It!*
```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force;  iex (iwr 'https://raw.githubusercontent.com/Microsoft/artifacts-credprovider/master/helpers/installcredprovider.ps1').content
```

You can also [install manually](https://github.com/Microsoft/artifacts-credprovider#manual-installation-windows) if you prefer.
