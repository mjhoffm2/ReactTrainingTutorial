[[_TOC_]]

## Overview

This part of the tutorial is continued from [Part 5 - Bundling for Production](/Part-5-%2D-Bundling-for-Production).

So far, we have set up our project to be served from a Node server running Express.  We haven't actually implemented any business logic using this server yet, we just use it to serve files and support hot module replacement.  In this part of the tutorial, I will be going over how to accomplish the same thing using .NET Core, and will be migrating the existing UI code to that project.

## Initialize the .NET Core project

I would recommend initializing the project from a template, and then modifying the template to suit your needs.  If you already have set up some kind of api server using .NET Core, then you can work from there.  In this tutorial, I will be using .NET Core 2.1, and Visual Studio 2017.

First, you will need to install the .NET Core SDK
https://www.microsoft.com/net/download/dotnet-core/2.1
I am using v2.1.5 - SDK 2.1.403

![image.png](/.attachments/image-ed7cea1a-77f9-4e8e-8286-5d57c7acb92f.png)

![image.png](/.attachments/image-5c11032d-54ef-40ab-afcc-e6e6cd4f4136.png)

**Warning**: If you check the box that says "Create new Git repository", and the folder or one of its parents already has an existing .git folder, then you may end up corrupting the existing git repository.  To help you get started, here is a zip containing a .gitignore file that you can place in the same directory as your .csproj file: [gitignore.zip](/.attachments/gitignore-f78fa947-dee4-4e71-8270-5aedfe985437.zip).

![image.png](/.attachments/image-8f1c5101-06bc-4d03-b3bd-e687b8f50963.png)

For this tutorial, I chose to start with the API template, since it is the most basic starting point available (besides an empty project).  This starting point should provide the most flexibility, and also provide a good reference for the widest selection of projects.  You may choose to enable/disable authentication or enable/disable HTTPS.  I have tried using the React.js and Redux templates before, but I have found them to be very volatile, and changed drastically with each minor ASP.NET Core version.

## Migrate React Application

### Overview

With this starting template, we are now going to extend it so it can build and host our React application.  Our goals are:
1. Have the .NET Core server host our React-based front end.
2. Have Visual Studio automatically build our front end for us when we debug our project.
3. Be able to still use hot module reloading with the same speed as with node.
4. Publish our code for production with the same efficiency as node.

### Verify NuGet dependencies

In this project, I will not be using any dependencies except for the ones that were already included in the API template project.  Only 3 are used:

![image.png](/.attachments/image-05b3ef5b-0144-448e-89ae-709807500f4d.png)

### Update .csproj

Before we do anything, we will need to update our .csproj file to make sure Visual Studio doesn't try to generate JavaScript files from our TypeScript source.  Update the `<PropertyGroup />` section to set `<TypeScriptCompileBlocked />` to `true`.

_YourProject.csproj_
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>netcoreapp2.1</TargetFramework>
    <TypeScriptCompileBlocked>true</TypeScriptCompileBlocked>
    <DefaultItemExcludes>$(DefaultItemExcludes);node_modules\**</DefaultItemExcludes>
    ...
  </PropertyGroup>
  ...
</Project>
```

In this section, I have also made it so the `node_modules` folder will not show up in Visual Studio when it is created.  A complete .csproj example will be available later, but adding these two properties is completely mandatory before continuing.  If you add any typescript files before adding the `<TypeScriptCompileBlocked />` flag, then JavaScript files will be created which will get picked up by your build instead of your actual TypeScript source code.  This can result in some extremely confusing situations with stale data.

Next, we will need to decide our project structure before we can continue.  Here is the project structure that I decided to go with.  In a .NET Core application, the `wwwroot` folder has special meaning, and any files placed in it are publicly accessible.  Additionally, the `Views` folder has special meaning, it will be search through for `cshtml` template files if your project is using 

![image.png](/.attachments/image-f1b2cd8f-8f8b-4b94-aaad-9b3d9a885aa3.png)

Based on this, we can finish updating our .csproj file to properly build our solution.

Here is my completed .csproj file:

_YourProject.csproj_
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp2.1</TargetFramework>
    <TypeScriptCompileBlocked>true</TypeScriptCompileBlocked>
    <TypeScriptToolsVersion>3.0</TypeScriptToolsVersion>
    <IsPackable>false</IsPackable>
    <DefaultItemExcludes>$(DefaultItemExcludes);node_modules\**</DefaultItemExcludes>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.App" />
    <PackageReference Include="Microsoft.AspNetCore.Razor.Design" Version="2.1.2" PrivateAssets="All" />
  </ItemGroup>

  <ItemGroup>
    <!-- Don't publish the SPA source files, but do show them in the project files list -->
    <Content Remove="ClientApp\**" />
  </ItemGroup>

  <ItemGroup>
    <Folder Include="wwwroot\dist\" />
  </ItemGroup>

  <Target Name="DebugEnsureNodeEnv" BeforeTargets="Build" Condition=" '$(Configuration)' == 'Debug' And !Exists('node_modules') ">
    <!-- Ensure Node.js is installed -->
    <Exec Command="node --version" ContinueOnError="true">
      <Output TaskParameter="ExitCode" PropertyName="ErrorCode" />
    </Exec>
    <Error Condition="'$(ErrorCode)' != '0'" Text="Node.js is required to build and run this project. To continue, please install Node.js from https://nodejs.org/, and then restart your command prompt or IDE." />
    <Message Importance="high" Text="Restoring dependencies using 'npm'. This may take several minutes..." />
    <Exec Command="npm install" />
  </Target>

  <Target Name="PublishRunWebpack" AfterTargets="ComputeFilesToPublish">
    <!-- As part of publishing, ensure the JS resources are freshly built in production mode -->
    <ItemGroup>
        <FilesToDelete Include="wwwroot\dist\**\*" />
    </ItemGroup>
    <Delete Files="@(FilesToDelete)" />
    <Exec Command="npm install" />
    <Exec Command="npm run build:prod" />

    <!-- Include the newly-built files in the publish output -->
    <ItemGroup>
      <DistFiles Include="wwwroot\dist\**" />
      <ResolvedFileToPublish Include="@(DistFiles->'%(FullPath)')" Exclude="@(ResolvedFileToPublish)">
        <RelativePath>%(DistFiles.Identity)</RelativePath>
        <CopyToPublishDirectory>PreserveNewest</CopyToPublishDirectory>
      </ResolvedFileToPublish>
    </ItemGroup>
  </Target>

</Project>

```

The important things this is doing:
1. Installs NPM dependencies the first time we try to build the project.
2. Cleans and builds the front end in production mode when publishing.
3. Configures which files to copy and which to ignore for publishing.  In this case, all files in the ClientApp folder will be excluded from the publish.

### Migrate Source Code and Configuration

As far as actually moving the files, we will simply take the contents of the node application's `src/web` folder, and place that in `ClientApp`.  We will not be using the code from the node server.  We will be placing the `webpack.config.js`, `tsconfig.js`, and `package.json` files all in the same directory as the .csproj file.  You can copy the `package-lock.json` file too if you wish, otherwise it will be generated again.  We will need to update our webpack configuration to reflect the new structure (and the lack of a node server), as well as update our dependencies in the `package.json` file. 