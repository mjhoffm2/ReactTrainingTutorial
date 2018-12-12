# Part 7 - Database Integration and Entity Framework Core

## Overview

This part of the tutorial is continued from [Part 6 - Setting up UI with .NET Core instead of Node](/Part-6-%252D-Setting-up-UI-with-.NET-Core-instead-of-Node.md).  However, it is not necessary to have followed the tutorial up to this point.  The only prerequisite for this part of the tutorial will be to have some sort of .NET Core application already set up.

In this part of the tutorial will be going over setting up a sql database for our Slack Training App, and using that to create an Entity Framework Database Context.  We will also be going over the configuration for how to connect to our database, as well as creating some example service calls using Entity Framework Core.  The process for starting with a sql database, and creating a C# database context from it is referred to as the "database first" approach.  The alternative is referred to as "code first", which is a valid approach but I will not be going over it in this tutorial.

## Set up SQL Database

### Overview

For this part of the tutorial, we will set up a very simple database with just 4 tables.  Here is an ER diagram for these tables:

![image.png](/.attachments/image-521990cf-e13f-46e2-820f-8f75f57dcc42.png)

For more information about the model we will be using, please see [Slack Training App - Tutorial Overview](/Slack-Training-App-%252D-Tutorial-Overview.md).

### Getting Required Tools

The first thing you will need is a database server.  For this tutorial, I will be using SQL Server Express 2017, which you can [download from Microsoft](https://www.microsoft.com/en-us/sql-server/sql-server-editions-express).  This should Just Work™ without much additional configuration.  This will run in the background of your computer as a service.

You will also need something like [SQL Server Management Studio](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-2017) to interact with your database.  This is the software you will use to manage your SQL Server Express instance, run queries, and update your schema.  If you require additional help getting a database server set up, you will need to do some googling.

### Create Schema

To save you some time, I will provide a zip containing two sql scripts to get started: [SlackTraining init sql.zip](/.attachments/SlackTraining%20init%20sql-c82d9fcf-c464-4f68-b037-a2ed32b70f56.zip)

The 'init db' script will create an empty database on the database server called `SlackTraining`.  You could easily make this database yourself if you want.  The other 'init ddl' script will create the four tables you see in the diagram above, as well as the indices and foreign key constraints between them.  The `SlackTraining` database must exist before you run the 'init ddl' script.  You should be able to simply open up SSMS, connect to the local SQL Server Express instance, and run these queries without any additional configuration.

If you want to change the database name, or if the init db script doesn't work for you, you can simply skip it and create the database yourself with any name.  Next, update or remove the `USE [SlackTraining]` at the beginning of the init ddl script, and run it.  Make sure that the database name that you pick is reflected in your connection strings.

## Create Entity Framework Database Context

### Overview

For this tutorial, I will be using the "database first" approach of first creating the database, and then creating a database context from it.  In this case, I will actually be using a tool which will automatically generate our database context and c# models.  This will be roughly equivalent to the entire DAO layer, if you are familiar with that concept.  This database context and c# models will serve as the interface between our .NET Core application and the SQL database.

### Entity Framework Scaffolding Tool

To generate the Entity Framework database context, we will be using the Entity Framework Scaffolding tool.  This will analyze our database and generate a database context and C# models based on the tables in our SQL database.  It is smart enough to pick up on foreign key constraints, unique keys, data types, and much more to generate an intelligent structure for your database context.

Before we can use it, we need to figure out what connection string we want to use, and where we want the files to be placed.

For this tutorial, I decided on the following file structure:

![image.png](/.attachments/image-fe8ad13e-e4fa-4e56-852a-f75149473cdf.png)

For now, we can just use windows authentication for the connection string.  This is an authentication scheme that simply uses the fact you are signed in to your local windows account.  To connect to the SlackTraining database on our local SQL Express server using windows authentication, we use the following connection string:

`"Server=.\SQLEXPRESS;Database=SlackTraining;Trusted_Connection=True;"`

If you are using a different server, database, or authentication scheme, you will need to figure out the connection string for that.

With our connection string and our output location, we can now use the scaffolding tool.  In Visual Studio, you can open up the Package Manage Console, and run the following command:

```
Scaffold-DbContext "Server=.\SQLEXPRESS;Database=SlackTraining;Trusted_Connection=True;" Microsoft.EntityFrameworkCore.SqlServer -o Models/Database -Context SlackTrainingDbEntities -f -d -v
```

This will build our project, then generate a bunch of files in the `Models/Database` directory.  These files will include a `SlackTrainingDbEntities.cs` file, which will be our Entity Framework Core Database Context.  Please note that within this file, the connection string will be included in the `OnConfiguring` method.  If you put any actual credentials into your connection string when using the scaffolding tool, you should remove them from your code before committing them to source control.  Also note that if your project is currently in a state that does not successfully build, the scaffolding tool will not run.

The scaffolding tool may also add a section to your .csproj file that looks like `<Folder Include="Models\" />` or `<Folder Include="Models\Database" />` under an ItemGroup.  This is unnecessary and you can remove it.

### Extending the Generated Classes

#### Code changes

After using the entity framework scaffolding tool, we will have a model created for each of our database tables, and a model created for our entity framework context, extending the `DbContext` class from EF Core.  However, I would strongly advise against making any changes to these files, since those changes will be overwritten if you run the scaffolding tool again.  As you add more tables and update existing tables, it will save a lot of time to be able to automate the process by re-running the same scaffolding command.  Therefore, any adjustments to the generated code should ideally be maintained in separate files.

The EF scaffolding tool generates partial classes, which makes it possible to extend these existing classes quite easily.  

#### The appsettings file

The first thing we will likely need to adjust is our connection string.  Instead of placing the connection string directly in the code, we will place it in the appsettings file for the particular environment.  When debugging in your local environment, this environment is 'Development'.  Edit or create a file called `appsettings.Development.json` located in the same directory as your .csproj file, and add a field called `"ConnectionStrings"`.  Within this field, we will add any connection strings which are necessary for our application.

_appsettings.Development.json_
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information"
    }
  },
  "ConnectionStrings": {
    "SlackTrainingConnectionString": "Server=.\\SQLEXPRESS;Database=SlackTraining;Trusted_Connection=True;"
  }
}
```
In this example, I am adding the same connection string as I used to run the scaffolding tool, but you can have any valid connection string.  Make sure not to commit sensitive credentials to version control.  You can have different connection strings for different environments, and ASP.NET Core will automatically use the appsettings file that has a name matching `appsettings.{ENVIRONMENT}.json`.

#### Creating a new Database Context

While the automatically generated database context likely contains 99% of what we want, it is still important to be able to make careful adjustments to the model and add behavior depending on the current configuration.  To accomplish this, we will create a new database context that extends the generated one.  I will place this new context in a new top-level folder called `Services`

_Services\/SlackTrainingDb.cs_
```cs
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using React_Demo.Models.Database;

