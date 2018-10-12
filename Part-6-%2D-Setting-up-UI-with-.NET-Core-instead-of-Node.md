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

In this part of the tutorial, I will not be using any dependencies except for the ones that were already included in the API template project.  Only 3 are used:

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

In this section, I have also made it so the `node_modules` folder will not show up in Visual Studio when it is created.  A complete .csproj is available below, but adding these two properties is completely mandatory before adding any of our front end code.  If you add any typescript files before adding the `<TypeScriptCompileBlocked />` flag, then JavaScript files will be created which will get picked up by your build instead of your actual TypeScript source code.  This can result in some extremely confusing situations with stale data.

Before we can finish updating the .csproj file, we will need to decide our project structure.  Here is the project structure that I decided to go with.  In a .NET Core application, the `wwwroot` folder has special meaning, and any files placed in it are publicly accessible.  Additionally, the `Views` folder has special meaning, it will be searched through for `cshtml` template files, which we will be using to serve our index html page.

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
3. Configures which files to copy and which to ignore for publishing.  In this case, all files in the ClientApp folder will be excluded from the publish.  All files in the wwwroot/dist folder are included in the publish.

Note that this only runs `npm install` if the `node_modules` folder doesn't already exist.  In the future, if you update your npm dependencies, it will not automatically run `npm install` when you debug your application.  You will need to manually run `npm install` from the command line after updating your npm dependencies before you start debugging.

### Migrate Source Code and Configuration

As far as actually moving the files, we will simply take the contents of the node application's `src/web` folder, and place them in the `ClientApp` folder.  We will not be using the code from the node server, or the old .html files.  We will be placing the `webpack.config.js`, `tsconfig.js`, and `package.json` files all in the same directory as the .csproj file.  You can copy the `package-lock.json` file too if you wish, otherwise it will be generated again.  We will need to update our webpack configuration to reflect the new structure (and the lack of a node server), as well as update our dependencies in the `package.json` file.

Here is the updated `package.json` file I am using:

_package.json_
```json
{
  "name": "react-demo",
  "version": "1.0.0",
  "description": "",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build": "node_modules/.bin/webpack --mode=development",
    "build:prod": "node_modules/.bin/webpack --mode=production"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@types/es6-promise": "^3.3.0",
    "@types/react": "^16.4.13",
    "@types/react-bootstrap": "^0.32.13",
    "@types/react-dom": "^16.0.7",
    "@types/react-redux": "^6.0.9",
    "@types/react-router": "^4.0.30",
    "@types/react-router-dom": "^4.3.1",
    "@types/webpack-hot-middleware": "^2.16.4",
    "aspnet-webpack": "^3.0.0",
    "awesome-typescript-loader": "^5.2.1",
    "bootstrap": "^3.3.7",
    "connected-react-router": "^4.5.0",
    "css-loader": "^1.0.0",
    "es6-promise": "^4.2.5",
    "file-loader": "^2.0.0",
    "mini-css-extract-plugin": "^0.4.4",
    "react": "^16.5.0",
    "react-bootstrap": "^0.32.4",
    "react-dom": "^16.5.0",
    "react-hot-loader": "^4.3.11",
    "react-redux": "^5.0.7",
    "react-router": "^4.3.1",
    "react-router-dom": "^4.3.1",
    "redux": "^4.0.0",
    "style-loader": "^0.23.0",
    "typescript": "^3.0.3",
    "url-loader": "^1.1.1",
    "webpack-dev-middleware": "^3.4.0",
    "webpack-hot-middleware": "^2.24.3",
    "whatwg-fetch": "^3.0.0"
  },
  "devDependencies": {
    "webpack": "^4.17.2",
    "webpack-cli": "^3.1.0"
  }
}
```

The major changes are:
1. Removing anything that was only used by the node server running express.  I am keeping the webpack-dev-middleware, since it will still be required later.
2. I have added the `aspnet-webpack` package, which will also be required later.

