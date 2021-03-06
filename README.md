# Serilog.AspNetCore [![Build status](https://ci.appveyor.com/api/projects/status/4rscdto23ik6vm2r?svg=true)](https://ci.appveyor.com/project/serilog/serilog-aspnetcore) [![NuGet Version](http://img.shields.io/nuget/v/Serilog.AspNetCore.svg?style=flat)](https://www.nuget.org/packages/Serilog.AspNetCore/) 


Serilog logging for ASP.NET Core. This package routes ASP.NET Core log messages through Serilog, so you can get information about ASP.NET's internal operations logged to the same Serilog sinks as your application events.

### Instructions

**First**, install the _Serilog.AspNetCore_ [NuGet package](https://www.nuget.org/packages/Serilog.AspNetCore) into your app. You will need a way to view the log messages - _Serilog.Sinks.Console_ writes these to the console; there are [many more sinks available](https://www.nuget.org/packages?q=Tags%3A%22serilog%22) on NuGet.

```powershell
Install-Package Serilog.AspNetCore -DependencyVersion Highest
Install-Package Serilog.Sinks.Console
```

**Next**, in your application's _Program.cs_ file, configure Serilog first:

```csharp
public class Program
{
    public static int Main(string[] args)
    {
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Debug()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .CreateLogger();
```

Then, add `UseSerilog()` to the web host builder. A `try`/`catch` block will ensure any configuration issues are appropriately logged:

```csharp
        try
        {
            Log.Information("Starting web host");

            var host = new WebHostBuilder()
                .UseKestrel()
                .UseContentRoot(Directory.GetCurrentDirectory())
                .UseIISIntegration()
                .UseStartup<Startup>()
                .UseSerilog() // <-- Add this line
                .Build();

            host.Run();

            return 0;
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "Host terminated unexpectedly");
            return 1;
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }
}
```

**Finally**, clean up by removing the remaining configuration for the default logger:

 * Remove calls to `AddLogging()`
 * Remove the `"Logging"` section from _appsettings.json_ files (this can be replaced with [Serilog configuration](https://github.com/serilog/serilog-settings-configuration) as shown in [this example](https://github.com/serilog/serilog-aspnetcore/blob/dev/samples/SimpleWebSample/Program.cs), if required)
 * Remove `ILoggerFactory` parameters and any `Add*()` calls on the logger factory in _Startup.cs_
 * Remove `UseApplicationInsights()` (this can be replaced with the [Serilog AI sink](https://github.com/serilog/serilog-sinks-applicationinsights), if required)

That's it! With the level bumped up a little you will see log output like:

```
[22:14:44.646 DBG] RouteCollection.RouteAsync
	Routes: 
		Microsoft.AspNet.Mvc.Routing.AttributeRoute
		{controller=Home}/{action=Index}/{id?}
	Handled? True
[22:14:44.647 DBG] RouterMiddleware.Invoke
	Handled? True
[22:14:45.706 DBG] /lib/jquery/jquery.js not modified
[22:14:45.706 DBG] /css/site.css not modified
[22:14:45.741 DBG] Handled. Status code: 304 File: /css/site.css
```

Tip: to see Serilog output in the Visual Studio output window when running under IIS, select _ASP.NET Core Web Server_ from the _Show output from_ drop-down list.

### Using the package

With _Serilog.AspNetCore_ installed and configured, you can write log messages directly through Serilog or any `ILogger` interface injected by ASP.NET. All loggers will use the same underlying implementation, levels, and destinations.

**Tip:** change the minimum level for `Microsoft` to `Warning` and plug in this [custom logging middleware](https://github.com/datalust/serilog-middleware-example/blob/master/src/Datalust.SerilogMiddlewareExample/Diagnostics/SerilogMiddleware.cs) to clean up request logging output and record more context around errors and exceptions.

### Alternative configuration

You can chose to build the logger as part of the `WebHostBuilder` pipeline, and thus benefit from the application configuration.
The following code shows an example of such a configuration:

````csharp
public class Program
{
    public static void Main(string[] args)
    {
        var host = new WebHostBuilder()
            .UseKestrel()
            .UseContentRoot(Directory.GetCurrentDirectory())
            // Load the application configuration over the web host configuration.
            .ConfigureAppConfiguration((hostingContext, configurationBuilder) =>
            {
                configurationBuilder
                    .SetBasePath(hostingContext.HostingEnvironment.ContentRootPath)
                    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                    .AddJsonFile($"appsettings.{hostingContext.HostingEnvironment.EnvironmentName}.json", optional: true, reloadOnChange: true)
                    .AddEnvironmentVariables();
            })
            // Configure Serilog to be used as the logger for the whole application.
            .UseSerilog((hostingContext, loggerConfiguration) =>
                loggerConfiguration.ReadFrom.Configuration(hostingContext.Configuration)
                    .Enrich.FromLogContext()
                    .WriteTo.Console()
            )
            .UseIISIntegration()
            .UseStartup<Startup>()
            .Build();

        host.Run();
    }
}
````

With this code, the default behavior is to set the created `ILogger` as the default logger. `Log.Logger` can be used as usual to access the created logger.
