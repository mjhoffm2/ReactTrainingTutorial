# Part 8 - Establishing a Basic API

## Overview

This part of the tutorial is continued from [Part 7 - Database Integration and Entity Framework Core](/Part-7-%252D-Database-Integration-and-Entity-Framework-Core).

In the previous part of the tutorial, we already set up an API controller with two basic endpoints.  In this part of the tutorial, we are going to extend that API by making our channel manager service and controller more interesting.  We are also going to set up a system for setting response codes for when things go wrong.

## Setting Response Codes

### Overview

Right now, our API will return a 500 Internal Server Error if something goes wrong with the request.  However, if the error stemmed from a problem with the input provided by the user, this should be a 4XX response code instead.  These response codes can be set by returning the appropriate `IActionResult` in the controller methods.  For example, a service call that throws an `ArgumentException` due to invalid user input could be caught by the controller, which would return a `BadRequest();` response instead of an `Ok();` or `Json();` response.

However, I believe that wrapping every single service call in every controller method in a try-catch block and handing each possible exception is completely impractical.  Instead, I would like to be able to put a single try-catch block somewhere in the application and have it handle all exceptions without the controller endpoints needing to deal with it.  Additionally, I want a standardized way of mapping exceptions thrown by service methods to the correct response codes.  While an `ArgumentException` usually means that there was a problem with user input, it can also be thrown for other reasons, which should still be identified as a 500 error.

My solution to this is to have a special set of `ApiException` classes, and a middleware which catches them.

### File Structure

![image.png](/.attachments/image-5ee9103a-961e-4c9b-9f02-e3e238811079.png)

### Response Model

When we send an error to the user, it is helpful to have a consistent and understandable error format that can be parsed by the front end code.  By default, ASP.NET Core will generate html pages as the response bodies when an error happens.  However, since we are making an API, it would be more useful if we produced a consistently shaped JSON response instead.  We will define the shape of the response body in a new class.

_Models\/API\/ApiExceptionResponse.cs_
```cs
namespace React_Demo.Models.API
{
	public class ApiExceptionResponse
	{
		public string Type { get; set; }
		public string Message { get; set; }
	}
}
```

### Exception Classes

Instead of throwing generic exceptions like `ArgumentException`, our service endpoints will throw special exceptions which can be mapped to specific status codes.  For this example, I will create three special exception types (`NotFoundException`, `BadRequestException`, and `UnauthorizedException`), which all extend the same base `ApiException` class.

_Util\/ApiExceptions.cs_
```cs
using System;

namespace React_Demo.Util
{
	public abstract class ApiException : Exception
	{
		public ApiException() { }
		public ApiException(string message) : base(message) { }
		public ApiException(string message, Exception inner) : base(message, inner) { }
	}

	public class NotFoundException : ApiException
	{
		public NotFoundException() { }
		public NotFoundException(string message) : base(message) { }
		public NotFoundException(string message, Exception inner) : base(message, inner) { }
	}

	public class BadRequestException : ApiException
	{
		public BadRequestException() { }
		public BadRequestException(string message) : base(message) { }
		public BadRequestException(string message, Exception inner) : base(message, inner) { }
	}

	public class UnauthorizedException : ApiException
	{
		public UnauthorizedException() { }
		public UnauthorizedException(string message) : base(message) { }
		public UnauthorizedException(string message, Exception inner) : base(message, inner) { }
	}
}
```

### Exception Middleware

When one of these exceptions is thrown by our application, we will handle them by adding a middleware to our request pipeline.  A decent reference for this can be found here: https://blogs.msdn.microsoft.com/brandonh/2017/07/31/using-middleware-to-trap-exceptions-in-asp-net-core/ or https://code-maze.com/global-error-handling-aspnetcore/

The idea is that we will wrap the entire pipeline in a big try-catch, and then catch exceptions that extend ApiException.  In the handler for this exception, we will set the appropriate status code, and write the response body with properly formatted JSON.  If we don't recognize the error, then we keep the 500 status code.