_webpack.config.js_
```js
const path = require('path');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

var config = (env, options) => {
    let isProduction;

    if (env && env.NODE_ENV && env.NODE_ENV !== 'development') {
        isProduction = true;
    } else if (options && options.mode === 'production') {
        isProduction = true;
    } else {
        isProduction = false;
    }

    return {
        name: 'web',
        mode: isProduction ? 'production' : 'development',
        entry: {
            client: './ClientApp/boot-client.tsx'
        },
        output: {
            path: path.resolve(__dirname, './wwwroot/dist'),
            publicPath: '/dist/',
            filename: 'bundle.js'
        },
        resolve: {
            //automatically infer '.ts' and '.tsx' when importing files
            extensions: ['.js', '.jsx', '.ts', '.tsx']
        },
        module: {
            rules: [
                {
                    test: /\.css$/,
                    use: isProduction ?
                        [MiniCssExtractPlugin.loader, 'css-loader'] :
                        ['style-loader', 'css-loader']
                },
                {
                    test: /\.tsx?$/,
                    include: path.resolve(__dirname, "./ClientApp/"),
                    loader: "awesome-typescript-loader"
                },
                {
                    test: /\.(png|jpg|jpeg|gif|svg|ttf|otf|woff|woff2|eot)$/,
                    loader: 'url-loader?limit=4096'
                }
            ]
        },

        //see https://webpack.js.org/configuration/devtool/ for options
        devtool: isProduction ? "source-map" : "cheap-module-eval-source-map",

        plugins: isProduction ? [ new MiniCssExtractPlugin() ] : []
    };
};
module.exports = config;
```

Important changes:
1. Instead of just checking for `options.mode === 'production'`, we are now also checking `env.NODE_ENV !== 'development'`.  If either of these are true, then we are in production mode.  The reason for this is because from .NET Core, we can easily provide the `env` parameter but not the `mode` parameter.
2. We have completely removed the 'server' configuration, and are now returning only the web configuration.
3. We are setting the 'mode' field to match the value of `isProduction`.  This is required by webpack v4 when using `env.NODE_ENV` instead of `options.mode`.
4. Removed the hot module entry point and gave the boot-client entry point the name 'client'.  This will be handled automatically by .NET Core.
5. Adjusted the output to reflect the new output directory and public path.
6. Removed the hot module replacement plugin during development.  This will be handled automatically by .NET Core.

## Configuring ASP.NET Core to host the code.

### Webpack Dev Middleware

Just like on the node server, we will be using webpack dev middleware to build and serve our code during development.  This will be configured as part of our HTTP request pipeline in the `Configure` method of our `Startup` class.  Simply add the Webpack Dev Middleware, which is available as part of AspNetCore.SpaServices.  This middleware will run a node server in the background, which will basically do the same thing that our previous node server was doing.

_Startup.cs_
```cs
if (env.IsDevelopment())
{
	//do not attempt to use hot reloading on staging or production
	app.UseWebpackDevMiddleware(new WebpackDevMiddlewareOptions
	{
		HotModuleReplacement = true,
		EnvParam = new { NODE_ENV = "development" }
	});
}

app.UseStaticFiles();
```

It is important to make sure that you add this middleware _before_ the call to `app.UseStaticFiles();`.  It is also important that this middleware is only used in development mode.  In this middleware, we have configured hot module replacement to be enabled, as well as specified the value of `env.NODE_ENV` that will be provided to the webpack configuration.

It is also important that the `Startup.cs` file and the `webpack.config.js` file for your project are both located in the same directory as the .csproj file.  While it is possible to provide `ConfigFile` and `ProjectPath` parameters to the webpack dev middleware options, I was not successful in getting these to work properly.  Additionally, there is a `ReactHotModuleReplacement` option, but this doesn't do anything useful as far as I can tell.  The `react-hot-loader` that we use on the front end doesn't require anything special configured on the back end to work.

### Hosting the Index .html Page

Just like on the node server running express, we will need our server to host some kind of .html page which will kick start our react application.  To do this, we will be using .cshtml pages.  If you want to learn more about creating and configuring these templates, you will need to do some googling.  I am just going to dump them here and talk about roughly what they are doing.

