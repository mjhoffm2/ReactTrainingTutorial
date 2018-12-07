# Part 7 - Database Integration and Entity Framework Core

## Overview

This part of the tutorial is continued from [Part 6 - Setting up UI with .NET Core instead of Node](/Part-6-%252D-Setting-up-UI-with-.NET-Core-instead-of-Node.md).  However, it is not necessary to have followed the tutorial up to this point.  The only prerequisite for this part of the tutorial will be to have some sort of .NET Core application already set up.

In this part of the tutorial will be going over setting up a sql database for our Slack Training App, using that to create an Entity Framework Database Context.  We will also be going over the configuration for how to connect to our database, as well as creating some example service calls using Entity Framework Core.  The process for starting with a sql database, and creating a C# database context from it is referred to as the "database first" approach.  The alternative is referred to as "code first", which is a valid approach but I will not be going over it in this tutorial.

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

One issue I ran into with the scaffolding tool is the fact that the name of my project is 'React Demo', but an indentifier cannot have a space in it so my namespace is actually 'react_demo'.  The scaffolding tool, unfortunately, didn't pick up on this and generated files with `namespace React Demo.Models.Database` in every file, which is a compilation error.  Unfortunately, the only workarounds that I could come up with were to either not have placed a space in my project name to begin with, or to edit all my generated models after I use the scaffolding tool.  Hopefully, this will be addressed soon.  Alternately, I could set up a separate project in my solution for my generated database models and context.  In this tutorial, I will simply be manually fixing the names after using the scaffolding tool.

### Extending the Generated Classes

After using the entity framework scaffolding tool, we will have a model created for each of our database tables, and a model created for our entity framework context, extending the `DbContext` class from EF Core.  However, I would strongly advise against making any changes to these files, since those changes will be overwritten if you run the scaffolding tool again.  As you add more tables and update existing tables, it will save a lot of time to be able to automate the process by re-running the same scaffolding command.  Therefore, any adjustments to the generated code should ideally be maintained in separate files.

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
In this example, I am adding the same connection string as I used to run the scaffolding tool, but you can have any valid connection string.  Make sure not to commit sensitive credentials to version control.

The EF scaffolding tool generates partial classes, which makes it possible to extend our existing classes quite easily.