_Util\/ExceptionMiddleware.cs_
```cs
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using Newtonsoft.Json.Serialization;
using React_Demo.Models.API;
using System;
using System.Net;
using System.Threading.Tasks;

namespace React_Demo.Util
{
	public class ExceptionMiddleware
	{
		private readonly RequestDelegate next;
		private ILogger log;

		public ExceptionMiddleware(RequestDelegate next, ILogger<ExceptionMiddleware> log)
		{
			this.next = next;
			this.log = log;
		}

		public async Task Invoke(HttpContext context)
		{
			try
			{
				await next(context);
			}
			catch (ApiException e)
			{
				log.LogWarning(e, $"Handled API exception: {e.Message}");
				context.Response.ContentType = "application/json";

				if (e is NotFoundException)
				{
					context.Response.StatusCode = (int)HttpStatusCode.NotFound;
				}
				else if (e is BadRequestException)
				{
					context.Response.StatusCode = (int)HttpStatusCode.BadRequest;
				}
				else if (e is UnauthorizedException)
				{
					//Note: 401 Unauthorized actually means "Not Authenticated", so we want 403 "Forbidden" instead
					context.Response.StatusCode = (int)HttpStatusCode.Forbidden;
				}
				else
				{
					context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
				}

				ApiExceptionResponse errObj = new ApiExceptionResponse()
				{
					Type = "ApiException",
					Message = e.Message
				};

				await context.Response.WriteAsync(JsonConvert.SerializeObject(errObj, new JsonSerializerSettings()
				{
					ContractResolver = new CamelCasePropertyNamesContractResolver()
				}));
			}
			catch (Exception)
			{
				context.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
				throw;
			}
		}
	}

	public static class ExceptionStatusCodeExtensions
	{
		/// <summary>
		/// Causes specific uncaught exceptions to generate status codes and api error responses instead of the normal error handling
		/// </summary>
		public static IApplicationBuilder UseExceptionStatusCodes(this IApplicationBuilder builder)
		{
			return builder.UseMiddleware<ExceptionMiddleware>();
		}
	}
}
```

We use this middleware by creating an extension method called `UseExceptionStatusCodes` on the `IApplicationBuilder` class, and then invoke it in the `Configure` method of our `Startup` class.

_Startup.cs_
```cs
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
	//...

	//This middleware turns certain uncaught exceptions into status codes/error responses for the api
	app.UseExceptionStatusCodes();

	//anything which might generate an API Exception should be down here
}
```

This middleware should be earlier in the pipeline than any other middleware that could produce an `ApiException`.  It should also be added later than any middleware which relies on accurate status codes and responses.

## Updating the Channel Manager Service

Next up, it is time to update our service methods to utilize the new exception status code system we made, as well as add some new features.

### Channel Search

Let's extend the method that fetches all channels to include a search string and paging options.  Let's also take into account whether or not a channel is public and whether or not the channel has been marked as deleted.  Ideally, any private channels that a user is a member of would also show up, but we don't yet have a concept of users or authentication built, so we will ignore this for now.

_Services\/ChannelManagerService.cs_
```cs
public async Task<List<Channel>> GetChannels(string search, int limit, int offset)
{
	return await context.Channel
		.Where(channel => channel.IsPublic)
		.Where(channel => !channel.IsSoftDeleted)
		.Where(channel => search == "" || channel.DisplayName.StartsWith(search) || channel.Description.Contains(search))
		.Skip(offset)
		.Take(limit)
		.ToListAsync();
}
```

### Getting a Single Channel

Next, let's add an endpoint that will return a channel given an id, regardless of whether or not the channel is public or has been marked as deleted.  We can also make use of the new `NotFoundException` that we created in the event a bad channelId is given.

_Services\/ChannelManagerService.cs_
```cs
public async Task<Channel> GetChannel(long channelId)
{
	Channel channel = await context.Channel.FindAsync(channelId);

	if (channel == null)
	{
		throw new NotFoundException("No channel exists with the id: " + channelId);
	}
	return channel;
}
```

### Adding a Channel