My final .cshtml files are as follows:

_Views/\_ViewStart.cshtml_
```
@{
    Layout = "_Layout";
}
```

_Views/\_ViewImports.cshtml_
```
@using React_Demo
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@addTagHelper *, Microsoft.AspNetCore.SpaServices
```

_Views/Shared/\_Layout.cshtml_
```
@inject Microsoft.AspNetCore.Hosting.IHostingEnvironment hostingEnv
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link id="favicon-shortcut-icon" rel="shortcut icon" href="~/favicon.ico?v=1" type="image/x-icon" />
    <link id="favicon-icon" rel="icon" href="~/favicon.ico?v=1" type="image/x-icon" />
    <title>React Demo</title>
    <base href="~/" />
    @if (hostingEnv.EnvironmentName != "Development")
    {
        <link href="~/dist/client.css" asp-append-version="true" rel="stylesheet">
    }
</head>
<body>
    @RenderBody()

    @RenderSection("scripts", required: false)
</body>
</html>
```

_Views/Shared/Error.cshtml_
```
@{
    ViewData["Title"] = "Error";
}

<h1 class="text-danger">Error.</h1>
<h2 class="text-danger">An error occurred while processing your request.</h2>

@if (!string.IsNullOrEmpty((string)ViewData["RequestId"]))
{
    <p>
        <strong>Request ID:</strong> <code>@ViewData["RequestId"]</code>
    </p>
}

<h3>Development Mode</h3>
<p>
    Swapping to <strong>Development</strong> environment will display more detailed information about the error that occurred.
</p>
<p>
    <strong>Development environment should not be enabled in deployed applications</strong>, as it can result in sensitive information from exceptions being displayed to end users. For local debugging, development environment can be enabled by setting the <strong>ASPNETCORE_ENVIRONMENT</strong> environment variable to <strong>Development</strong>, and restarting the application.
</p>
```

_Views/Home/Index.cshtml_
```
<div id="root">Loading...</div>

@section scripts {
    <script src="~/dist/bundle.js" asp-append-version="true"></script>
}
```
![image.png](/.attachments/image-e572aa55-94fb-493c-b75b-e2be9ff5b7d8.png)

The two files that actually assemble the content on the page are the `_Layout.cshtml` and `Index.cshtml` files.  For our application, we could probably just merge these two files together.  Make sure that you use the correct namespace in the `_ViewImports.cshtml_` file, since you probably didn't name your project 'React Demo'.

Note that in the `_Layout.cshtml` file, we are checking to see if the current environment is "Development" or not.  In any environment other than local debugging, we will need to include our .css source file separately.  If you recall from [Part 5 - Bundling for Production](/Part-5-%2D-Bundling-for-Production), in the section about separating css, we needed to provide two separate .html files due to the fact that our .css stylesheets are included inline in development, and as a separate `main.css` file in production.  In our new build process using .NET Core, we have a similar situation.  The main differences are that we are checking the environment directly in the template instead of having separate .html files, and our stylesheet will be called `client.css` due to the fact that is how we named the entry point in the webpack configuration.

Next, we need to actually tell our application when to serve the page.  We need a controller that will serve it.  We will create a `HomeController.cs` file in the `Controllers` folder to do this.

_HomeController.cs_
```cs
using System.Diagnostics;
using Microsoft.AspNetCore.Mvc;

namespace React_Demo.Controllers
{
	public class HomeController : Controller
	{

		public HomeController()
		{

		}
		
		public IActionResult Index()
		{
			return View();
		}

		public IActionResult Error()
		{
			ViewData["RequestId"] = Activity.Current?.Id ?? HttpContext.TraceIdentifier;
			return View();
		}
	}
}
```

Again, make sure to correct the namespace since you probably didn't name your project 'React Demo'.

Finally, we need to hook up this controller to our HTTP request pipeline.  We will do this by adding the MVC middleware at the end of our request pipeline in the `Configure` method of our `Startup.cs` class.

