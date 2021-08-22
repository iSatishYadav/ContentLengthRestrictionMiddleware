# Defence In-Depth: Designing an HTTP Content Length Restriction Middleware - ASP.NET 5 (or .NET Core)
## The What?

We want to design a Middleware that - when plugged into an ASP.NET 5 (or .NET Core) application pipeline - restricts the input payload size, so attacks which rely of sending a bigger sized (or larger than our specified size) payload can be rejected by Application itself.

## The Why?
You'd argue that _I have a firewall and other Security and Network devices so why do I need to implement this at Application level?_

Well, if you follow Defence-in-Depth, [US-CERT (United States Computer Emergency Readiness Team)](https://us-cert.cisa.gov/bsi/articles/knowledge/principles/defense-in-depth) has this to say about redundant security mechanisms:

> Layering security defenses in an application can reduce the chance of a successful attack. Incorporating redundant security mechanisms requires an attacker to circumvent each mechanism to gain access to a digital asset. For example, a software system with authentication checks may prevent an attacker that has subverted a firewall. Defending an application with multiple layers can prevent a single point of failure that compromises the security of the application.

## The How? - Let's code
So I broken down this designing to 3 parts:
1. Create the basic logic
2. Make it a re-useable Middleware
3. Make it look like a Native Middleware

So let's get into it:


### 1. Create the basic logic

We want to check the `ContentLength` of `Request` payload against our limit, if it exceeds, it should send the `HTTP 413 Entity Too Large` as per [IETF RFC 7231](https://tools.ietf.org/html/rfc7231#section-6.5.11) specification of HTTP.

````csharp
if (httpContext.Request.ContentLength > SOME_LIMIT)
{    
    httpContext.Response.StatusCode = StatusCodes.Status413RequestEntityTooLarge;
    await httpContext.Response.WriteAsJsonAsync(new
    {
        Title = "Request too large",
        Status = StatusCodes.Status413RequestEntityTooLarge,
        Type = "https://tools.ietf.org/html/rfc7231#section-6.5.11",
    });
    await httpContext.Response.CompleteAsync();
}
else
{
    await _requestDelegate.Invoke(httpContext);
}
````
Let's break it down:

* First we're checking the content length.
* If it's greater than our limit, we're writing a `JSON` response with `HTTP 413`. And completing the response as we don't need to execute any further Middleware.
* If not, we can continue the Middleware pipe and execute next Middleware.

### 2. Create a re-usable middleware
Now we have our basic logic ready, let's create a re-usable middleware out of it.
We need to do following:
1. Create a Middleware class and run our logic in `Invoke` or `InvokeAsync` method.
2. Take input at the runtime instead of hard-coding it.
3. Add logging

Let's complete this one by one:

#### 2.1.  Create a Middleware class and run our logic in `Invoke` or `InvokeAsync` method.
Here's a basic structure of a Middleware class looks like. It should have an `Invoke` or `InvokeAsync` method which would be called by runtime based on our configuration. 

````csharp
public class ContentLengthRestrictionMiddleware
{
    private readonly RequestDelegate _requestDelegate;

    public ContentLengthRestrictionMiddleware(RequestDelegate nextRequestDelegate)
    {
        _requestDelegate = nextRequestDelegate;        
    }
    public async Task InvokeAsync(HttpContext httpContext)
    {
        if (_contentLengthRestrictionOptions != null && _contentLengthRestrictionOptions.ContentLengthLimit > 0 && httpContext.Request.ContentLength > _contentLengthRestrictionOptions.ContentLengthLimit)
        {
            _logger.LogWarning("Rejecting request with Content-Length {0} more than allowed {1}.", httpContext.Request.ContentLength, _contentLengthRestrictionOptions.ContentLengthLimit);
            httpContext.Response.StatusCode = StatusCodes.Status413RequestEntityTooLarge;
            await httpContext.Response.WriteAsJsonAsync(new
            {
                Title = "Request too large",
                Status = StatusCodes.Status413RequestEntityTooLarge,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.11",
            });
            await httpContext.Response.CompleteAsync();
        }
        else
        {
            await _requestDelegate.Invoke(httpContext);
        }
    }
}                   
````
#### 2.2. Take input at the runtime instead of hard-coding it.
Currently we're hard-coding the limit (`SOME_LIMIT`), we can create a class and take input at runtime.
````csharp
public class ContentLengthRestrictionOptions
{
    public long ContentLengthLimit { get; set; }
}
````
And modify our Middleware to use this:

````csharp
public class ContentLengthRestrictionMiddleware
{
    private readonly ContentLengthRestrictionOptions _contentLengthRestrictionOptions;
    private readonly RequestDelegate _requestDelegate;

    public ContentLengthRestrictionMiddleware(RequestDelegate nextRequestDelegate, ContentLengthRestrictionOptions contentLengthRestrictionOptions)
    {
        _requestDelegate = nextRequestDelegate;
        _contentLengthRestrictionOptions = contentLengthRestrictionOptions;        
    }
    public async Task InvokeAsync(HttpContext httpContext)
    {
        if (_contentLengthRestrictionOptions != null && _contentLengthRestrictionOptions.ContentLengthLimit > 0 && httpContext.Request.ContentLength > _contentLengthRestrictionOptions.ContentLengthLimit)
        {
            _logger.LogWarning("Rejecting request with Content-Length {0} more than allowed {1}.", httpContext.Request.ContentLength, _contentLengthRestrictionOptions.ContentLengthLimit);
            httpContext.Response.StatusCode = StatusCodes.Status413RequestEntityTooLarge;
            await httpContext.Response.WriteAsJsonAsync(new
            {
                Title = "Request too large",
                Status = StatusCodes.Status413RequestEntityTooLarge,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.11",
            });
            await httpContext.Response.CompleteAsync();
        }
        else
        {
            await _requestDelegate.Invoke(httpContext);
        }
    }
}
````

#### 2.3. Add logging

Here's little tricky part, you can't just inject an `ILogger<T>` and expect runtime to give you that dependency, you can get an `ILoggerFactory` and then you can create your own `ILogger<T>`.

````csharp
public class ContentLengthRestrictionMiddleware
{
    private readonly ContentLengthRestrictionOptions _contentLengthRestrictionOptions;
    private readonly ILogger<ContentLengthRestrictionMiddleware> _logger;
    private readonly RequestDelegate _requestDelegate;

    public ContentLengthRestrictionMiddleware(RequestDelegate nextRequestDelegate, ContentLengthRestrictionOptions contentLengthRestrictionOptions, ILoggerFactory loggerFactory)
    {
        _requestDelegate = nextRequestDelegate;
        _contentLengthRestrictionOptions = contentLengthRestrictionOptions;
        _logger = loggerFactory.CreateLogger<ContentLengthRestrictionMiddleware>();
    }
    public async Task InvokeAsync(HttpContext httpContext)
    {
        if (_contentLengthRestrictionOptions != null && _contentLengthRestrictionOptions.ContentLengthLimit > 0 && httpContext.Request.ContentLength > _contentLengthRestrictionOptions.ContentLengthLimit)
        {
            _logger.LogWarning("Rejecting request with Content-Length {0} more than allowed {1}.", httpContext.Request.ContentLength, _contentLengthRestrictionOptions.ContentLengthLimit);
            httpContext.Response.StatusCode = StatusCodes.Status413RequestEntityTooLarge;
            await httpContext.Response.WriteAsJsonAsync(new
            {
                Title = "Request too large",
                Status = StatusCodes.Status413RequestEntityTooLarge,
                Type = "https://tools.ietf.org/html/rfc7231#section-6.5.11",
            });
            await httpContext.Response.CompleteAsync();
        }
        else
        {
            await _requestDelegate.Invoke(httpContext);
        }
    }
}
````
I've added a log only when the Middleware rejects the requests.

### 3. Make it look like a native Middleware.
We can create an Extension method on `IApplicationBuilder`, something to call like `UseXXX`, so it feels like a Native Middleware.

````csharp
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseContentLengthRestriction(this IApplicationBuilder builder, ContentLengthRestrictionOptions contentLengthRestrictionOptions)
        => builder.UseMiddleware<ContentLengthRestrictionMiddleware>(contentLengthRestrictionOptions);
}
````
And we can use it like:

````csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    //This is our Middleware 😍
    app.UseContentLengthRestriction(new ContentLengthRestrictionOptions
    {
        ContentLengthLimit = 10
    });

    /// Other configuration
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }

    app.UseHttpsRedirection();

    app.UseRouting();

    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
````

## 4. The Execution
Sending a `POST` request to our endpoint:
````curl
curl -X POST "https://localhost:5001/WeatherForecast" -H  "accept: */*" -H  "Content-Type: application/json" -d "{\"date\":\"2021-08-22T14:17:58.115Z\",\"temperatureC\":0,\"summary\":\"string\"}"
````

Results in following:
````json
{
  "title": "Request too large",
  "status": 413,
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.11"
}
````

And logs following in to stdout:
````
warn: AwesomeApi.ContentLengthRestrictionMiddleware[0]
      Rejecting request with Content-Length 71 more than allowed 10.
````

## Conclusion
That's it! Congratulations! You just designed and created a custom Middleware! Give yourself a pat in the back. Don't forget to stretch your shoulders and neck once in a while.

Happy Coding!