In the previous part of the tutorial, we already created a service method for adding a channel.  For consistency with other methods in this service class, I'm going to change it to take a `Channel` as a parameter, and also have it return the channel that is created.

Please keep in mind that the channel provided by the user is the deserialized object from the request body provided by the user.  We don't want to just add this directly to the database context, since we probably don't want to let the user directly set all of the fields on the object.  Instead, we pick and choose the specific fields that we want to let the user set, and create a new `Channel` object with those fields set.  It is particularly important to not set the primary key `ChannelId` on the channel being created, since that is set to be an auto generated identity column.

After we add the channel to the context and then save the changes to the database, we return the channel that we created, which will now have its primary key set.  It is useful to return the Channel from a POST method since it will inform the client what the generated primary key is, which can then be used immediately.

_Services\/ChannelManagerService.cs_
```cs
public async Task<Channel> AddChannel(Channel channel)
{
	string displayName = channel.DisplayName;
	string description = channel.Description;
	bool isPublic = channel.IsPublic;

	if (String.IsNullOrWhiteSpace(displayName))
	{
		throw new BadRequestException("Channel display name cannot be empty");
	}

	Channel channelToCreate = new Channel()
	{
		DisplayName = displayName,
		Description = description,
		IsPublic = isPublic
	};

	context.Channel.Add(channelToCreate);

	await context.SaveChangesAsync();

	return channelToCreate;
}
```

### Updating a Channel

Updating a channel is done by looking up the existing channel by `ChannelId`, and updating the fields on it to match the provided channel.  Again, we only want to allow the user to modify certain fields, so we can't just take all the changes directly.  Instead, we will check to make sure the channel being modified actually exists, validate the input from the user, and then we can update the channel.  In this example, I am returning the updated channel from the service method, but that isn't really necessary since the client should know exactly what to expect in the new channel.  It is fine if a PATCH endpoint simply returns an empty 200 OK response.

_Services\/ChannelManagerService.cs_
```cs
public async Task<Channel> UpdateChannel(Channel channel)
{
	long channelId = channel.ChannelId;
	string displayName = channel.DisplayName;
	string description = channel.Description;
	bool isPublic= channel.IsPublic;

	//the channel provided as an argument is simply a deserialized object from the user
	//we need to obtain the tracked entity from EF Core
	Channel oldChannel = await context.Channel.FindAsync(channelId);

	if (oldChannel == null)
	{
		throw new NotFoundException("No channel exists with the id: " + channelId);
	}

	if(String.IsNullOrWhiteSpace(displayName))
	{
		throw new BadRequestException("Channel display name cannot be empty");
	}

	oldChannel.DisplayName = displayName;
	oldChannel.Description = description;
	oldChannel.IsPublic = isPublic;

	await context.SaveChangesAsync();

	return oldChannel;
}
```

### Deleting a Channel

When deleting a channel, we only need the primary key.  None of the other information is of any use, so we can just take a `long channelId` instead of an entire `Channel channel` parameter.  There is also nothing to return (it was deleted).  To actually perform the delete, we need to obtain the channel from the database to make sure it is valid.  Then, we can call `context.Channel.Remove(oldChannel);`, which will mark it as deleted.  Finally, we call `await context.SaveChangesAsync();`, which will actually delete the corresponding row from the database.

_Services\/ChannelManagerService.cs_
```cs
public async Task DeleteChannel(long channelId)
{
	Channel oldChannel = await context.Channel.FindAsync(channelId);

	if (oldChannel == null)
	{
		throw new NotFoundException("No channel exists with the id: " + channelId);
	}

	if (oldChannel.IsGeneral)
	{
		throw new UnauthorizedException("You do not have permission to delete this channel");
	}

	context.Channel.Remove(oldChannel);

	await context.SaveChangesAsync();
}
```

We can also make use of the new `UnauthorizedException` to prevent someone from accidentally deleting the general channel.  Don't confuse it with the `UnauthorizedAccessException`, which is different.

## Updating the Channel Manager Controller

