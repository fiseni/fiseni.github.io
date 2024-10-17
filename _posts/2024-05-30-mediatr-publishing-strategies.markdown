---
# layout: post
# author: Fati Iseni
title: "Extending MediatR with publishing strategies"
date: 2024-05-30 12:00:00 +0100
description: Explore advanced publishing strategies with the MediatR library in .NET applications. This article describes how to extend the MediatR library to support custom strategies while publishing notifications. Learn how to implement, register, and use these strategies with ease in your .NET projects.
categories: [Tutorial, Software Development]
tags: [Mediator]
pin: false
#math: false
#toc: false
image:
  path: /assets/img/posts/777456/cover.png
---
I published an article on how to utilize different publishing strategies for MediatR notifications. Please check it [here](https://fiseni.com/posts/publishing-strategies-in-MediatR/) for more context. However, the authors introduced breaking changes in the API and the internals, so the approaches described in that article won't work for versions 12 and onward.

Starting with version 12, we can provide a custom `INotificationPublisher` implementation during the registration.

```csharp
services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssemblyContaining<Program>();
    cfg.NotificationPublisher = new MyCustomPublisher(); // this will be singleton
    cfg.NotificationPublisherType = typeof(MyCustomPublisher); // this will be the ServiceLifetime
});
```

This new capability offers some flexibility, but it's still not exactly what we want. Depending on various circumstances, we may want to publish different notifications using different strategies or even different strategies for the same notification. So, let’s define our requirements.

- Implement multiple publishing strategies
- We should be able to choose a strategy while publishing a notification.
- Ideally, the feature should be an extension to `IPublisher`. Consumers should not have to deal with new types.

## Publishing Strategies
Let’s first create an enum with the strategies we plan on supporting.

```csharp
public enum PublishStrategy
{
    /// <summary>
    /// The default publisher or the one set in MediatR configuration.
    /// </summary>
    Default = 0,

    /// <summary>
    /// Executes and awaits each notification handler after one another.
    /// Returns when all handlers complete or an exception has been thrown.
    /// In case of an exception, the rest of the handlers are not executed.
    /// </summary>
    Sequential = 1,

    /// <summary>
    /// Executes and awaits each notification handler after one another.
    /// Returns when all handlers complete. It continues on exception(s).
    /// In case of any exception(s), they will be captured in an AggregateException.
    /// </summary>
    SequentialAll = 2,
}
```

## Extending Mediator
Next, let's define the `INotificationPublisher` implementations. You may notice that `SequentialPublisher` is the same as the built-in one. I picked them just as an example to demonstrate how to hook them up. You'll define your desired custom publishers.

```csharp
public class SequentialPublisher : INotificationPublisher
{
    public async Task Publish(
        IEnumerable<NotificationHandlerExecutor> handlerExecutors,
        INotification notification,
        CancellationToken cancellationToken)
    {
        foreach (var handler in handlerExecutors)
        {
            await handler.HandlerCallback(notification, cancellationToken).ConfigureAwait(false);
        }
    }
}

public class SequentialAllPublisher : INotificationPublisher
{
    public async Task Publish(
        IEnumerable<NotificationHandlerExecutor> handlerExecutors,
        INotification notification,
        CancellationToken cancellationToken)
    {
        var exceptions = new List<Exception>();

        foreach (var handlerExecutor in handlerExecutors)
        {
            try
            {
                await handlerExecutor.HandlerCallback(notification, cancellationToken).ConfigureAwait(false);
            }
            catch (AggregateException ex)
            {
                exceptions.AddRange(ex.Flatten().InnerExceptions);
            }
            catch (Exception ex) when (ex is not (OutOfMemoryException or StackOverflowException))
            {
                exceptions.Add(ex);
            }
        }

        if (exceptions.Count != 0)
        {
            throw new AggregateException(exceptions);
        }
    }
}
```

The Mediator implementation expects this publisher implementation to be provided in the constructor, along with the `IServiceProvider`. So, we can not just do this, and call it a day.

```csharp
public static Task Publish<TNotification>(
    this IPublisher publisher,
    TNotification notification,
    PublishStrategy strategy,
    CancellationToken cancellationToken = default)
    where TNotification : INotification
{
    INotificationPublisher? notificationPublisher = strategy switch
    {
        PublishStrategy.Sequential => new SequentialPublisher(),
        PublishStrategy.SequentialAll => new SequentialAllPublisher(),
        _ => null
    };

    return notificationPublisher is null
        ? publisher.Publish(notification, cancellationToken)
        // We need to provide the IServiceProvider
        : new Mediator(.., notificationPublisher).Publish(notification, cancellationToken);
}
```

Instead, we must create our custom mediator implementation, register it during registration, and define the necessary extensions. The `ExtendedMediator` inherits `Mediator` and defines an additional `Publish` method that accepts the strategy as a parameter. Then, we create a new instance of Mediator and pass the chosen publisher. The Mediator implementation is a very lightweight object (contains only two references as a state), so creating a new instance is an acceptable approach. But, what if the consumer has defined a different lifetime (e.g. scoped) during configuration? That's not an issue since we're passing the resolved `IServiceProvider`. In the case of a scoped lifetime, the service provider itself is scoped and the behavior will be preserved.

```csharp
public class ExtendedMediator(IServiceProvider serviceProvider) : Mediator(serviceProvider)
{
    private readonly IServiceProvider _serviceProvider = serviceProvider;

    private static readonly Dictionary<PublishStrategy, INotificationPublisher> _publishers = new()
    {
        [PublishStrategy.Sequential] = new SequentialPublisher(),
        [PublishStrategy.SequentialAll] = new SequentialAllPublisher(),
    };

    public Task Publish<TNotification>(
        TNotification notification,
        PublishStrategy strategy,
        CancellationToken cancellationToken = default)
        where TNotification : INotification
    {
        if (_publishers.TryGetValue(strategy, out var publisher))
        {
            new Mediator(_serviceProvider, publisher).Publish(notification, cancellationToken);
        }

        return Publish(notification, cancellationToken);
    }
}

public static class MediatorExtensions
{
    public static Task Publish<TNotification>(
        this IPublisher publisher,
        TNotification notification,
        PublishStrategy strategy,
        CancellationToken cancellationToken = default)
        where TNotification : INotification
    {
        return publisher is ExtendedMediator extendedMediator
            ? extendedMediator.Publish(notification, strategy, cancellationToken)
            : throw new NotSupportedException("The extended mediator implementation is not registered! Register it with the IServiceCollection.AddExtendedMediatR extensions.");
    }

    public static IServiceCollection AddExtendedMediatR(
        this IServiceCollection services,
        Action<MediatRServiceConfiguration> configuration)
    {
        var serviceConfig = new MediatRServiceConfiguration();
        configuration.Invoke(serviceConfig);

        return services.AddExtendedMediatR(serviceConfig);
    }

    public static IServiceCollection AddExtendedMediatR(
        this IServiceCollection services,
        MediatRServiceConfiguration configuration)
    {
        configuration.MediatorImplementationType = typeof(ExtendedMediator);
        services.AddMediatR(configuration);

        return services;
    }
}
```

## Usage

Now that we’re all set, the usage is quite straightforward.

```csharp
builder.Services.AddExtendedMediatR(cfg =>
{
    // All your desired configuration.
    cfg.RegisterServicesFromAssemblyContaining<Program>();
	
    // Our extension will always set the MediatorImplementationType to ExtendedMediator.
});
```

```csharp
public class Foo(IPublisher publisher)
{
    public async Task Run(CancellationToken cancellationToken)
    {
        // The built-in behavior
        await publisher.Publish(new Ping(), cancellationToken);

        // Publish with specific strategy
        await publisher.Publish(new Ping(), PublishStrategy.SequentialAll, cancellationToken);
    }
}
```

## NoWait strategies
There might be cases where you want to publish a notification and not wait for the handler's completion. Simply, you want them to run as background tasks. I'd avoid this approach and use these strategies sparingly, however, you might have a valid use case. To implement this feature, we might be compelled to just add a new publisher as follows.

```csharp
public enum PublishStrategy
{
    Default = 0,
    Sequential = 1,
    SequentialAll = 2,
    SequentialNoWait = 3,
    SequentialAllNoWait = 4,
}
```

```csharp
public class SequentialNoWaitPublisher : INotificationPublisher
{
    public Task Publish(
        IEnumerable<NotificationHandlerExecutor> handlerExecutors,
        INotification notification,
        CancellationToken cancellationToken)
    {
        _ = Task.Run(async () =>
        {
            try
            {
                foreach (var handler in handlerExecutors)
                {
                    await handler.HandlerCallback(notification, cancellationToken).ConfigureAwait(false);
                }
            }
            catch (Exception ex)
            {
                // Handle as needed.
            }
        }, cancellationToken);
        return Task.CompletedTask;
    }
}
```

However, this is a bad idea. This implementation won't behave correctly; for not so obvious reasons. If the consumer has defined a scoped lifetime, once the request is completed, the scope and all resolved dependencies out of that scope will be disposed. The background task we've defined may outlive the request, and our handlers will end up with disposed dependencies.

We must run the background task in a separate scope. Moreover, we can't create a scope out of the injected `IServiceProvider` since the provider itself is scoped. Instead, we need a separate scope out of the root provider. That said, the updated implementation would be as follows.

```csharp
internal class ExtendedMediator(
    IServiceScopeFactory serviceScopeFactory,
    IServiceProvider serviceProvider)
    : Mediator(serviceProvider)
{
    private readonly IServiceScopeFactory _serviceScopeFactory = serviceScopeFactory;
    private readonly IServiceProvider _serviceProvider = serviceProvider;

    private static readonly Dictionary<PublishStrategy, (INotificationPublisher Publisher, bool NoWaitMode)> _publishers = new()
    {
        [PublishStrategy.Sequential] = (new SequentialPublisher(), false),
        [PublishStrategy.SequentialNoWait] = (new SequentialPublisher(), true),
        [PublishStrategy.SequentialAll] = (new SequentialAllPublisher(), false),
        [PublishStrategy.SequentialAllNoWait] = (new SequentialAllPublisher(), true),
    };

    public Task Publish<TNotification>(
        TNotification notification,
        PublishStrategy strategy,
        CancellationToken cancellationToken = default)
        where TNotification : INotification
    {
        if (_publishers.TryGetValue(strategy, out (INotificationPublisher Publisher, bool NoWaitMode) item))
        {
            return item.NoWaitMode
                ? PublishNoWait(_serviceScopeFactory, notification, item.Publisher, cancellationToken)
                : Publish(_serviceProvider, notification, item.Publisher, cancellationToken);
        }

		// Fall back to the default behavior
        return Publish(notification, cancellationToken);
    }

    private static Task Publish<TNotification>(
        IServiceProvider serviceProvider,
        TNotification notification,
        INotificationPublisher publisher,
        CancellationToken cancellationToken) where TNotification : INotification
        => new Mediator(serviceProvider, publisher).Publish(notification, cancellationToken);

    private static Task PublishNoWait<TNotification>(
        IServiceScopeFactory serviceScopeFactory,
        TNotification notification,
        INotificationPublisher publisher,
        CancellationToken cancellationToken) where TNotification : INotification
    {
        _ = Task.Run(async () =>
        {
            using var scope = serviceScopeFactory.CreateScope();
            var logger = scope.ServiceProvider.GetService<ILogger<ExtendedMediator>>();

            try
            {
                var mediator = new Mediator(scope.ServiceProvider, publisher);
                await mediator.Publish(notification, cancellationToken).ConfigureAwait(false);
            }
            catch (Exception ex)
            {
                // The aggregate exceptions are already flattened by the publishers.
                logger?.LogError(ex, "Error occurred while executing the handler in NoWait mode!");
            }

        }, cancellationToken);

        return Task.CompletedTask;
    }
}
```

This implementation also offers the ability to wrap any publisher in a background task. So, instead of having separate NoWait enum items, you may define an additional `runAsBackgroundTask` parameter for the `Publish` method. That's up to you to decide which API is more convenient for you.

I hope you found this article useful. Happy coding!