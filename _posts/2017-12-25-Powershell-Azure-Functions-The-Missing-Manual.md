---
title: Powershell Azure Functions The Missing Manual
---

# Powershell Azure Functions: The Missing Manual

Recently the Azure Functions runtime was updated to run on Windows Server 2016. This means that Powershell now runs 5.1 instead of 4.0 and there is a lot of benefit that comes from this that may not be documented yet, hence I am documenting it here.

This article is targeted towards people who have Powershell experience, and need to know how the behavior is different from a normal Powershell 5.1 environment.

## Why Azure Functions over Azure Automation (or a Windows Server for that matter)?

Azure Automation is a much more mature Powershell environment in Azure. Azure Functions is still considered experimental. So why would you want to use it?

1. **It's Dirt Cheap**: Azure Functions gives you 400,000 GB-s and 1 million executions for free every month under the Consumption plan. Azure Automation only gives you 500 minutes. You can build an incredibly advanced Powershell function workflow and API environment and never get billed a single dime.
2. **It's Fast**: With Azure Automation you usually have to wait for the job to be scheduled, queued, etc. and the feedback loop is very slow. Azure Functions start in seconds and scale nearly limitlessly and automatically, making it far easier to test and troubleshoot.
3. **It's better for APIs**: Azure Functions makes it really easy to make a query with a URL or trigger on an object, take that input into Powershell, do Powershell-y things to it, and output it back out as JSON, XML, or whatever you want. You can build great APIs with Powershell very rapidly, why should the C# and Java guys have all the fun?

## Azure Functions Environment

Azure Functions operates basically by running on top of [Azure Web Jobs](https://buildazure.com/2017/03/08/azure-functions-vs-web-jobs-how-to-choose/) which in turn run on [Azure App Service](https://azure.microsoft.com/en-us/services/app-service/). As a result, [the same OS enviroment restrictions apply](https://docs.microsoft.com/en-us/azure/app-service/web-sites-available-operating-system-functionality).

The biggest limitation is that under the free Consumption (Dynamic) plan, jobs will timeout if they take longer than 5 minutes to execute by default. [You can increase that up to 10 minutes by editing the host JSON](https://buildazure.com/2017/08/17/azure-functions-extend-execution-timeout-past-5-minutes/). However, Azure Functions are generally meant to be designed to be "small and fast" so if your process takes longer than 10 minutes, consider breaking it into chunks and connecting them via something like Azure Queues, or run a control script on Azure Automation, a function on a dedicated App Service plan, or a Windows VM.

### Logs Panel and Verbose/Debug Output

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
2. **Modules Directory:** Inside of your function directory, you can create a directory called "modules" which Azure Functions will autoload any powershell modules that are part of your script. While this is an easy option, I generally recommend not using this, and instead loading modules as you need them using Powershell 5.1's built-in loading logic to optimize performance. For instance, if your full function package has a library of 50 modules, maybe only 1 or 2 need to be run for any individual invocation, so why 

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
* APPSETTING_* - 

### Powershell Variables

Similarly, all the PS variables in the environment can seen with:

~~~powershell
Get-Variable | Format-Table -AutoSize | Out-String
~~~

Some notable Azure Function specific Powershell Variables:

* EXECUTION_CONTEXT_FUNCTIONDIRECTORY - The root of where your Azure Function code resides. Very handy when trying to reference or source other code.
* EXECUTION_CONTEXT_FUNCTIONNAME - The name of your function, e.g. "Sandbox". It is also helpful to test on this if it exists if you need to know if you are in Azure Functions or not (for instance if your code is designed to run multiple places) e.g.  `if ($EXECUTION_CONTEXT_FUNCTIONNAME) {"We are in Azure Functions!"}`
* REQ_* - Several variables including information about how the Azure Function was started if via HTTP request. [Mark Kean has written a great article on how these work in general](https://marckean.com/2017/10/18/powershell-based-azure-functions/).