We will create a total of 5 endpoints in our api:
1. Search for public channels, `GET /api/channels`
2. Get a specific channel, `GET /api/channels/:channelId`
3. Add a channel, `POST /api/channels`
4. Update a channel, `PATCH /api/channels`
5. Delete a channel, `DELETE /api/channels/:channelId`

You could make the case that the PATCH request to update a channel should also have a `:channelId` in the url path.  However, that would be redundant with the primary key that should already be present in the request body, so I chose to omit it in this example.  Similarly, the DELETE request could accept a request body containing the channel to be deleted rather putting the primary key in the url path, to make it more consistent with other requests.  However, having request bodies in DELETE requests is generally frowned upon.

We already have a service method for each one of our api endpoints, so all we need to do is handle the interface between the http request and the service methods.  Most of these are pretty trivial.  Parameters from the url path are immediately available as parameters.  Parameters annotated with `[FromQuery]` indicate that the parameter will be pulled from the query string in the url.  Parameters annotated with `[FromBody]` indicate that they will be deserialized from the request body.  If the request has the correct `application/json` header, then this is automatic.  For our GET and POST endpoints, we will return a JSON response.  For the PATCH and DELETE endpoints, we will just return an empty `200 OK` response.

Below is the final code for the controller.

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
		public async Task<IActionResult> GetChannels([FromQuery] string search = "", [FromQuery] int limit = 20, [FromQuery] int offset = 0)
		{
			var channels = await channelService.GetChannels(search, limit, offset);
			return Json(channels);
		}

		[HttpGet("{channelId:long}")]
		public async Task<IActionResult> GetChannel(long channelId)
		{
			var channel = await channelService.GetChannel(channelId);
			return Json(channel);
		}

		[HttpPost]
		public async Task<IActionResult> AddChannel([FromBody] Channel channel)
		{
			var createdChannel = await channelService.AddChannel(channel);
			return Json(createdChannel);
		}

		[HttpPatch]
		public async Task<IActionResult> UpdateChannel([FromBody] Channel channel)
		{
			await channelService.UpdateChannel(channel);
			return Ok();
		}

		[HttpDelete("{channelId:long}")]
		public async Task<IActionResult> DeleteChannel(long channelId)
		{
			await channelService.DeleteChannel(channelId);
			return Ok();
		}
	}
}
```

## Try it out

To run any of these demos, run the application using the debugger, and wait for it to start.  You should see a web browser launch as part of the debugger. 
 Simply copy and paste the JavaScript snippet into the developer tools in the web browser that appears.  The JavaScript snippet will make a request to the server with the provided json payload (if applicable), and will print out the response, if any.

### Empty name

First, let's test out our error middleware by trying to create a channel with a blank name.

```js
var data = JSON.stringify({
  "displayName": "" //empty name
});

var xhr = new XMLHttpRequest();
xhr.withCredentials = true;

xhr.addEventListener("readystatechange", function () {
  if (this.readyState === 4) {
    var log = (this.status >= 200 && this.status < 300) ? console.log : console.error;

    log(this.status + " " + this.statusText);

    try {
      log(JSON.parse(this.response || this.responseText));
    } catch(e) {
      //ignore
    }
  }
});

xhr.open("POST", "/api/channels");
xhr.setRequestHeader("Content-Type", "application/json");
xhr.setRequestHeader("Cache-Control", "no-cache");

xhr.send(data);
```

If all goes as intended, you should see the following response:

![image.png](/.attachments/image-8e4b0efb-b860-4d9e-9acb-7fb1226f7302.png)

This is good, we have a proper 400 Bad Request response instead of a 500 Internal Server Error.  Additionally, we have a JSON response body that we can work with if we want to.

Next, add a channel for real.  I'll call it 'my channel' and give it a description

```js
var data = JSON.stringify({
  "displayName": "my channel",
  "description": "only cool people allowed"
});

var xhr = new XMLHttpRequest();
xhr.withCredentials = true;

xhr.addEventListener("readystatechange", function () {
  if (this.readyState === 4) {
    var log = (this.status >= 200 && this.status < 300) ? console.log : console.error;

    log(this.status + " " + this.statusText);

    try {
      log(JSON.parse(this.response || this.responseText));
    } catch(e) {
      //ignore
    }
  }
});

