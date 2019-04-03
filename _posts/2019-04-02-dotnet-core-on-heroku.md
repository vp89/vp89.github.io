---
layout: post
title:  ".NET Core on Heroku"
date:   2019-04-02 00:00:00 -0000
categories: programming
---

On my current project I am using .NET Core. I initially threw it up on Heroku because it's free and quick to get going. As things stands now, I'm probably going to stay on Heroku for the main web and database hosting and drop-in a little AWS for some specific things.

Heroku has some excellent documentation and integration with the languages/runtimes they have chosen to fully support, unfortunately .NET Core is not currently one of them. I thought I would post some things I have learned while standing up my project on there.

### Build / deploy

There are some .NET Core buildpacks out there but I didn't trust that they would be forward compatible with newer versions, so I just wrote my own simple Dockerfile that does exactly what I need.

`Dockerfile`
```
FROM microsoft/dotnet:2.2-sdk AS build-environment
COPY . /app
WORKDIR /app
RUN dotnet publish -c Release -o /app/output

FROM microsoft/dotnet:2.2-aspnetcore-runtime
WORKDIR /app
COPY --from=build-environment /app/output .
CMD ASPNETCORE_URLS=http://*:$PORT && dotnet dojomanage.web.dll
```

You also need to put this `heroku.yml` file at the root of your repo so that it knows to use the Dockerfile for your web dyno.

```
build:
  docker:
    web: Dockerfile
```

Heroku has a full integration with GitHub, and they now have free private repos. So my code is hosted on there and anytime I push a commit to master it triggers a build and deploys the image. I'm sure I will get more fancy with it once I have users, but for now this works fine.

### Logging

There are many different ways to do logs, but IMO the best way is to just print to `stdout` and then pull them from the supervising process. Heroku has APIs to do this it calls [Log Drains](https://devcenter.heroku.com/articles/log-drains). There are 3rd party products which are already integrated with these APIs and you can set them up with 1-click from your Heroku dashboard. I settled on Timber.io because I liked their UI and free-tier offering the best.

I'm a big proponent of structured logs, but had only used log4net before this. I tried to get structured JSON logs out using log4net but it's not as customizable so I wasn't able to get certain specific things I wanted to work. I ended up settling on Serilog with some tweaks and I'm really pleased with it.

This is what my config looks like:

```
Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Warning()
                .WriteTo.Console(new CustomJsonFormatter())
                .Enrich.FromLogContext()
                .CreateLogger();
```

I ended up writing my own `ITextFormatter` because I wanted to strip away a lot of the extraneous fields and I wanted the `TraceIdentifier` to be logged as `request_id` to match the logs from the Heroku reverse proxy. I'm also doing some other stuff to basically get the app logs to match the format of the proxy logs so if I get an exception I can search and get the full trace end-to-end for the whole request.

```
public class CustomJsonFormatter : ITextFormatter
{
    private JsonValueFormatter _formatter;

    public CustomJsonFormatter()
    {
        _formatter = new JsonValueFormatter();
    }

    public void Format(LogEvent logEvent, TextWriter output)
    {
        output.Write("{");

        output.Write($"\"level\":\"{ConvertLogLevel(logEvent.Level)}\"");

        output.Write($",\"message\":");
        var message = logEvent.MessageTemplate.Render(logEvent.Properties);
        JsonValueFormatter.WriteQuotedJsonString(message, output);

        if (logEvent.Properties.ContainsKey("RequestId"))
        {
            output.Write(",");
            JsonValueFormatter.WriteQuotedJsonString("request_id", output);
            output.Write(":");
            _formatter.Format(logEvent.Properties["RequestId"], output);
        }

        if (logEvent.Exception != null)
        {
            output.Write(",");
            JsonValueFormatter.WriteQuotedJsonString("exception", output);
            output.Write(":");
            JsonValueFormatter.WriteQuotedJsonString(logEvent.Exception.ToString(), output);
        }

        var propertyExclusionList = new HashSet<string>()
        {
            "CorrelationId", "RequestId", "ConnectionId", "SourceContext", "ActionId", "ActionName", "RequestPath", "EventId"
        };

        foreach (var property in logEvent.Properties.Where(p => !propertyExclusionList.Contains(p.Key)))
        {
            output.Write(",");
            JsonValueFormatter.WriteQuotedJsonString(property.Key, output);
            output.Write(":");
            _formatter.Format(property.Value, output);
        }

        output.Write("}");
        output.WriteLine();
    }

    private string ConvertLogLevel(LogEventLevel level)
    {
        switch (level)
        {
            case LogEventLevel.Debug:
                return "debug";
            case LogEventLevel.Error:
                return "error";
            case LogEventLevel.Fatal:
                return "fatal";
            case LogEventLevel.Information:
                return "info";
            case LogEventLevel.Verbose:
                return "trace";
            case LogEventLevel.Warning:
                return "warn";
            default:
                return "";
        }
    }
}
```

I also have some middleware to enrich the logging context with some values like the current `userId`. This saves me from having to include it into each log line.

```
public class UserContextMiddleware
{
    private readonly RequestDelegate _next;

    public UserContextMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        if (int.TryParse(context.User.FindFirstValue("UserId"), out int userId))
        {
            LogContext.PushProperty("userId", userId);
        }

        await _next(context);
    }
}
```

### Headers

If you are unfamiliar, your app runs in a Docker container listening on a certain port and the actual requests on port 443 go through Heroku's reverse proxy to your app. Fortunately Heroku forwards some of the data about these requests in the headers so you can pass it through to your app and make it behave like it's actually listening to those real requests.

One thing I did was write a simple middleware to set the `x-request-id` as the TraceIdentifier. This allows me to log it and show it to my user when my app throws an exception. They can then report this request-id to me and I can easily look up my logs on Timber and find the full trace.

```
public class RequestScopingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestScopingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        // The Heroku router passes this along in x-request-id, otherwise use the one generated by Kestrel
        if (context.Request.Headers.TryGetValue("x-request-id", out StringValues xRequestId))
        {
            context.TraceIdentifier = xRequestId.ToString();
        }

        await _next(context);
    }
}
```

If you need to generate URLs anywhere in your app and you are using https, you will need to forward the scheme from the reverse proxy because your internal app will be running on http. If you don't do this then when you call `HttpContext.Request.Scheme` you will get `http` even though all the requests going through to your app are `https`.

[There is more written here about configuring .NET Core to work with reverse proxies.](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/proxy-load-balancer?view=aspnetcore-2.2)

`Startup.cs / ConfigureServices`
```
services.Configure<ForwardedHeadersOptions>(options =>
{
    // this forwards the protocol that heroku receives in the web request
    options.ForwardedHeaders = ForwardedHeaders.XForwardedProto;
});
```

I put this right at the start of my request pipeline, even before the exception handling.

`Startup.cs / Configure`
```
app.UseForwardedHeaders();
```
