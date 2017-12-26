---
title: Powershell Azure Functions The Missing Manual
---

# Powershell Azure Functions: The Missing Manual

Recently the Azure Functions runtime was updated to run on Windows Server 2016. This means that Powershell now runs 5.1 instead of 4.0 and there is a lot of benefit that comes from this that may not be documented yet, hence I am documenting it here.

## Azure Functions Environment

Azure Functions operates basically by running jobs within [Azure App Service](https://azure.microsoft.com/en-us/services/app-service/). As a result, [the same OS enviroment restrictions apply](https://docs.microsoft.com/en-us/azure/app-service/web-sites-available-operating-system-functionality).

### Logs Panel and Verbose/Debug Output

Azure Functions have a built in log streaming service panel. For powershell, unlike the normal powershell console, this log streaming will only show the Output and Error streams and will not show the Debug, Verbose, or Host streams at all, so in other words Write-Host, Write-Debug, and Write-Verbose will show nothing in this log. The DebugPreference and VerbosePreference variables also have no effect on this.

In order to see the verbose output of a command, you need to use the stream redirection commands. After you understand how [Streams and Redirection work in Powershell](https://blogs.technet.microsoft.com/heyscriptingguy/2014/03/30/understanding-streams-redirection-and-write-host-in-powershell/), you can see the verbose output of a command with:

~~~Powershell
Write-Verbose "This is Verbose Text" 4>&1
~~~

Where 4>&1 means take the Verbose stream (numbered 4), and redirect it to the output stream which shows in the log. You can do this for warning (3), Error (2), and Debug (5) as well.

### Home and Local Drive Directories

In Azure Functions you have two primary local storage directories available to you: **D:\Home** and **D:\Local**.

**D:\Home** is your persistent area where files are kept between multiple runs. It is backed by an Azure storage account. The code for your azure function is stored in this location, generally in **D:\home\site\wwwroot\<functionname>**

**D:\Local** is where temporary files are stored and are not persistent between multiple function instances. You can use this area to store files that are specific to a run, for instance temporary data you've downloaded from another source or data you want to dump out. Keep in mind that these files may or may not be present depending on if your next function run started on the same instance or not (which depends on the type of App Service plan you chose), so ensure if you write scripts that write to the temp that you overwrite anything in place, and optionally clean up any temp files you use, so that you don't get junk from one run affecting another run.

### Special Azure Functions Directories

1. **Function Directory:** This is where your function files live as you see them in the "View Files" tab. The path is stored in the **$EXECUTION_CONTEXT_FUNCTIONDIRECTORY** powershell variable. This location is **D:\home\site\wwwroot\\\<functionname\>**, where \<functionname\> is the name of your Azure function. For instance, if you name your function Sandbox, the path is *D:\home\site\wwwroot\Sandbox*. I recommend running `Set-Location $EXECUTION_CONTEXT_FUNCTIONDIRECTORY` as one of your first function activities so your relative commands can call other resources in your function directory for ease of use.
2. **Modules Directory:** Inside of your function directory, you can create a directory called "modules" which will autoload any powershell modules that are part of your script. While this is an easy option, I generally recommend not using this, and instead loading modules as you need them using Powershell 5.1's built-in loading logic.

## Variables

### Environment Variables

All the environment variables in Azure Functions that you can use can be seen with:

~~~powershell
Get-Item env: | format-table -autosize | out-string
~~~

The format table and out-string are necessary because Functions by default will output the object as its object type, this gives it a more "powershell-y" output to the Function log.

### Powershell Variables

Similarly, all the variables in the environment can seen with:

~~~powershell
Get-Variable
~~~

## Inputs and Outputs

There are several ways to provide input to a powershell function and handle the resulting output. [Mark Kean has written a great article on how these work in general](https://marckean.com/2017/10/18/powershell-based-azure-functions/).