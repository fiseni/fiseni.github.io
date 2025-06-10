---
# layout: post
# author: Fati Iseni
title: "Testing .NET Worker Services with WebApplicationFactory"
date: 2024-10-21 12:00:00 +0100
last_modified_at: 2024-10-21 12:00:00 +0100
description: How to use WebApplicationFactory for .NET worker services.
categories: [Tech, Software Development]
tags: [dotnet]
pin: false
#math: false
#toc: false
image:
  path: /assets/img/posts/349857/cover.png
---

The .NET Worker Services are ideal for background processing tasks. They use a .NET Generic Host, and you can easily configure dependency injection, logging, and configuration. However, setting up tests for them can be a bit tricky. The `WebApplicationFactory<TEntryPoint>` class, commonly used for ASP.NET Core integration tests, does not work with the generic host out of the box.

This article demonstrates how to set up a Worker Service, explains the problem encountered when using `WebApplicationFactory`, and shows the solution.

## Worker Service Example

Below is a simple Worker Service that registers a background worker and a scoped service:

```csharp
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddScoped<IFooService, FooService>();
builder.Services.AddHostedService<Worker>();

var host = builder.Build();
host.Run();


public class Worker(IServiceScopeFactory scopeFactory) : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory = scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var fooService = scope.ServiceProvider.GetRequiredService<IFooService>();
            fooService.Calculate();

            await Task.Delay(1000, stoppingToken);
        }
    }
}

public interface IFooService
{
    int Calculate();
}

public class FooService : IFooService
{
    public int Calculate()
    {
        return 11;
    }
}
```

## The Testing Challenge

When you try to use `WebApplicationFactory` for a Worker Service, you may encounter this exception:

```
System.InvalidOperationException: No application configured. Please specify an application via IWebHostBuilder.UseStartup, IWebHostBuilder.Configure, or specifying the startup assembly via StartupAssemblyKey in the web host configuration.
```

This happens because Worker Services do not configure an HTTP request pipeline, which `WebApplicationFactory` expects by default.

## The Solution: No-op Configure

To resolve this, add a no-op configuration in your test factory:

```csharp
public class TestFactory : WebApplicationFactory<WorkerServiceMarker>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        // This line is required to satisfy the host builder
        builder.Configure(_ => { });

        builder.ConfigureServices(services =>
        {
            // Optionally remove hosted services to prevent them from running during tests
            var hostedServiceDescriptors = services
                .Where(descriptor => descriptor.ServiceType == typeof(IHostedService))
                .ToList();

            foreach (var descriptor in hostedServiceDescriptors)
            {
                services.Remove(descriptor);
            }
        });
    }
}
```

The key line is `builder.Configure(_ => { });`. This provides a minimal, no-op application configuration, allowing the host to build successfully.

## Summary

- Worker Services can be tested with `WebApplicationFactory` by adding a no-op `Configure` call.
- This enables you to use the full host, configuration, and DI setup in your tests.
- Optionally, you can remove hosted services during tests to prevent background tasks from running.

I hope you found this article useful. Happy coding!