namespace React_Demo.Services
{
	public class SlackTrainingDb : SlackTrainingDbEntities
	{
		public readonly IConfiguration config;

		//config will be provided via dependency injection
		public SlackTrainingDb(IConfiguration config)
		{
			this.config = config;
		}

		protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
		{
			//use the connection string provided in the appsettings.{Environment}.json file
			optionsBuilder.UseSqlServer(config.GetConnectionString("SlackTrainingConnectionString"));
		}
	}
}
```

This simple service simply extends the existing SlackTrainingDbEntities class and provides a new OnConfiguring method which pulls the connection string from our configuration.

We will also need to remember to register this service for dependency injection in the `ConfigureServices` method in the `Startup` class.

_Startup.cs_
```cs
public void ConfigureServices(IServiceCollection services)
{
	services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);
	services.AddDbContext<SlackTrainingDb>();
}
```

Now, the `SlackTrainingDb` service is ready to be used.

## Use the EF Core to make a CRUD API

### Overview

Now that we have got our `SlackTrainingDb` database context working and configured as a service, it can be used by our other services.  The easiest way to do this is to use dependency injection in the constructors for the service.  You should not be manually instantiating your services with `new` in ASP.NET Core.

In this section, we are going to create a very basic api to manage channels.  This api will have an endpoint for getting the list of channels, and another endpoint for adding a channel.  Later on, we can extend this api to do things like update channels, delete channels, perform permission checks, validation, and so on.

Our basic CRUD api will need a service and a controller, as well as a tiny bit of other configuration.

### Service Implementation

Using the database context is super simple, and this simplicity is the whole reason we went through all the previous effort.  To provide our channel manager service with the SlackTrainingDb service we made, we simply add it as a parameter to the constructor and let dependency injection handle the rest.

If we want to get the list of channels from the database, we can use `context.Channel`.  This will give us a `DbSet<Channel>`, which extends `IEnumerable<Channel>`.  For performance reasons (and avoid issues with the database getting disposed), we will convert this DbSet to a List asynchronously using `await context.Channel.ToListAsync()` before returning.  It is important to keep in mind that the data doesn't actually get read from the database until the results are actually read and enumerated, which in this case is done by converting it to a list.

When we want to add a channel to the database, we do it in two steps.  First, we add the information about the channel we want to create to the database context.  This will allow EF Core to start tracking the entity.  Next, we call `await context.SaveChangesAsync();` to write it to the database.

Below is an example implementation for the channel manager service using the SlackTrainingDb service we put together.

_Services\/ChannelManagerService.cs_
```cs
using Microsoft.EntityFrameworkCore;
using React_Demo.Models.Database;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace React_Demo.Services
{
	public class ChannelManagerService
	{
		readonly SlackTrainingDb context;

		public ChannelManagerService(SlackTrainingDb context)
		{
			this.context = context;
		}

		public async Task<List<Channel>> GetChannels()
		{
			return await context.Channel.ToListAsync();
		}

		public async Task AddChannel(string displayName, string description, bool isPublic)
		{
			if(String.IsNullOrWhiteSpace(displayName))
			{
				throw new ArgumentException("Channel display name cannot be empty", nameof(displayName));
			}

			context.Channel.Add(new Channel()
			{
				DisplayName = displayName,
				Description = description,
				IsPublic = isPublic
			});

			await context.SaveChangesAsync();
		}
	}
}
```

Don't forget to also register the new service for dependency injection by adding it to the `ConfigureServices` method in the `Startup` class.  Otherwise, you won't be able to use this service via dependency injection in other classes.

_Startup.cs_
```cs
		// This method gets called by the runtime. Use this method to add services to the container.
		public void ConfigureServices(IServiceCollection services)
		{
			services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);

			services.AddDbContext<SlackTrainingDb>();
			services.AddScoped<ChannelManagerService>();
		}
