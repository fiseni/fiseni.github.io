---
# layout: post
# author: Fati Iseni
title: "Asynchronous programming in ASP.NET web stack!"
date: 2023-05-16 12:00:00 +0100
description: How to apply TPM (async/await) in various components of ASP.NET
categories: [Software Development]
tags: [dotnet]
pin: false
#math: false
#toc: false
image:
  path: /assets/img/posts/896547/cover.png
---
In .NET we have three patterns for asynchronous operations, Asynchronous Programming Model (APM), Event-based Asynchronous Pattern (EAP) and Task-based Asynchronous Pattern (TAP). Nowadays, the TAP is ubiquitous and the recommended asynchronous pattern. The TAP model greatly simplifies the asynchronous calls with async/await, and that's the way to go. 

Unfortunately, that's not the case for some of the older platforms and frameworks. Since the TAP was not yet introduced at that time, there was no native support for it. Instead, there are some well-defined "workarounds" on how to make use of the async/await pattern in the older ASP.NET web applications. In this article, I'll go through various components in ASP.NET and describe how to use asynchronous code in them.

## ASP.NET MVC/WebApi

These are fairly new frameworks and there is native support for async/await. You don't need to apply any special workaround.

## ASP.NET WebForms (aspx)

Let's say we have the following handler in a given aspx page.

```csharp
protected void Button1_Click(object sender, EventArgs e)
{
    // Some logic
    var customer = _customerService.GetCustomer(1);
    TextBox1.Text = customer.Name;
    // Some other logic
}
```

The `CustomerService` also exposes an async method `GetCustomerAsync`, and we would like to utilize it in our handler. We need to apply the following changes:
- Open the markup of the page and add the `Async="true"` directive. It would be something as shown below.
```csharp
<%@ Page Async="true" Title="Home Page" Language="C#" MasterPageFile="~/Site.Master" AutoEventWireup="true" CodeBehind="Default.aspx.cs" Inherits="HttpClientDemo.WebForms._Default" %>
```
- Create a new method with `async Task` signature and copy the whole content of the handler into it. We'll adopt a convention where we'll name these methods same as the handlers, and we'll just add the `Async` suffix.
- In the original handler, we'll use `RegisterAsyncTask` to register the newly created method for asynchronous execution.

We'll end up with the following code. This is all we need to do, so we can make async calls from our pages.

```csharp
protected void Button1_Click(object sender, EventArgs e)
{
    RegisterAsyncTask(new PageAsyncTask(Button1_ClickAsync));
}

protected async Task Button1_ClickAsync()
{
    // Some logic
    var customer = await _customerService.GetCustomerAsync(1);
    TextBox1.Text = customer.Name;
    // Some other logic
}
```

### What not to do
- Do not use `async void` for the handlers. In some platforms like WinForms/WPF we don't have a choice, but for WebForms pages we have a better solution. So, let's adopt the above solution.
- Do not partially extract the content of the handlers to the new async method. We have to understand that with `RegisterAsyncTask` we're just registering the action for asynchronous execution, but the handler is still a void method, and it won't wait for the execution to complete. So, any code that you have after `RegisterAsyncTask` will be executed immediately. Avoid adding any code after it, unless you're completely sure that it is not dependent on the results or side effects of the async method.


## ASP.NET HttpHandlers (ashx)

We have a lot of HTTP handlers in our solution, and we might need to execute async methods. Luckily, it's quite easy to do that for HTTP handlers. All we need is to inherit from `HttpTaskAsyncHandler` abstract base class.
Let's say we have the following handler.
```csharp
public class CustomerHandler : IHttpHandler
{

    public void ProcessRequest(HttpContext context)
    {
        var customerService = new CustomerService();
        var customer = customerService.GetCustomer(1);
        context.Response.ContentType = "text/plain";
        context.Response.Write(customer.Name);
    }

    public bool IsReusable
    {
        get
        {
            return false;
        }
    }
}
```

If we want to utilize the `GetCustomerAsync` method, we'll update the handler as follows:

```csharp
public class CustomerHandler : HttpTaskAsyncHandler
{
    public override async Task ProcessRequestAsync(HttpContext context)
    {
        var customerService = new CustomerService();
        var customer = await customerService.GetCustomerAsync(1);
        context.Response.ContentType = "text/plain";
        context.Response.Write(customer.Name);
    }
}
```