_Startup.cs_
```cs
app.UseMvc(routes =>
{
	routes.MapRoute(
		name: "default",
		template: "{controller}/{action=Index}/{id?}");

	//If no server route matched, assume that this is a route handled by the client side router in the react app
	//Note: urls that look like static resources will not be handled by this, such as urls that end in file extensions
	routes.MapSpaFallbackRoute(
		name: "spa-fallback",
		defaults: new { controller = "Home", action = "Index" });
});
```

The first call to `routes.MapRoute` will configure our api routes, which we will set up in a later part of the tutorial.  The following call to `routes.MapSpaFallbackRoute` will invoke our HomeController and serve our .html page for any routes that aren't handled by the api controllers.  Note that this will also skip routes that look like static files, just like we had configured previously on the node server running express.

The final `Startup.cs` file should look something like this:

_Startup.cs_
```cs
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.AspNetCore.SpaServices.Webpack;

namespace React_Demo
{
	public class Startup
	{
		public Startup(IConfiguration configuration)
		{
			Configuration = configuration;
		}

		public IConfiguration Configuration { get; }

		// This method gets called by the runtime. Use this method to add services to the container.
		public void ConfigureServices(IServiceCollection services)
		{
			services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_1);
		}

		// This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
		public void Configure(IApplicationBuilder app, IHostingEnvironment env)
		{
			if (env.IsDevelopment())
			{
				app.UseExceptionHandler("/Error");
				app.UseDeveloperExceptionPage();
			}
			else
			{
				app.UseHsts();
			}

			app.UseHttpsRedirection();

			if (env.IsDevelopment())
			{
				//do not attempt to use hot reloading on staging or production
				app.UseWebpackDevMiddleware(new WebpackDevMiddlewareOptions
				{
					HotModuleReplacement = true,
					EnvParam = new { NODE_ENV = "development" }
				});
			}

			app.UseStaticFiles();

			app.UseMvc(routes =>
			{
				routes.MapRoute(
					name: "default",
					template: "{controller}/{action=Index}/{id?}");

				//If no server route matched, assume that this is a route handled by the client side router in the react app
				//Note: urls that look like static resources will not be handled by this, such as urls that end in file extensions
				routes.MapSpaFallbackRoute(
					name: "spa-fallback",
					defaults: new { controller = "Home", action = "Index" });
			});
		}
	}
}
```

## Try it out

### Development

At this point, you should be able to try debugging your application.  You should already have a working `launchSettings.json` file, but in case you run into problems, here is mine for reference.  The port numbers are arbitrary.

_launchSettings.json_
```json
{
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "iisSettings": {
    "windowsAuthentication": false, 
    "anonymousAuthentication": true, 
    "iisExpress": {
      "applicationUrl": "http://localhost:61802",
      "sslPort":  44301
    }
  },
  "profiles": {
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

To start running the code to debug it, simply click the green play button at the top that says "IIS Express".
![image.png](/.attachments/image-6218027b-7f1a-4014-bd9a-b12f5f23ae58.png)

The first time you debug the application, it will need to run `npm install` as we configured in the .csproj file.  This can take a while, so be patient.  Additionally, you will probably be asked to install an ssl security certificate for your localhost.

Eventually, a new chrome window will open up, and you will see the application running.

![image.png](/.attachments/image-dc5e3e51-3159-4e6a-b2b0-d73281ffd939.png)

You can test out hot module reloading by making a change to the source code, and seeing the changes appear immediately in your browser.

![image.png](/.attachments/image-e156d5b6-1295-4247-a06e-be1459d2edc7.png)

Your react state should also be preserved.

Closing this window will stop the debugger, and stopping the debugger will close this window.

### Publishing

Most of the difficult configuration for publishing is already taken care of in the .csproj file that I provided.  If you run into other issues, you will need to do some googling.

To get started, right click on your project and select 'Publish...'
![image.png](/.attachments/image-0df45af7-617a-4bf4-8c79-2d3acbf4c3e9.png)

Choose the 'Folder' target
![image.png](/.attachments/image-614541cb-30e2-40ff-bf71-66ea635fcc30.png)

Before creating the profile, I recommend clicking the 'Advanced...' button and applying the following settings:
![image.png](/.attachments/image-a5f206d1-3e2d-41f0-a474-7e95f4431c4f.png)
Doing a self-contained deployment is not optimal, but it reduces the number of things that could go wrong with the deployment.

When you finish creating the profile, you should see something like this:
![image.png](/.attachments/image-ebf764e1-024a-4d3a-aa26-79ac0ef3b570.png)

At this point, you can go ahead and click the 'Publish' button.  When it finishes, you can head over to the folder you selected for publishing.  In this example, that folder is `React Demo\bin\Release\netcoreapp2.1\publish`.  From there, you can open up a command prompt and do the following:
1. Set the ASPNETCORE_ENVIRONMENT variable to something other than 'Development'.  This will have defaulted to 'Production' if you didn't set anything, so you likely don't need to do anything.
2. Run the .exe that was generated for your project.  In this example, the exe is `React Demo.exe`.  You should see something like the following:

```ps
> $env:ASPNETCORE_ENVIRONMENT = "Staging"
> & '.\React Demo.exe'

