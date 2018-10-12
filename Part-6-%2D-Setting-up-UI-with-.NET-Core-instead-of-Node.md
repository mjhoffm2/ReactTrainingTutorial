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
1. Instead of just checking for `options.mode === 'production'`, we are now also checking `env.NODE_ENV !== 'development'`.  If either of these are true, then we are in production mode.  The reason for this is because from asp.net, we can easily provide the `env` parameter but not the `mode` parameter.
2. We have completely removed the 'server' configuration, and are now returning only the web configuration.
3. We are setting the 'mode' field to match the value of `isProduction`.  This is required by webpack v4 when using `env.NODE_ENV` instead of `options.mode`.
4. Removed the hot module entry point and gave the boot-client entry point the name 'client'.
5. Adjusted the output to reflect the new output directory and public path.
6. Removed the hot module replacement plugin during development.

## Configuring ASP.NET Core to host the code.

### Webpack Dev Middleware

Just like on the node server, we will be using webpack dev middleware to build and serve our code during development.  This will be configured as part of our HTTP request pipeline in the `Configure` method of our `Startup` class.  Simply add the Webpack Dev Middleware, which is available as part of AspNetCore.SpaServices.

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

My final .cshtml files are as follows

_Views/\_ViewStart.cshtml_
```
@{
    Layout = "_Layout";
}
```

_Views/\_ViewImports.cshtml_
```
@using test2
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

The two files that actually assemble the content on the page are the `_Layout.cshtml` and `Index.cshtml` files.  For our application, we could probably just merge these two files together.

Note that in the `_Layout.cshtml` file, we are checking to see if the current environment is "Development" or not.  In any environment other than local debugging, we will need to include our .css source file separately.  If you recall from [Part 5 - Bundling for Production](/Part-5-%2D-Bundling-for-Production), in the section about separating css, we needed to provide two separate .html files due to the fact that our .css stylesheets are included inline in development, and as a separate `main.css` file in production.  In our new build process using .NET Core, we have a similar situation.  The main differences are that we are checking the environment directly in the template instead of having separate .html files, and our stylesheet will be called `client.css` due to the fact that is how we named the entry point in the webpack configuration.



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