```

### Controller Implementation

In order for our service methods to be accessible through our api, we will need to make some endpoints for it.  To create some endpoints, we will create a `ChannelManagerController` class which extends the `Controller` class.  Classes that extend `Controller` are treated specially by the ASP.NET Core runtime.  In our controller class, we can make the channel manager service available using dependency injection, just like how we made the database context available to the channel manager service.

_Controllers\/ChannelManagerController.cs_
```cs
using Microsoft.AspNetCore.Mvc;
using React_Demo.Models.Database;
using React_Demo.Services;
using System.Threading.Tasks;

namespace React_Demo.Controllers
{
	[Route("api/channels")]
	public class ChannelManagerController : Controller
	{
		readonly ChannelManagerService channelService;

		public ChannelManagerController(ChannelManagerService channelService)
		{
			this.channelService = channelService;
		}

		[HttpGet]
		public async Task<IActionResult> GetChannels()
		{
			var channels = await channelService.GetChannels();
			return Json(channels);
		}

		[HttpPost]
		public async Task<IActionResult> AddChannel([FromBody] Channel channel)
		{
			await channelService.AddChannel(channel.DisplayName, channel.Description, channel.IsPublic);
			return Ok();
		}
	}
}
```

In the above example controller class below, we create two endpoints.  The first endpoint, `GET /api/channels`, doesn't take any arguments and returns the list of channels from the channel manager service we made.  The second endpoint, `POST /api/channels`, requires a JSON body which matches the fields on a Channel.  Only the `displayName` field is required.  We use annotations such as `[FromBody]` and `[HttpGet]` to configure how our controller methods are accessed and how requests should be interpreted.

Controllers do not need to by registered in the `ConfigureServices` method in the `Startup` class, so we are done.

## Test it out

At this point, we can go ahead and run our server to test our our api endpoints.  You can test out these endpoints using a tool like postman, or you can just run some javascript in the browser window that appears when you run the debugger.  The below snippet of javascript code will create a channel called "the cool kids club".

```js
var data = JSON.stringify({
  "displayName": "the cool kids club"
});

var xhr = new XMLHttpRequest();
xhr.withCredentials = true;

xhr.addEventListener("readystatechange", function () {
  if (this.readyState === 4) {
    if (this.status >= 200 && this.status < 300) {
      console.log(this.status + " " + this.statusText);
    } else {
      console.error(this.status + " " + this.statusText);
    }
  }
});

xhr.open("POST", "/api/channels");
xhr.setRequestHeader("Content-Type", "application/json");
xhr.setRequestHeader("Cache-Control", "no-cache");

xhr.send(data);
```
Running this in your browser should produce a `200 OK` response.

After the channel has been created, you can test out the endpoint for getting the list of channels to see it

GET http://localhost:53609/api/channels
```json
[
    {
        "channelId": 1,
        "ownerId": null,
        "displayName": "the cool kids club",
        "description": null,
        "isPublic": false,
        "canAnyoneInvite": false,
        "isGeneral": false,
        "isActiveDirectMessage": false,
        "isSoftDeleted": false,
        "owner": null,
        "message": [],
        "userChannel": []
    }
]
```

## Issues I ran into and how to address them

### Generated code contains errors due to project name

One issue I ran into with the scaffolding tool is the fact that the name of my project was originally 'React Demo', but an indentifier cannot have a space in it so my namespace is actually 'React_Demo'.  The scaffolding tool, unfortunately, didn't pick up on this and generated files with `namespace React Demo.Models.Database` in every file, which is a compilation error.  Unfortunately, the only workarounds that I could come up with were to either not have placed a space in my project name to begin with, or to edit all my generated models after I use the scaffolding tool.  Hopefully, this will be addressed soon.  Alternately, I could set up a separate project in my solution for my generated database models and context.  In this tutorial, I simply fixed the names after using the scaffolding tool.  Later, I renamed the project to be 'React_Demo' with an underscore instead of a space.  This was done by renaming and updating the solution (.sln) file, then renaming the .csproj file.

### When I try to create a channel, I get a 415 response code

A 415 response code indicates that your request body could not be bound to the `channel` parameter in the controller endpoint.  This is likely due to one of two reasons.  Either your json body is missing or malformed, or you forgot to include the `Content-Type: application/json` header.

## Download Source

The source code up to this point can be found here:

https://github.com/mjhoffm2/react-demo/tree/Part-7

The code for this part of the tutorial can be found in the `.net core 2.1` folder.