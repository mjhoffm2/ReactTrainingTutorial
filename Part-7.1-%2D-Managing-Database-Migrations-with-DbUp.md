# Managing Database Migrations with DbUp

## Overview

This page is not part of the main series, but if you have followed along so far, this may be a useful interlude.

In this tutorial, I will be going over an approach I use for managing database migrations in a project with many developers where the schema is constantly changing.  In the past, this has easily gotten out of hand.  In one project, we had developers creating scripts that other developers needed to manually run against their development databases to update them.  It was difficult to version control these migrations and keep track of which scripts needed to be run on different branches of the code.  It was also difficult to create and keep track of these scripts.  Frequently, it was necessary to just delete the developer's database and run the all the scripts from scratch.  Sometimes a developer would just create a giant script that just deleted everything and re-created all tables and other objects.

When it came to deploying updates to a non-local environment, things got even more challenging.  On a local environment, the developer is usually keeping their database up-to-date and is aware of what scripts they have run.  On a staging or production environment, though, this isn't always the case.  Often these environments are deployed to infrequently, so it is necessary to run many migrations at once.  Additionally, you usually can't just delete the database and restart the migrations from scratch.

To address these issues, we now use a strategy which involves having the application itself determine what migrations need to be applied to the database on startup.

## DbUp

### Overview

The tool we ended up using to solve our database migrations was DbUp: https://dbup.readthedocs.io/en/latest/

This is a tool which can be configured to run when your application starts, and apply necessary migrations to the database.  It keeps track of what migrations have already been run by creating a `SchemaVersions` table in the database.  This table keeps track of the name of the scripts that have been run, as well as when they were run.  By default, it will run scripts in alphanumeric order, so we prefix each migration script with a `Script0001` in the name to enforce the order.

In addition to helping developers keep track of what scripts have already been run, it also makes database updates during deployments easy.  This also allows us to get a new developer up-to-date immediately.  Just as importantly, it also makes those scripts part of version control along with the rest of the code.

### Implementation

![image.png](/.attachments/image-5773bcfb-fcc4-4a32-99b0-78d3b6e51143.png)

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

### Best practices

#### Updating migration scripts

It is important to keep in mind that once a script has run, it will not be re-run on the same database even if it changes.  As a result, if a mistake is made in a migration script, it should be addressed by creating a new migration script that fixes the problem, deleting the old script if necessary.  If the script is simply modified without changing the name, it will not be re-run on any database that already had the script applied.

#### Ttansitioning from manual to DbUp

_Migrations\/Script0000 - prevent incorrect execution.sql_
```sql
if OBJECT_ID(N'MyTable') IS NOT NULL
BEGIN
	throw 51000, 'Baseline migrations are only valid in a fresh database, but it was detected that other migrations have already run.  You must either (1) reset the database or (2) manually insert script names into the SchemaVersions table to skip them.', 1
END
```

#### Stored procedures and functions

Stored procedures cannot be incrementally updated.  They need to be completely replaced with an `ALTER PROCEDURE` command.  As a result, migration scripts to update a stored procedure are essentially going to be mass copy-paste versions of the previous migration script with some changes.  This can make it inconvenient to compare the stored procedure to the last version to see exactly what changed, and creates a lot of redundancy.

To address this issue, the strategy that I use is to simply update the previous migration script for that stored procedure with the changes that I want, and then also update the name of that script.  This will cause the script to be run again, and makes it easy to see the diff in git.  Additionally, there will only be one copy of the stored procedure in the migration scripts at a time, instead of a bunch of old and redundant versions.  The only caveat is that it is necessary to start the script with `CREATE OR ALTER PROCEDURE` to accommodate both updating the procedure, as well as creating it in a fresh database.