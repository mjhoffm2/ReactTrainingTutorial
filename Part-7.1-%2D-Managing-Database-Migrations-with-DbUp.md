# Managing Database Migrations with DbUp

## Overview

This page is not part of the main series on creating a basic slack clone, but if you have followed along so far, this may be a useful interlude after [Part 7 - Database Integration and Entity Framework Core](/Part-7-%252D-Database-Integration-and-Entity-Framework-Core).

In this tutorial, I will be going over an approach I use for managing database migrations in a project with many developers where the schema is constantly changing.  In the past, this has easily gotten out of hand.  In one project, we had developers creating scripts that other developers needed to manually run against their development databases to update them.  It was difficult to version control these migrations and keep track of which scripts needed to be run on different branches of the code.  It was also difficult to create and keep track of these scripts.  Frequently, it was necessary to just delete the developer's database and run the all the scripts from scratch.  Sometimes a developer would just create a giant script that just deleted everything and re-created all tables and other objects.

When it came to deploying updates to a non-local environment, things got even more challenging.  On a local environment, the developer is usually keeping their database up-to-date and is aware of what scripts they have run.  On a staging or production environment, though, this isn't always the case.  Often these environments are deployed to infrequently, so it is necessary to run many migrations at once.  Additionally, you usually can't just delete the database and restart the migrations from scratch.

To address these issues, we now use a strategy which involves having the application itself determine what migrations need to be applied to the database on startup.

## DbUp

### Overview

The tool we ended up using to solve our database migrations was DbUp: https://dbup.readthedocs.io/en/latest/

DbUp is available as a .NET package on NuGet: https://www.nuget.org/packages/dbup-core/ and https://www.nuget.org/packages/dbup-sqlserver/

This is a tool which can be configured to run when your application starts, and apply necessary migrations to the database.  It keeps track of what migrations have already been run by creating a `SchemaVersions` table in the database.  This table keeps track of the name of the scripts that have been run, as well as when they were run.  By default, it will run scripts in alphanumeric order, so we prefix each migration script with a `Script0001` in the name to enforce the order.

In addition to helping developers keep track of what scripts have already been run, it also makes database updates during deployments easy.  This also allows us to get a new developer up-to-date immediately.  Just as importantly, it also makes those scripts part of version control along with the rest of the code.

### Set up Migration Scripts

Before we configure DbUp, we need to decide where our migration scripts are going to be and some conventions for ordering them.  In this tutorial, I will create a folder in the project called `Migrations`, which will contain sql scripts following a consistent naming pattern to enforce the order alphanumerically.  I have a separate nested folder called `Manual` where I put scripts that I do not want to be run automatically, but that isn't necessary for this tutorial.

![image.png](/.attachments/image-5773bcfb-fcc4-4a32-99b0-78d3b6e51143.png)

In order for DbUp to pick up on the migration scripts we have added, we need to make sure that they are built as embedded resources.

![image.png](/.attachments/image-fc46bd60-ffd4-444f-b7bb-f64b14645158.png)

You should include these migration scripts in your version control system, as well as the .csproj configuration changes that will happen when you set them as an embedded resource.

### Implementation

Once we set up our migration scripts, or at least decided where they will go once we have them, we can configure DbUp.  We will run DbUp during the startup of our application.  This consists of two steps.

The first step is to build the upgradeEngine, which will decide if an upgrade is required and what migrations need to be applied.  The following is an example of how this can be done.

```cs
var upgradeEngine = DeployChanges.To
	.SqlDatabase("Server=.\SQLEXPRESS;Database=MyDatabase;Trusted_Connection=True;")
	.WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), (string s) => s.StartsWith("MyProject.Migrations.Script"))
	.Build();
```

The important parts are (1) the connection string, and (2) the filter to decide what embedded scripts to run.

Once the upgrade engine is built, we can use it to actually perform the upgrade.

```cs
if(upgradeEngine.IsUpgradeRequired())
{
	var result = upgadeEngine.PerformUpgrade();

	if(!result.Successful)
	{
		throw result.Error;
	}
}
```

Putting everything together, this is roughly what it should look like:

_Startup.cs_
```cs
using System;
using System.Reflection;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using Microsoft.AspNetCore.Hosting;
using DbUp;

namespace MyProject
{
	public class Startup
	{
		private IConfiguration config;
		private ILogger log;
		private IHostingEnvironment env;

		public Startup(IConfiguration config, IHostingEnvironment env, ILogger<Startup> log)
		{
			this.config = config;
			this.log = log;
			this.env = env;

			RunMigrations();
		}

		private void RunMigrations()
		{
			var upgradeEngine = DeployChanges.To
				.SqlDatabase(config.GetConnectionString("MyConnectionString"))
				.WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), (string s) => s.StartsWith("MyProject.Migrations.Script") && (env.IsDevelopment() || !s.Contains("DEVONLY")))
				.WithTransactionPerScript()
				.LogToConsole()
				.Build();

			if (upgradeEngine.IsUpgradeRequired())
			{
				log.LogInformation("Database Update required.");

				var result = upgradeEngine.PerformUpgrade();

				if (!result.Successful)
				{
					log.LogError(result.Error, "Failed to run migrations due to an error executing the sql scripts.");
					throw result.Error;
				}
				log.LogInformation("Completed database upgrade.  The following scripts were executed:");

				foreach (var script in result.Scripts)
				{
					log.LogInformation(script.Name);
				}
			}
			else
			{
				log.LogInformation("No database update is required.");
			}
		}
	}
}
```