## ASP.NET WebServices (asmx)

This is the trickiest one. The web services (asmx) don't have support for the TAP model (async/await), but they do have support for APM. The APM was the first asynchronous model introduced in `.NET Framework 1.0`. In this pattern, asynchronous operations require Begin and End methods (for example, BeginSomething and EndSomething to implement an asynchronous operation). It's a legacy model and no longer recommended. We have written some TAP to APM interop extensions, so we can consume the Task based methods from our web services. I'll spare you from the details and demonstrate the usage.

Let's say we have the following web service.

```csharp
public class CustomerWebService : System.Web.Services.WebService
{
    [WebMethod]
    public string GetCustomerName(int id)
    {
        var customerService = new CustomerService();
        var customerName = customerService.GetCustomerName(id);
        return customerName;
    }
}
```

If we want to utilize the async methods in `CustomerService`, we'll have to make the following changes.

```csharp
public class CustomerWebService : System.Web.Services.WebService
{
    [WebMethod]
    public IAsyncResult BeginGetCustomerName(int id, AsyncCallback callback, object state)
    {
        var customerService = new CustomerService();
        return customerService.GetCustomerNameAsync(id).AsApm(callback, state);
    }

    [WebMethod]
    public string EndGetCustomerName(IAsyncResult result)
    {
        return result.Unwrap<string>();
    }
}
```

The APM interop implementation is as follows.

```csharp
public static class ApmInteropExtensions
{
    public static IAsyncResult AsApm(this Task task, AsyncCallback callback, object state)
    {
        if (task == null)
        {
            throw new ArgumentNullException(nameof(task));
        }

        var tcs = new TaskCompletionSource<object>(state);

        task.ContinueWith(t =>
        {
            if (t.IsFaulted)
            {
                tcs.TrySetException(t.Exception.InnerExceptions);
            }
            else if (t.IsCanceled)
            {
                tcs.TrySetCanceled();
            }
            else
            {
                tcs.TrySetResult(null);
            }

            if (callback != null)
            {
                callback(tcs.Task);
            }
        }, TaskScheduler.Default);

        return tcs.Task;
    }

    public static IAsyncResult AsApm<T>(this Task<T> task, AsyncCallback callback, object state)
    {
        if (task == null)
        {
            throw new ArgumentNullException(nameof(task));
        }

        var tcs = new TaskCompletionSource<T>(state);

        task.ContinueWith(t =>
        {
            if (t.IsFaulted)
            {
                tcs.TrySetException(t.Exception.InnerExceptions);
            }
            else if (t.IsCanceled)
            {
                tcs.TrySetCanceled();
            }
            else
            {
                tcs.TrySetResult(t.Result);
            }

            if (callback != null)
            {
                callback(tcs.Task);
            }
        }, TaskScheduler.Default);

        return tcs.Task;
    }

    public static void Unwrap(this IAsyncResult asyncResult)
    {
        if (asyncResult == null)
        {
            throw new ArgumentNullException(nameof(asyncResult));
        }

        if (asyncResult is Task task)
        {
            task.GetAwaiter().GetResult();
        }
        else
        {
            throw new ArgumentException("Invalid asyncResult", nameof(asyncResult));
        }
    }

    public static T Unwrap<T>(this IAsyncResult asyncResult)
    {
        if (asyncResult == null)
        {
            throw new ArgumentNullException(nameof(asyncResult));
        }

        if (asyncResult is Task<T> task)
        {
            return task.GetAwaiter().GetResult();
        }
        else
        {
            throw new ArgumentException("Invalid asyncResult", nameof(asyncResult));
        }
    }
}
```

Let's summarize what we have done here:
- For a given web method, we have created two methods with the same name. Added `Begin` and `End` prefixes to the name of the methods.
- The `Begin` method will accept the original input parameters, and additionally the `AsyncCallBack` and `state` (these will be provided by the framework). It returns `IAsyncResult`.
- The `End` method will accept `IAsyncResult` (provided by the framework) and will return the original output.

Note: The `EndGetCustomerName` won't run on the original thread and it won't have access to the original context. It means you won't have access to `HttpContext.Current`. Bear that in mind and avoid some complex logic in `End` methods. Its job is to just extract and return the result.

I hope you found the article useful and happy coding!