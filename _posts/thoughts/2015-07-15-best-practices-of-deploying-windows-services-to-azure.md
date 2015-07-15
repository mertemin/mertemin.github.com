---
layout: post
category : thoughts
tagline: "best practices of deploying windows services to azure"
tags : [thoughts]
---
{% include JB/setup %}

Azure provides a set of services and components that you can use to develop distributable and scalable applications. Even though the whole platform and its pieces (names, versions, definitions etc.) change every so often, the value you can get from it is still high. However, the lack of documentation creates an ambigiuty about how things work or what is possible and not. Often I found myself digging through `n`th search result or years old blog to figure things out. That's why I want to share what I have learned about deploying Windows services to Azure.

### Background
Recently I had a task to create custom out-of-process diagnostics and monitoring agent for cloud roles. I thought that was going to be an easy task, since I could just create another role and implement the agent within the role. Using Azure role definition entry points for start, stop and update, it was going to be pretty clear and simple implementation.

I was pushing log entries to ETW (Event Tracing for Windows) and generating performance data (using `System.Diagnostics` performance counter definitions) in the actual role. The agent role was able to listen ETW for logs and read performance data periodically. It didn't take long till I figure out roles are like virtual machine templates, which means each role goes to a separate virtual machine, by deploying my locally perfectly working proof of concept.

After some research, I learned about [startup task](https://msdn.microsoft.com/en-us/library/azure/hh180155.aspx)s. I converted my agent role to a Windows service and created a startup task to start up the agent service when role is being started on the cloud. In the following, there is a quick summary on how to deploy Windows services along with a role.

#### Create the Service
Create a new Windows Service project. The project will come with the following files:
- App.config: Application specific configurations live here. In the case of Windows service living in a cloud role, this file is invaluable to resolve common assembly conflict that could occur between the role and service.
- Program.cs: This is a sample program for the service to run.
- Service1.cs: This is the sample service that will be called by the program. You can remove the UI-specific methods and properties if you want. There are three methods for you to implement: `OnStart(string[] args)`, `OnStop()` and `Dispose(bool disposing)`.

#### Add a Service Installer
Add a class to install service to the machine. First, reference `System.Configuration.Install` library and then add a class, let's call it `AgentInstaller`, which should look like the following code piece. Select your service's `Account` type (whether it is going to be `Service` account or `System` account) and `StartType` (manual or automatic). With manual start option you can pass arguments to your service from your cloud role and make the service general to work with given parameters. In my case, I was passing connection strings, table names, app tokens to the agent service.

	using System.ComponentModel;
	using System.Configuration.Install;
	using System.ServiceProcess;

	namespace WindowsService1
	{
	    /// <summary>
	    /// This the installer for the monitoring service.
	    /// The monitoring service is installed to the target machine as a long-running local service.
	    /// </summary>
	    [RunInstaller(true)]
	    public class AgentInstaller : Installer
	    {
	        /// <summary>
	        /// Process installer
	        /// </summary>
	        public ServiceProcessInstaller ServiceProcessInstaller { get; set; }

	        /// <summary>
	        /// Default constructor
	        /// </summary>
	        public AgentInstaller()
	        {
	            ServiceProcessInstaller = new ServiceProcessInstaller { Account = ServiceAccount.LocalService };

	            var serviceInstaller =
	                new ServiceInstaller
	                {
	                    ServiceName = "Your service name",
	                    Description = "Your service description",
	                    DisplayName = "Your service's display name",
	                    StartType = ServiceStartMode.Manual
	                };

	            Installers.AddRange(new Installer[] { ServiceProcessInstaller, serviceInstaller });
	        }
	    }
	}

#### Reference Your Service in Role Project
Find your Web or Worker role project (an MVC project, a class library etc) and reference Windows service project. Make sure to enable `copy local` option to bring dependent assemblies. If your Windows Service project depends on an external library, be sure to enable copy local there too. This is where you might have issues. You service executable will be placed to the app folder of your role. Therefore, it will be expecting libraries (dlls) to be placed there for successful execution.

#### Copy Installer Tool and Create an Install Script
You need the installer tool and install script to install the service. The tool is not available in the virtual machine by default. You need to copy the executable yourself. Find it on your machine or download from the Web. Then, place the executable to the folder of your role in your cloud project's (not your role project). [Here](http://blogs.msdn.com/b/philliphoff/archive/2012/06/08/add-files-to-your-windows-azure-package-using-role-content-folders.aspx) is a post on that. You can do this by dragging and dropping the file onto role name in the project. For web roles, create a folder called `bin/` first, since Web executables are placed in a folder called bin in application root folder. This copy will make sure the executables are placed in your application's root folder.

Now create an install script, let's assume it is called `InstallAgent.cmd`, in the same folder described above. The sample script given below first stops and deletes if service is running, and then reinstalls the one in the newly deployed package. `"%TEMP%"` is an environment variable pointing to a temporary folder to create files. Therefore, you can use to write logs that might be useful later to diagnose problems.

	:: Stop and delete the current service
	sc stop "Your service's name" >> "%TEMP%\StartupLog.txt" 2>&1
	sc delete "Your service's name" >> "%TEMP%\StartupLog.txt" 2>&1

	:: Uninstall and install the service
	InstallUtil.exe /u "Your executable name" >> "%TEMP%\StartupLog.txt" 2>&1
	InstallUtil.exe "Your executable name" >> "%TEMP%\StartupLog.txt" 2>&1

	:: Exit
	EXIT /B 0

#### Create a Startup Task
Create a startup task in your service definition file. Select the execution context and task type. This will initiate a task with given script file when a new package is deployed.

	<Startup>
		<Task commandLine="InstallAgent.cmd" executionContext="elevated" taskType="background" />
	</Startup>

#### Start Service in Your Role
Start your service in your role's `OnStart` entry point using `ServiceController` helper classes.

	var agentController = new ServiceController("Your service name");
	try
	{
	    if (agentController.Status != ServiceControllerStatus.Running)
	    {
	        agentController.Start(
	            new[] { "argument 1", "argument 2", "argument 3" });
	        agentController.WaitForStatus(ServiceControllerStatus.Running);
	    }
	}
	catch (Exception exception)
	{
	    throw;
	}