xhr.open("POST", "/api/channels");
xhr.setRequestHeader("Content-Type", "application/json");
xhr.setRequestHeader("Cache-Control", "no-cache");

xhr.send(data);
```

![image.png](/.attachments/image-be01c766-6e57-4d16-917f-e117189f3f97.png)

When I ran it, I got a generated channelId of 13.  You will probably get something different.

Next let's test out the endpoint for searching for channels.

```js
var xhr = new XMLHttpRequest();
xhr.withCredentials = true;

xhr.addEventListener("readystatechange", function () {
  if (this.readyState === 4) {
    var log = (this.status >= 200 && this.status < 300) ? console.log : console.error;

    log(this.status + " " + this.statusText);

    try {
      log(JSON.parse(this.response || this.responseText));
    } catch(e) {
      //ignore
    }
  }
});

xhr.open("GET", "/api/channels");
xhr.setRequestHeader("Cache-Control", "no-cache");

xhr.send();
```

![image.png](/.attachments/image-cfeeac6e-64ff-4cac-b909-b64e59aaad9f.png)

We got a 200 OK response, but no channels came back.  This is because our channel has `isPublic` set to `false`.  Let's modify our channel so that it shows up when we search for it.  We can make use of our PATCH endpoint for this.  You will need to update this snippet to have the correct channelId in the JSON data.

```js
var data = JSON.stringify({
  "channelId": 13,
  "isPublic": true
});

var xhr = new XMLHttpRequest();
xhr.withCredentials = true;

xhr.addEventListener("readystatechange", function () {
  if (this.readyState === 4) {
    var log = (this.status >= 200 && this.status < 300) ? console.log : console.error;

    log(this.status + " " + this.statusText);

    try {
      log(JSON.parse(this.response || this.responseText));
    } catch(e) {
      //ignore
    }
  }
});

xhr.open("PATCH", "/api/channels");
xhr.setRequestHeader("Content-Type", "application/json");
xhr.setRequestHeader("Cache-Control", "no-cache");

xhr.send(data);
```

![image.png](/.attachments/image-079e3dbc-d9a6-4c35-b882-1bf5bc2f8864.png)

Still foiled!  In our request, we neglected to include the `displayName` and `description` fields, because we only wanted to update the `isPublic` field.  However, our API will try replace those fields regardless.  If we were to set up our API so that it ignored null values, that would prevent us from being able to explicitly set fields to null, which we likely want to do sometimes.  When we deserialize an object in C#, there is no distinction between null and undefined, so trying to propagate information about excluded fields (as opposed to intentionally null fields) is complicated.  For now, let's just include them.

```js
var data = JSON.stringify({
  "channelId": 13,
  "displayName": "my channel",
  "description": "only cool people allowed",
  "isPublic": true
});

var xhr = new XMLHttpRequest();
xhr.withCredentials = true;

xhr.addEventListener("readystatechange", function () {
  if (this.readyState === 4) {
    var log = (this.status >= 200 && this.status < 300) ? console.log : console.error;

    log(this.status + " " + this.statusText);

    try {
      log(JSON.parse(this.response || this.responseText));
    } catch(e) {
      //ignore
    }
  }
});

xhr.open("PATCH", "/api/channels");
xhr.setRequestHeader("Content-Type", "application/json");
xhr.setRequestHeader("Cache-Control", "no-cache");

xhr.send(data);
```

![image.png](/.attachments/image-42ac64aa-ee83-4cc7-a5a3-7bd403bfd983.png)

Now that worked.  We can re-run our previous request to search for all channels and finally see it come back.

![image.png](/.attachments/image-69d3db4b-2e56-40e7-9764-0ef0def2c01b.png)

## Download Source

The source code up to this point can be found here:

https://github.com/mjhoffm2/react-demo/tree/Part-8

The code for this part of the tutorial can be found in the `.net core 2.1` folder, on the `Part-8` branch.