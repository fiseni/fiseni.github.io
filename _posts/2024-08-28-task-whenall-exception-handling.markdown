---
# layout: post
# author: Fati Iseni
title: "Handling exceptions for Task.WhenAll"
date: 2024-08-28 12:00:00 +0100
last_modified_at: 2024-08-28 12:00:00 +0100
description: How to properly handle exceptions for Task.WhenAll.
categories: [Tech, Software Development]
tags: [Async]
pin: false
#math: false
#toc: false
image:
  path: /assets/img/posts/321587/cover1.png
---

The `Task.WhenAll` in .NET is a powerful construct for coordinating multiple asynchronous I/O operations. It represents a mechanism to mediate concurrent actions. Contrary to popular belief, it doesn't create threads on your behalf. It doesn't create concurrency, it just helps you manage it effectively.

In this article, we will focus on the exception handling and the common pitfalls. We might be compelled just to wrap the call in a try/catch block as shown in the below snippet. However, there are several issues with this approach. The `Task.WhenAll` returns a new `Task` which indeed defines an `Exception` state of type `AggregateException`. However, once the task is awaited, the exception is unwrapped, and a single exception is thrown. And it's important to note that the thrown exception is not an `AggregateException` of all the exceptions thrown by all the tasks. That implies that if an exception has occurred in more than one task, only one of them will be caught in the catch block. Does that mean that the thrown exception is always of type `Exception` and never `AggregateException`? No, that's not the case either. The unwrapping happens only for the main Task returned by `Task.WhenAll`. But if the inner Task has thrown an `AggregateException`, then that's the exception that will be caught in the catch block. In a nutshell, having a simple try/catch block won't provide a full picture of all the errors that have occurred during the execution of these tasks.

```csharp
try
{
    await Task.WhenAll(tasks);
}
catch (Exception ex)
{
}
```

So, what's the proper approach? Moreover, if a single exception is thrown, which one? Let's explore the scenarios in the following sections and try to summarize the rules at the end.

## Example1

In this scenario, we have a list of 3 tasks and each of them has a slightly different delay. If we hit a breakpoint in the catch block, we'll observe the following.
- The `ex` is of type `Exception`.
- The message is "Task3". That's the exception thrown by `Task3()` method.

So, we have the first hint. Since the `Task3()` has the lowest delay, that implies that always the first exception is thrown. And the type of the exception will be whatever is thrown by this method. In this case it is `Exception`, but if this method throws an `AggregateException` then that's what will be caught in the block.

```csharp
var cts = new CancellationTokenSource();

List<Task> tasks =
[
    Task1(cts.Token),
    Task2(cts.Token),
    Task3(cts.Token)
];

try
{
    await Task.WhenAll(tasks);
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}

async Task Task1(CancellationToken cancellationToken)
{
    await Task.Delay(1100, cancellationToken);
    throw new Exception("Task1");
}

async Task Task2(CancellationToken cancellationToken)
{
    await Task.Delay(1200, cancellationToken);
    throw new Exception("Task2");
}

async Task Task3(CancellationToken cancellationToken)
{
    await Task.Delay(1000, cancellationToken);
    throw new Exception("Task3");
}
```


## Example2

In this scenario, we have the same example. However, the token will be canceled after 500ms, earlier than the delay in the methods. We'll observe the following.
- The `ex` will be of type `TaskCanceledException`
- The message will be "A task was canceled.".

This makes sense. The `cts` throws after 500ms, and since we're passing the token to the methods they all have been terminated.

```csharp
var cts = new CancellationTokenSource();

cts.CancelAfter(500);

List<Task> tasks =
[
    Task1(cts.Token),
    Task2(cts.Token),
    Task3(cts.Token)
];

try
{
    await Task.WhenAll(tasks);
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}

async Task Task1(CancellationToken cancellationToken)
{
    await Task.Delay(1100, cancellationToken);
    throw new Exception("Task1");
}

async Task Task2(CancellationToken cancellationToken)
{
    await Task.Delay(1200, cancellationToken);
    throw new Exception("Task2");
}

async Task Task3(CancellationToken cancellationToken)
{
    await Task.Delay(1000, cancellationToken);
    throw new Exception("Task3");
}
```


## Example3

Let's make a minor change to the previous scenario. In the `Task1()` method we'll ignore the provided token and pass `CancellationToken.None` to `Task.Delay`.
- The `ex` will be of type `Exception`.
- The message will be "Task1".

This seems weird. The `cts` will throw a `TaskCanceledException` before the `Task1()` and yet it's not the one unwrapped by `Task.WhenAll`. This implies that the cancellations have a different status. It has lower precedence than all the other exceptions perhaps? But, what if `Task1` actually throws `TaskCanceledException` itself manually with a given message (e.g. `throw new TaskCanceledException("Task1")`)? In that case, the caught exception will be of type `TaskCanceledException` but with the message "A task was canceled.". This implies it's the exception thrown by `cts` since it's the first one and not the one from `Task1`.

