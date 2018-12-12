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

### Getting a Single Channel

### Adding a Channel

### Updating a Channel

### Deleting a Channel

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

		[HttpGet]
		public async Task<IActionResult> GetChannel([FromQuery] long channelId)
		{
			var channel = await channelService.GetChannel(channelId);
			return Json(channel);
		}

		[HttpPost]
		public async Task<IActionResult> AddChannel([FromBody] Channel channel)
		{
			await channelService.AddChannel(channel);
			return Ok();
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

## Download Source