I pull the connection string from the app configuration instead of putting it directly in the code.  I also have logic which only allows migration scripts with "DEVONLY" in the name to be run on local development environments.  This makes it so we can have automatic test data created as part of the migrations in development environments, without affecting staging and production environments.

## Try it out

After you start the application server, you should be able to see the scripts that have been run by checking the `SchemaVerions` table.

![image.png](/.attachments/image-b8605fb0-6876-47e3-bf17-6b5c935399a3.png)

If you have a migration script that failed to run, this will be treated as a startup error and will prevent the application from starting.

### Best practices

#### Migrations from an empty database

DbUp can create the `SchemaVersions` table automatically, but you shouldn't use it to create the database itself for you.  Before running the application for the first time, you should manually create the initial empty database.  You should then set up your migration scripts to be able to go from the completely empty database to fully up-to-date without any manual intervention.

#### Writing migration scripts

1. When writing a migration script, you should always write it to target the database as it would be at the end of the previous migration script.
2. Avoid including `Using [MyDatabase]` statements, since it creates an unnecessary dependency on the name of the database.  This should only be necessary if you have multiple databases.
3. Keep changes as minimal and non-invasive as possible.  Avoid dropping and re-creating tables.  Not only do we want to preserve tables, but we also want to avoid overwriting changes that another developer might be creating a migration script for at the same time.
4. Always keep data in mind when making changes.  If you change the data type of a column, you will need to make sure that you properly migrate the data in that column.  When adding constraints, you need to make sure that the data meets those constraints.  Just because you don't have any test data locally doesn't mean that other people won't.
5. Coordinate order with other developers.  In my experience, this is as simple as calling 'dibs' on the next script number in a slack channel.  If you are working on a stored procedure or something else that needs to be completely replaced, mention that when you are calling dibs.  This way, other developers know not to mess with that stored procedure until the script with that number has been committed and run.

#### Updating migration scripts

It is important to keep in mind that once a script has run, it will not be re-run on the same database even if it changes.  As a result, if a mistake is made in a migration script, it should be addressed by creating a new migration script that fixes the problem, deleting the old script if necessary.  If the script is simply modified without changing the name, it will not be re-run on any database that already had the script applied.

#### Transitioning from manual to DbUp

You may have situations where you want to introduce DbUp part way through a project.  In those scenarios, while you still want to be able to support creating all the schema from an empty database, it is also useful for developers with existing databases to be able to preserve their current setup without resetting.

However, the developers will already have all the initial schema set up on their database without a `SchemaVersions` table recording it.  When DbUp tries to run, it will try to run the migration scripts from the beginning, which will likely fail since it will try to create tables that already exist.  In order to avoid this issue, it is necessary for the developer to either completely reset their database and let DbUp create the schema from scratch, or add the relevant script names into the `SchemaVersions` table to skip them.  If a developer already has a table created, they can add the relevant script to their `SchemaVersions` table so DbUp won't try to run it.

The solution that I came up with is to create a special `Script0000` which will throw a friendly error if the database is not empty.  This way, the startup will fail immediately with a helpful error message instead of trying to run scripts in a non-empty database, potentially messing it up.

_Migrations\/Script0000 - prevent incorrect execution.sql_
```sql
if OBJECT_ID(N'MyTable') IS NOT NULL
BEGIN
	throw 51000, 'Baseline migrations are only valid in a fresh database, but it was detected that other migrations have already run.  You must either (1) reset the database or (2) manually insert script names into the SchemaVersions table to skip them.', 1
END
```

![image.png](/.attachments/image-64f850bd-3f5a-4270-9c59-df2e244a7dbd.png)

This way, a developer can't accidentally run migrations on an old database without first deciding what migration scripts need to run in the `SchemaVersions` table.

#### Stored procedures and functions

Stored procedures cannot be incrementally updated.  They need to be completely replaced with an `ALTER PROCEDURE` command.  As a result, migration scripts to update a stored procedure are essentially going to be mass copy-paste versions of the previous migration script with some changes.  This can make it inconvenient to compare the stored procedure to the last version to see exactly what changed, and creates a lot of redundancy.

To address this issue, the strategy that I use is to simply update the previous migration script for that stored procedure with the changes that I want, and then also update the name of that script.  This will cause the script to be run again, and makes it easy to see the diff in git.  Additionally, there will only be one copy of the stored procedure in the migration scripts folder at a time, instead of a bunch of old and redundant versions.  The only caveat is that it is necessary to start the script with `CREATE OR ALTER PROCEDURE` to accommodate both updating the procedure, as well as creating it in a fresh database.