---
title: Powershell Azure Functions The Missing Manual
---
# Powershell Azure Functions: The Missing Manual

Recently the Azure Functions runtime was updated to run on Windows Server 2016. This means that Powershell now runs 5.1 instead of 4.0 and there is a lot of benefit that comes from this that may not be documented yet, hence I am documenting it here.

This article is targeted towards people who have Powershell experience, and need to know how the behavior is different from a normal Powershell 5.1 environment.

* TOC
{:toc}

## Why Azure Functions over Azure Automation

Azure Automation is a much more mature Powershell environment in Azure. Azure Functions is still considered experimental. So why would you want to use it?

1. **It's Dirt Cheap**: Azure Functions gives you 400,000 GB-s and 1 million executions for free every month under the Consumption plan. Azure Automation only gives you 500 minutes. You can build an incredibly advanced Powershell function workflow and API environment and never get billed a single dime.
2. **It's Fast**: With Azure Automation you usually have to wait for the job to be scheduled, queued, etc. and the feedback loop is very slow, taking often several minutes before your job runs and you see the result. Azure Functions start in seconds and scale nearly limitlessly and automatically, making it far easier to test and troubleshoot.
3. **It's better for APIs**: Azure Functions makes it really easy to make a query with a URL or trigger on an object, queue input, OneDrive for Business file upload, etc., take that input into Powershell, do Powershell-y things to it, and output it back out as JSON, XML, or whatever you want. You can build great APIs with Powershell very rapidly, why should the C# and Java guys have all the fun? When you first create a Powershell Azure HTTP Trigger Function, the sample code is basically a REST API example written in Powershell that is ready-to-go for you. [Brian Bunke has written a good "getting started" article on writing REST APIs in Azure Functions Powershell.](http://www.brianbunke.com/blog/2018/02/26/serverless-api-in-azure/)

## What can you do with Azure Functions

This is a running list of examples:

* [Run a SQL Cleanup task on a timer](https://dfinke.github.io/2017/Use-PowerShell-in-Azure-Functions-to-perform-a-scheduled-clean-up-task/)
* [Extend Slack to get stock data via Yahoo](https://dfinke.github.io/2017/Use-Serverless-PowerShell-to-Extend-Slack/)

## Azure Functions Environment

Azure Functions operates basically by running on top of [Azure Web Jobs](https://buildazure.com/2017/03/08/azure-functions-vs-web-jobs-how-to-choose/) which in turn run on [Azure App Service](https://azure.microsoft.com/en-us/services/app-service/). As a result, [the same OS enviroment restrictions apply](https://docs.microsoft.com/en-us/azure/app-service/web-sites-available-operating-system-functionality).

### Limitations

#### Consumption Plan Job Runtime Timeout

The biggest limitation is that under the free Consumption (Dynamic) plan, jobs will timeout if they take longer than 5 minutes to execute by default. [You can increase that up to 10 minutes by editing the host JSON](https://buildazure.com/2017/08/17/azure-functions-extend-execution-timeout-past-5-minutes/). However, Azure Functions are generally meant to be designed to be "small and fast" so if your process takes longer than 10 minutes, consider breaking it into chunks and connecting them via something like Azure Queues, or run a control script on Azure Automation, a function on a dedicated App Service plan, or a Windows VM.

#### Consumption Plan 1.5GB Memory Limit

As of this writing, the Consumption plan instances only provide 1.5GB of memory, so if your function requires more than that, either refactor it to store the data somewhere temporarily and flush memory, or place it on a dedicated App Service Plan instance with more memory.

#### Logs Panel and Verbose/Debug Output

Azure Functions have a built in log streaming service panel. For powershell, unlike the normal powershell console, this log streaming will only show the Output and Error streams and will not show the Debug, Verbose, or Host streams at all, so in other words Write-Host, Write-Debug, and Write-Verbose will show nothing in this log. The DebugPreference and VerbosePreference variables also have no effect on this.

These same logs are visible at **D:\home\LogFiles\Application\Functions** in the app service instance as well.

In order to see the verbose output of a command, you need to use the stream redirection commands. After you understand how [Streams and Redirection work in Powershell](https://blogs.technet.microsoft.com/heyscriptingguy/2014/03/30/understanding-streams-redirection-and-write-host-in-powershell/), you can see the verbose output of a command with:

~~~Powershell
Write-Verbose "This is Verbose Text" 4>&1
~~~

Where 4>&1 means take the Verbose stream (numbered 4), and redirect it to the output stream which shows in the log. You can do this for warning (3), Error (2), and Debug (5) as well.

In addition, the logs do not auto-format Powershell Objects the way the normal Powershell host does for you. If you want to display an object in the log as a textual representation other than its object type, it is recommended to display it using the following format:

~~~Powershell
$PSObjectToDisplay | Format-List | Out-String
~~~

You can also use Select-Object or Format-Table to select only certain properties.

### Home and Local Drive Directories

In Azure Functions you have two primary local storage directories available to you: **D:\Home** and **D:\Local**.

**D:\Home** is your persistent area where files are kept between multiple runs. It is backed by an Azure storage account. The code for your azure function is stored in this location, generally in `D:\home\site\wwwroot\<functionname\`

**D:\Local** is where temporary files are stored and are not persistent between multiple function instances. You can use this area to store files that are specific to a run, for instance temporary data you've downloaded from another source or data you want to dump out. Keep in mind that these files may or may not be present depending on if your next function run started on the same instance or not (which depends on the type of App Service plan you chose), so ensure if you write scripts that write to the temp that you overwrite anything in place, and optionally clean up any temp files you use, so that you don't get junk from one run affecting another run.

### Special Azure Functions Directories

1. **Function Directory:** This is where your function files live as you see them in the "View Files" tab. The path is stored in the **$EXECUTION_CONTEXT_FUNCTIONDIRECTORY** powershell variable. This location is **D:\home\site\wwwroot\\\<functionname\>**, where \<functionname\> is the name of your Azure function. For instance, if you name your function Sandbox, the path is *D:\home\site\wwwroot\Sandbox*. I recommend running `Set-Location $EXECUTION_CONTEXT_FUNCTIONDIRECTORY` as one of your first function activities so your relative commands can call other resources in your function directory for ease of use.
2. **Modules Directory:** Inside of your function directory, you can create a directory called "modules" which Azure Functions will autoload any powershell modules that are part of your script. While this is an easy option, I generally recommend not using this, and instead loading modules as you need them using Powershell 5.1's built-in loading logic to optimize performance. For instance, if your full function package has a library of 50 modules, maybe only 1 or 2 need to be run for any individual invocation, so no need to take up unnecessary time loading a bunch of modules you won't use.

## Variables

### Environment Variables

All the environment variables in Azure Functions that you can use can be seen with:

~~~powershell
Get-Item env: | Format-Table -AutoSize | Out-String
~~~

The format table and out-string are necessary because Functions by default will output the object as its object type, this gives it a more "powershell-y" output to the Function log.

Some notable Azure Function specific environmental variables:

* TEMP - References D:\Local\Temp. Handy for writing out temp files you need to save for the life of a function. LOCALAPPDATA can also work for this. `"Temp File" > $env:TEMP\mytempfile.txt`
* FUNCTIONS_EXTENSION_VERSION - Refers to the version of Azure Functions Runtime being used. Not super useful now except as a test to see if you are running in Azure Functions if you need it.
* APPSETTING_* - This is a list of the function application settings. These are "secret" to the app itself, so you can edit these in the Functions Azure control panel app settings to add things like API keys and whatnot (though the better solution is to use Azure Key vault instead)

### Powershell Variables

Similarly, all the PS variables in the environment can seen with:

~~~powershell
Get-Variable | Format-Table -AutoSize | Out-String
~~~

Some notable Azure Function specific Powershell Variables:

* EXECUTION_CONTEXT_FUNCTIONDIRECTORY - The root of where your Azure Function code resides. Very handy when trying to reference or source other code.
* EXECUTION_CONTEXT_FUNCTIONNAME - The name of your function, e.g. "Sandbox". It is also helpful to test on this if it exists if you need to know if you are in Azure Functions or not (for instance if your code is designed to run multiple places) e.g.  `if ($EXECUTION_CONTEXT_FUNCTIONNAME) {"We are in Azure Functions!"}`
* REQ_* - Several variables including information about how the Azure Function was started if via HTTP request. [Mark Kean has written a great article on how these work in general](https://marckean.com/2017/10/18/powershell-based-azure-functions/).

## Powershell Modules

With Powershell 5.1, you get all the latest modules that came with 2016, including Pester and some newer version of the Azure Resource Manager modules. You can see the full list available:

``` powershell
Get-Module -ListAvailable | Format-Table -AutoSize | Out-String
```

### Powershell Gallery

Now that Azure Functions runs Powershell 5.1, you can use the native PackageManagement commands to run and import modules from the Gallery. Unfortunately, Powershell Gallery relies on NuGet, and it is not installed by default in the App Service VM. Fortunately, it's easy to fetch and install quickly:

~~~ powershell
#Silently Installs the NuGET requirement for Powershell Gallery if it isn't present.
if (-not (get-packageprovider Nuget)) {
    Install-PackageProvider Nuget -Scope currentuser | out-null
}
~~~

Once this is run, you can use `find-module`, `install-module`, `save-module` etc. just fine against the Powershell Gallery by default or you can add your own custom module repository. However, as of this writing `install-module` doesn't seem to work, however `Save-Module` works just fine if you point it to a folder already in the PSModulePath.

The following code will test if the POSHRSJob module is present, and if not, download it to your userprofile temp directory (which is stored on the TEMP storage and not persistent between App Service VMs). This keeps your function fast by not downloading it again if it already exists.

~~~ powershell
$ModuleToInstall = "POSHRSJob"
$PSLocalModulePath = "$($env:UserProfile)\Documents\WindowsPowershell\Modules"

<#
Normally we would use install-module -scope currentuser, but that doesn't work for some reason.
Save-Module accomplishes the same thing and can target multiple paths.
You can remove the Verbose part to cut down on the logs as well.
#>
if (-not (get-module $ModuleToInstall -listavailable)) {
    save-module $ModuleToInstall -Path $PSLocalModulePath -verbose 4>&1
}
~~~

At this point you can just invoke the Cmdlets in the module and Powershell will auto-import the module and its cmdlets on-demand, no need to import the module directly with `Import-Module`.

As noted above you can also create a "modules" directory in your function directory and save your files there if you want the modules to load every single invocation.

## Deployment

### Access Functions Directory via FTP

Since Azure Functions runs on App Service, [you can use FTP to access and deploy your functions](https://docs.microsoft.com/en-us/azure/app-service/app-service-deploy-ftp). The easiest way to find the credentials is to go to your function, go to Platform Featuers, then click Deployment Credentials and set your FTP password. Once complete you can go to Platform Featuers -> Properties and find your FTPS host name URL and FTP Deployment User to use to connect.

I find this connection is extremely useful in conjunction with the [ftp-simple Visual Studio Code extension](https://marketplace.visualstudio.com/items?itemName=humy2833.ftp-simple) to directly edit Azure Powershell Functions in VS Code rather than in the web editor. VS Code should also natively have this ftp remote edit functionality soon, [it already exists in preview mode](https://code.visualstudio.com/updates/v1_17#_preview-remote-file-system-api).

### Use the PowershellAzureFunctions module

In addition to the old fashioned FTP route, there is a powershell module you can use to make and upload Azure functions. If you want to get super-meta, you can deploy Azure Functions from an Azure Function this way (whoah, dude).

This MVP article covers how to use the Powershell Module as well as a brief "serverless computing" overview.

[Rapidly Deploy Serverless Azure Functions Using Powershell](https://blogs.msdn.microsoft.com/mvpawardprogram/2017/08/15/serverless-azure-functions/)

### Set up Continuous Deployment for Azure Functions

While direct editing is nice for sandboxing and practice, once you get serious and start designing solutions, you will want to save your Powershell Functions in some sort of source control (e.g. GitHub) and [publish them more automatically to Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-continuous-deployment). When used in combination with the [Azure Functions Slots Preview](https://blog.elmah.io/continuous-deployment-of-azure-functions-with-slots/) this is an extrempely powerful way to test and deploy your code in Production with minimal interruption to usage.

### Using Managed Service Identity with Functions

A common problem with Azure Functions is where to store credentials when accessing other services. Azure Key Vault is a logical location, and when combined with a Managed Services Identity, the Azure Function can securely access credentials it needs.

* [Enabling and Using Managed Service Identity to access an Azure Key Vault with Azure Powershell Functions](https://blog.darrenjrobinson.com/enabling-and-using-managed-service-identity-to-access-an-azure-key-vault-with-azure-powershell-functions/)
* [Using AD Managed Service Identity to Access Microsoft Graph with Azure Functions](https://gotoguy.blog/2017/09/21/using-azure-ad-managed-service-identity-to-access-microsoft-graph-with-azure-functions-and-powershell/)

## Troubleshoot and Debug with Kudu

Even though Azure Functions is supposed to take care of the "server" part for you, sometimes you need to get under the hood and fetch some files or test something in your App Service VM. Kudu is the console that gives you an interactive file browser, a process explorer, and powershell debug console to investigate problems. [David O'Brien has written a good article on how to access this troubleshooting environment](https://david-obrien.net/2016/07/azure-functions-kudu/).

## Powershell Azure Functions are Rad

I hope this helps you get your feet wet with Azure Powershell functions, they are an extremely powerful way to deploy solutions with Powershell that are highly scalable and extremely inexpensive.

