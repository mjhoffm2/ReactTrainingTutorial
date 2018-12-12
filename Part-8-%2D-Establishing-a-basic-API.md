# Part 8 - Establishing a basic API

## Overview

This part of the tutorial is continued from [Part 7 - Database Integration and Entity Framework Core](/Part-7-%2D-Database-Integration-and-Entity-Framework-Core).

In that part of the tutorial, we already set up an API controller with two basic endpoints.  In this part of the tutorial, we are going to extend that API by making our channel manager service and controller more interesting.  We are also going to set up a system for setting response codes for when things go wrong.

## Setting Response Codes

### Overview

Right now, our API will return a 500 Internal Server Error if something goes wrong with the request.  However, if the error stemmed from a problem with the input provided by the user, this should be a 4XX response code instead.  These response codes can be set by returning the appropriate `IActionResult` in the controller methods.  For example, a service call that throws an `ArgumentException` due to invalid user input could be caught by the controller, which would return a `BadRequest();` response instead of an `Ok();` or `Json();` response.

However, I believe that wrapping every single service call in every controller method in a try-catch block and handing each possible exception is completely impractical.  Instead, I would like to be able to put a single try-catch block somewhere in the application and have it handle all exceptions without the controller endpoints needing to deal with it.  Additionally, I want a standardized way of mapping exceptions thrown by service methods to the correct response codes.  While an `ArgumentException` usually means that there was a problem with user input, it can also be thrown for other reasons, which should still be identified as a 500 error.

My solution to this is to have a special set of `ApiException` classes, and a middleware which catches them.

### File Structure

![image.png](/.attachments/image-eec8f5b7-90dd-456f-ad69-21a9c9c588cf.png)

### Response Model

_Models\/API\/ApiException.cs_
```cs
namespace React_Demo.Models.API
{
	public class ApiException
	{
		public string Type { get; set; }
		public string Message { get; set; }
	}
}
```

### Exception Classes

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

_Util\/ExceptionMiddleware.cs_
```cs
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;
using Newtonsoft.Json.Serialization;
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

				Models.API.ApiException errObj = new Models.API.ApiException()
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

## Updating the Channel Manager Service

### Channel Search

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

_Services\/ChannelManagerService.cs_
```cs
public async Task DeleteChannel(Channel channel)
{
	long channelId = channel.ChannelId;

	//the channel provided as an argument is simply a deserialized object from the user
	//we need to obtain the tracked entity from EF Core
	Channel oldChannel = await context.Channel.FindAsync(channelId);

	if (oldChannel == null)
	{
		throw new NotFoundException("No channel exists with the id: " + channelId);
	}

	context.Channel.Remove(oldChannel);

	await context.SaveChangesAsync();
}
```

## Updating the Channel Manager Controller

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

		[HttpGet("byId")]
		public async Task<IActionResult> GetChannel([FromQuery] long channelId)
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

		[HttpDelete]
		public async Task<IActionResult> DeleteChannel([FromBody] Channel channel)
		{
			await channelService.DeleteChannel(channel);
			return Ok();
		}
	}
}
```

## Try it out

### Empty name

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

![image.png](/.attachments/image-8e4b0efb-b860-4d9e-9acb-7fb1226f7302.png)

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

![image.png](/.attachments/image-69d3db4b-2e56-40e7-9764-0ef0def2c01b.png)

## Download Source

The source code up to this point can be found here:

https://github.com/mjhoffm2/react-demo/tree/Part-8

The code for this part of the tutorial can be found in the `.net core 2.1` folder.