---
# layout: post
# author: Fati Iseni
title: "Publishing strategies in MediatR"
date: 2022-12-17 12:00:00 +0100
description: Explore advanced publishing strategies with the MediatR library in .NET applications. This article dives into the Mediator pattern, its benefits, and how to extend the MediatR library to support custom publishing strategies for handling notifications in various scenarios. Learn how to implement, register, and use these strategies with ease in your .NET projects.
categories: [Blogging, Tutorial, Software Development]
tags: [Mediator]
image: /assets/img/pozitron-cover.png
pin: false
# math: true
# toc: true
---
<strong> EDIT: There are breaking changes in MediatR 12.0.1, and the following implementation won't work. I'll publish soon an updated implementation.</strong>

The Mediator pattern is a behavioral design pattern that promotes loose coupling between objects by having a central point of communication, which is called the mediator. This pattern is particularly useful when you have a complex system with multiple interacting components that need to communicate with each other. By introducing a mediator object, you can reduce the dependencies between these components, making it easier to maintain and evolve the system over time. In the Mediator pattern, components don't interact with each other directly; instead, they send messages to the mediator, which then coordinates the communication between the components. This way, components can be added, removed, or modified without affecting other parts of the system. The mediator's primary responsibility is to facilitate the interaction between the components and to ensure that they collaborate correctly.

One of the popular and widely used implementation in .NET is the MediatR library. It has support for various scenarios, including request/response, commands, queries, and notifications. In this article we'll focus on notifications and how we can utilize different publishing strategies. The library by default calls and await each handler sequentially. In case of an exception, the execution is stopped, meaning the rest of the handlers won't execute. We'd like to extend this behavior, and use a different strategy on demand. First, let's define our requirements.
- Implement multiple publishing strategies
- We should be able to choose a strategy while publishing a notification.
- Ideally, the feature should be an extension to `IMediator`. Consumers should not have to deal with new types.

## Publishing Strategies

Let's first create an enum with the strategies we plan on supporting.

```csharp
public enum PublishStrategy
{
    /// <summary>
    /// Run each notification handler after one another. Returns when all handlers are finished or an exception has been thrown. In case of an exception, any handlers after that will not be run.
    /// </summary>
    AsyncSequentialStopOnException = 0,

    /// <summary>
    /// Run each notification handler after one another. Returns when all handlers are finished. In case of any exception(s), they will be captured in an AggregateException.
    /// </summary>
    AsyncSequentialContinueOnException = 1,

    /// <summary>
    /// Run all notification handlers asynchronously. Returns when all handlers are finished. In case of any exception(s), they will be captured in an AggregateException.
    /// </summary>
    AsyncWhenAll = 2,

    /// <summary>
    /// Run each notification handler on its own thread using Task.Run(). Returns when all threads (handlers) are finished. In case of any exception(s), if the call to Publish is awaited, they are captured in an AggregateException by Task.WhenAll. Do not use this strategy if you're accessing the database in your handlers, DbContext is not thread-safe.
    /// </summary>
    ParallelWhenAll = 3,

    /// <summary>
    /// Create a single new thread using Task.Run(), and run all notifications sequentially (continue on exception). Returns immediately and does not wait for any handlers to finish. Note that you cannot capture any exceptions, even if you await the call to Publish. To improve the traceability the exception is being captured internally and logged with ILogger if available.
    /// </summary>
    AsyncNoWait = 4,

    /// <summary>
    /// Run each notification handler on its own thread using Task.Run(). Returns immediately and does not wait for any handlers to finish. Note that you cannot capture any exceptions, even if you await the call to Publish. To improve the traceability the exception is being captured internally and logged with ILogger if available. Do not use this strategy if you're accessing the database in your handlers, DbContext is not thread-safe.
    /// </summary>
    ParallelNoWait = 5,
}
```

## Extending Mediator Implementation

Next, let's define a custom mediator implementation