Hosting environment: Staging
Content root path: C:\Users\Matt Hoffman\git\react-demo\.net core 2.1\React Demo\bin\Release\netcoreapp2.1\publish
Now listening on: http://localhost:5000
Now listening on: https://localhost:5001
Application started. Press Ctrl+C to shut down.
```

In this example, port 5001 is the secure port, so a request to localhost:5000 should get redirected to localhost:5001.  You can open up your browser to one of these and take a look at the application working.  If you take a look at the network tab, you should see something like this:

![image.png](/.attachments/image-16b99a40-e65c-4293-b3a5-db4cb41c6d7c.png)

Based on the size of the javascript bundle and the separate css bundle, we can see our production front end resources are correctly being built and used.  The only thing that we are missing from the node server running express is the gzip compression.  However, in a .NET Core application, that is typically configured on IIS, and not on the application server.  This will be out of scope for this tutorial.

## Issues I ran into and how to address them

### Unable to connect to IIS Express

When trying to debug the application for the first time, I got an error about being "unable to connect to web server 'IIS Express'.  I am not sure exactly what caused this and how to fix it, but the issue went away after I did the following.

1. Disable the ssl port in the launch settings by removing the "sslPort" field in "iisExpress"
2. Close Visual Studio
3. Stopped IIS Express in the windows toolbar.  See the below image:
![image.png](/.attachments/image-eaed5ac9-0cbc-4918-8459-1d30f162e962.png)
4. Deleted my .vs folder
5. Reopened the solution in Visual Studio

### Stale Code when debugging

When I was debugging, I would not see my changes applied to the code.  Even when I made sure my code was getting rebuilt, I would still see an old version without my changes.  This turned out to be because I had added the TypeScript files before I added the `<TypeScriptCompileBlocked />` flag to the .csproj file.  This caused a bunch of stale .js files to be placed alongside the actual .ts and .tsx files.  The .js files were being used instead of the .ts and .tsx files.  I simply deleted all those .js files to fix the issue.

### Publish sometimes misses files in the wwwroot/dist folder

Since upgrading to webpack v4, it seems that files are written to disc asynchronously.  The MSBuild process configured in the .csproj file wasn't waiting for these files to be written by the `<Exec Command="npm run build:prod" />` command before copying files to the publish directory.  This didn't happen consistently, and after adding `<DistFiles Include="wwwroot\dist\**" />` and `<Delete Files="@(FilesToDelete)" />`, it hasn't happened to me again.  However, it is something to look out for.

### Publish sometimes hangs with thousands of errors

Every once in a while, I would encounter a problem where the publish would encounter thousands of errors about being unable to copy certain .dll files.  To address this, I simply stopped the publish with ctrl + break, and tried again.


## Download Source

The source code up to this point can be found here:

https://dev.azure.com/echeloncons/_git/Slack%20Training%20App?version=GBPart-6

The code for this part of the tutorial can be found in the `.net core 2.1` folder.
