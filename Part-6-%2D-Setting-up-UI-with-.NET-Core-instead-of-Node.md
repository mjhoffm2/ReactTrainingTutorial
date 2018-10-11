[[_TOC_]]

## Overview

This part of the tutorial is continued from [Part 5 - Bundling for Production](/Part-5-%2D-Bundling-for-Production).

So far, we have set up our project to be served from a Node server running Express.  We haven't actually implemented any business logic using this server yet, we just use it to serve files and support hot module replacement.  In this part of the tutorial, I will be going over how to accomplish the same thing using .NET Core, and will be migrating the existing UI code to that project.

## Initialize the .NET Core project

I would recommend initializing the project from a template, and then modifying the template to suit your needs.  If you already have set up some kind of api server using .NET Core, then you can work from there.  In this tutorial, I will be using .NET Core 2.1.

First, you will need to install the .NET Core SDK
https://www.microsoft.com/net/download/dotnet-core/2.1
I am using v2.1.5 - SDK 2.1.403

![image.png](/.attachments/image-ed7cea1a-77f9-4e8e-8286-5d57c7acb92f.png)

![image.png](/.attachments/image-5c11032d-54ef-40ab-afcc-e6e6cd4f4136.png)

Warning: If you check the box that says "Create new Git repository", and the folder or one of its parents already has an existing .git folder, then you may end up corrupting the existing git repository.

![image.png](/.attachments/image-8f1c5101-06bc-4d03-b3bd-e687b8f50963.png)

To help you get started, here is a zip containing a .gitignore file that you can place in the same directory as your .csproj file: [gitignore.zip](/.attachments/gitignore-f78fa947-dee4-4e71-8270-5aedfe985437.zip).