```csharp
public class ExtendedMediator : Mediator
{
    private readonly ServiceFactory _serviceFactory;
    private readonly IServiceScopeFactory _serviceScopeFactory;
    private readonly Func<IEnumerable<Func<INotification, CancellationToken, Task>>, INotification, CancellationToken, Task> _publish;

    private ExtendedMediator(
        ServiceFactory serviceFactory,
        Func<IEnumerable<Func<INotification, CancellationToken, Task>>, INotification, CancellationToken, Task> publish)
        : base(serviceFactory)
    {
        _serviceFactory = serviceFactory;
        _serviceScopeFactory = default!;
        _publish = publish;
    }

    public ExtendedMediator(ServiceFactory serviceFactory, IServiceScopeFactory serviceScopeFactory)
        : base(serviceFactory)
    {
        _serviceFactory = serviceFactory;
        _serviceScopeFactory = serviceScopeFactory;
        _publish = base.PublishCore;
    }

    protected override Task PublishCore(
        IEnumerable<Func<INotification, CancellationToken, Task>> allHandlers,
        INotification notification,
        CancellationToken cancellationToken)
    {
        return _publish(allHandlers, notification, cancellationToken);
    }

    public Task Publish<TNotification>(
        TNotification notification,
        PublishStrategy strategy,
        CancellationToken cancellationToken) where TNotification : INotification
    {
        return strategy switch
        {
            PublishStrategy.AsyncNoWait => PublishNoWait(notification, AsyncSequentialContinueOnException, cancellationToken),
            PublishStrategy.ParallelNoWait => PublishNoWait(notification, ParallelWhenAll, cancellationToken),
            PublishStrategy.AsyncSequentialContinueOnException => new ExtendedMediator(_serviceFactory, AsyncSequentialContinueOnException).Publish(notification, cancellationToken),
            PublishStrategy.AsyncSequentialStopOnException => new ExtendedMediator(_serviceFactory, AsyncSequentialStopOnException).Publish(notification, cancellationToken),
            PublishStrategy.AsyncWhenAll => new ExtendedMediator(_serviceFactory, AsyncWhenAll).Publish(notification, cancellationToken),
            PublishStrategy.ParallelWhenAll => new ExtendedMediator(_serviceFactory, ParallelWhenAll).Publish(notification, cancellationToken),
            _ => throw new ArgumentException($"Unknown strategy: {strategy}")
        };
    }

    private Task PublishNoWait(
        INotification notification,
        Func<IEnumerable<Func<INotification, CancellationToken, Task>>, INotification, CancellationToken, Task> publish,
        CancellationToken cancellationToken)
    {
        _ = Task.Run(async () =>
        {
            using var scope = _serviceScopeFactory.CreateScope();
            var logger = scope.ServiceProvider.GetService<ILogger<ExtendedMediator>>();
            try
            {
                var mediator = new ExtendedMediator(scope.ServiceProvider.GetRequiredService, publish);
                await mediator.Publish(notification, cancellationToken).ConfigureAwait(false);
            }
            catch (Exception ex)
            {
                if (logger is not null)
                {
                    logger.LogError(ex, "Error occurred while executing the handler in ParallelNoWait mode");
                }
            }
        }, cancellationToken);

        return Task.CompletedTask;
    }

    private Task ParallelWhenAll(
        IEnumerable<Func<INotification, CancellationToken, Task>> handlers,
        INotification notification,
        CancellationToken cancellationToken)
    {
        var tasks = new List<Task>();

        foreach (var handler in handlers)
        {
            tasks.Add(Task.Run(() => handler(notification, cancellationToken), cancellationToken));
        }

        return Task.WhenAll(tasks);
    }

    private async Task AsyncWhenAll(
        IEnumerable<Func<INotification, CancellationToken, Task>> handlers,
        INotification notification,
        CancellationToken cancellationToken)
    {
        var tasks = new List<Task>();
        var exceptions = new List<Exception>();

        foreach (var handler in handlers)
        {
            try
            {
                tasks.Add(handler(notification, cancellationToken));
            }
            catch (Exception ex) when (!(ex is OutOfMemoryException || ex is StackOverflowException))
            {
                exceptions.Add(ex);
            }
        }

        try
        {
            await Task.WhenAll(tasks).ConfigureAwait(false);
        }
        catch (AggregateException ex)
        {
            exceptions.AddRange(ex.Flatten().InnerExceptions);
        }
        catch (Exception ex) when (!(ex is OutOfMemoryException || ex is StackOverflowException))
        {
            exceptions.Add(ex);
        }

        if (exceptions.Any())
        {
            throw new AggregateException(exceptions);
        }
    }

    private async Task AsyncSequentialContinueOnException(
        IEnumerable<Func<INotification, CancellationToken, Task>> handlers,
        INotification notification,
        CancellationToken cancellationToken)
    {
        var exceptions = new List<Exception>();

        foreach (var handler in handlers)
        {
            try
            {
                await handler(notification, cancellationToken).ConfigureAwait(false);
            }
            catch (AggregateException ex)
            {
                exceptions.AddRange(ex.Flatten().InnerExceptions);
            }
            catch (Exception ex) when (!(ex is OutOfMemoryException || ex is StackOverflowException))
            {
                exceptions.Add(ex);
            }
        }

        if (exceptions.Any())
        {
            throw new AggregateException(exceptions);
        }
    }

    private async Task AsyncSequentialStopOnException(
        IEnumerable<Func<INotification, CancellationToken, Task>> handlers,
        INotification notification,
        CancellationToken cancellationToken)
    {
        foreach (var handler in handlers)
        {
            await handler(notification, cancellationToken).ConfigureAwait(false);
        }
    }
}
```

## Registration and Extensions

We'll define few extension methods to help the registration and create an overload for the `Publish` method.

```c#
public static class MediatorExtensions
{
    public static Task Publish<TNotification>(this IMediator mediator, TNotification notification, PublishStrategy strategy, CancellationToken cancellationToken)
        where TNotification : INotification
    {
        if (mediator is ExtendedMediator customMediator)
        {
            return customMediator.Publish(notification, strategy, cancellationToken);
        }

        throw new NotSupportedException("The custom mediator implementation is not registered!");
    }

    public static IServiceCollection AddExtendedMediatR(this IServiceCollection services, params Assembly[] assemblies)
    {
        services.AddMediatR(options => options.Using<ExtendedMediator>().AsScoped(), assemblies);
        return services;
    }

    public static IServiceCollection AddExtendedMediatR(this IServiceCollection services, params Type[] handlerAssemblyMarkerTypes)
    {
        services.AddMediatR(options => options.Using<ExtendedMediator>().AsScoped(), handlerAssemblyMarkerTypes);

        return services;
    }
}

```

## Usage

Now that we're all set, the usage is quite straightforward. Register the extended mediator.

```c#
builder.Services.AddExtendedMediatR(typeof(Program));
```

Use it using existing `IMediator` contract.
```c#
[ApiController]
public class DummyController : ControllerBase
{
    private readonly IMediator _mediator;

    public DummyController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpGet("/")]
    public async Task<ActionResult> Get(CancellationToken cancellationToken)
    {
        await _mediator.Publish(new Ping(), PublishStrategy.AsyncNoWait, cancellationToken);
        return Ok();
    }
}

```

I hope you found this article useful. Happy coding!