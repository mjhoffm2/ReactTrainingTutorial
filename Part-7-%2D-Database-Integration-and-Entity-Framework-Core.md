[[_TOC_]]

## Overview

This part of the tutorial is continued from [Part 6 - Setting up UI with .NET Core instead of Node](/Part-6-%2D-Setting-up-UI-with-.NET-Core-instead-of-Node).  However, it is not necessary to have followed the tutorial up to this point.  The only prerequisite for this part of the tutorial will be to have some sort of .NET Core application already set up.

In this part of the tutorial will be going over setting up a sql database for our Slack Training App, using that to create an Entity Framework Database Context.  We will also be going over the configuration for how to connect to our database, as well as creating some example service calls using Entity Framework Core.  The process for starting with a sql database, and creating a C# database context from it is referred to as the "database first" approach.  The alternative is referred to as "code first", which is a valid approach but I will not be going over it in this tutorial.

## Set up SQL Database

### Overview

For this part of the tutorial, we will set up a very simple database with just 4 tables.  Here is an ER diagram for these tables:

![image.png](/.attachments/image-521990cf-e13f-46e2-820f-8f75f57dcc42.png)

For more information about the model we will be using, please see [Slack Training App - Tutorial Overview](/Slack-Training-App-%2D-Tutorial-Overview).

### Getting Required Tools

The first thing you will need is a database server.  For this tutorial, I will be using SQL Server Express 2017, which you can [download from Microsoft](https://www.microsoft.com/en-us/sql-server/sql-server-editions-express).  This should Just Work™ without much additional configuration.  This will run in the background of your computer as a service.

You will also need something like [SQL Server Management Studio](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-2017) to interact with your database.  This is the software you will use to manage your SQL Server Express instance, run queries, and update your schema.  If you require additional help getting a database server set up, you will need to do some googling.

### Create Schema

To save you some time, I will provide a zip containing two sql scripts to get started: [SlackTraining init sql.zip](/.attachments/SlackTraining%20init%20sql-c82d9fcf-c464-4f68-b037-a2ed32b70f56.zip)

The 'init db' script will create an empty database on the database server called `SlackTraining`.  You could easily make this database yourself if you want.  The other 'init ddl' script will create the four tables you see in the diagram above, as well as the indices and foreign key constraints between them.  The `SlackTraining` database must exist before you run the 'init ddl' script.