```csharp
var cts = new CancellationTokenSource();

cts.CancelAfter(500);

List<Task> tasks =
[
    Task1(cts.Token),
    Task2(cts.Token),
    Task3(cts.Token)
];

try
{
    await Task.WhenAll(tasks);
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}

async Task Task1(CancellationToken cancellationToken)
{
    await Task.Delay(1100, CancellationToken.None);
    throw new Exception("Task1");
}

async Task Task2(CancellationToken cancellationToken)
{
    await Task.Delay(1200, cancellationToken);
    throw new Exception("Task2");
}

async Task Task3(CancellationToken cancellationToken)
{
    await Task.Delay(1000, cancellationToken);
    throw new Exception("Task3");
}
```


## Example 4

Let's examine one more scenario. In this case, the `Task2` method is not marked as async, and we're not awaiting anything. We throw an exception immediately. What will be caught in this case?

Well, we won't even reach the try/catch block at all. The application will crash at line `Task2(cts.Token)`. This method is executed synchronously like any other method in C# and will throw immediately. It's important to note that this will happen even for async methods if the exception occurs before the first `await` statement. The execution is fully synchronous up to the point we reach the first `await` where the rest of the code *might* be dispatched in a separate thread.

This is important detail, and usually is missed by most developers. The assumption that methods that return Task will never throw synchronously is a flawed assumption. We should always take this in account while trying to handle exceptions for `Task.WhenAll`.

```csharp
var cts = new CancellationTokenSource();

cts.CancelAfter(500);

List<Task> tasks =
[
    Task1(cts.Token),
    Task2(cts.Token),
    Task3(cts.Token)
];

try
{
    await Task.WhenAll(tasks);
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}

async Task Task1(CancellationToken cancellationToken)
{
    await Task.Delay(1100, CancellationToken.None);
    throw new Exception("Task1");
}

Task Task2(CancellationToken cancellationToken)
{
    throw new Exception("Task2");
}

async Task Task3(CancellationToken cancellationToken)
{
    await Task.Delay(1000, cancellationToken);
    throw new Exception("Task3");
}
```


## Summary

Ok, let's summarize the rules and what we learned so far.

- The `Task.WhenAll` returns a new task (let's call it a main task) that reflects the completion state of all inner tasks.
- The `Task.WhenAll` completes when all inner tasks have been completed, either successfully or not.
- If more than one task throws an exception, the first exception to occur is the one thrown by the main task.
- If all thrown exceptions are of the type `OperationCancelledException`, analogously the first one will be thrown by the main task.
- If the cancellation is requested and all inner tasks are terminated, the expression thrown is the one from the cancellation source.
- If any of the tasks throws an exception other than `OperationCancelledException`, that's the one thrown by the main task regardless if there are `OperationCancelledException` exceptions thrown before it.


## Final Solution

Now that we're aware of the all different cases, let's try to create a proper exception handling for `Task.WhenAll`. We want to return an `AggregateException` that contains all the exceptions occurred in all of the tasks, including the `OperationCancelled` exceptions (you may omit these exceptions if you want, it depends on your context). Also, to keep it as generalized as possible, we'll receive the delegates for the inner tasks as a parameter. 

```csharp
public static async Task HandleAsync(
    IEnumerable<Func<CancellationToken, Task>> taskProviders, 
    CancellationToken cancellationToken)
{
    List<Exception>? exceptions = null;
    var tasks = new List<Task>();

    // Some of the tasks may throw an immediate exception.
    foreach (var taskProvider in taskProviders)
    {
        try
        {
            tasks.Add(taskProvider(cancellationToken));
        }
        catch (AggregateException ex)
        {
            (exceptions ??= []).AddRange(ex.Flatten().InnerExceptions);
        }
        catch (Exception ex)
        {
            (exceptions ??= []).Add(ex);
        }
    }

    try
    {
        await Task.WhenAll(tasks).ConfigureAwait(false);
    }
    catch
    {
        foreach (var task in tasks)
        {
            if (task.IsFaulted)
            {
                if (task.Exception.InnerExceptions.Count > 1 
                    || task.Exception.InnerException is AggregateException)
                {
                    (exceptions ??= []).AddRange(task.Exception.Flatten().InnerExceptions);
                }
                else if (task.Exception.InnerException is not null)
                {
                    (exceptions ??= []).Add(task.Exception.InnerException);
                }
            }
            else if (task.IsCanceled)
            {
                try
                {
                    // This will force the task to throw the exception if it's canceled.
                    // This will preserve all the information compared to creating a new TaskCanceledException manually.
                    task.GetAwaiter().GetResult();
                }
                catch (Exception ex)
                {
                    (exceptions ??= []).Add(ex);
                }
            }
        }
    }

    if (exceptions?.Count > 0)
    {
        throw new AggregateException(exceptions);
    }
}
```

I hope you found this article useful. Happy coding!