---
# layout: post
# author: Fati Iseni
title: "Current user implementation in ASP.NET Core!"
date: 2023-09-05 12:00:00 +0100
last_modified_at: 2023-09-05 12:00:00 +0100
description: How to implement a current user provider in ASP.NET Core.
categories: [Tech, Software Development]
tags: [ASP.NET, dotnet]
pin: false
#math: false
#toc: false
image:
  path: /assets/img/posts/336845/cover.png
---
In this article, I'll go through a couple of implementations on how to extract and provide user information in ASP.NET Core web applications. We apply authentication and authorization to protect our endpoints, and usually, we utilize the user information throughout our services. That may be for logging purposes, auditing or other reasons. Fetching this data over and over again is neither performant nor convenient. Since this data doesn't change per request, ideally, we'd like to extract it once and provide it in an immutable form to all our services.

Let's start by defining an interface that will expose user information.

```csharp
public interface ICurrentUser
{
    public string? UserId { get; }
    public string? Username { get; }
}
```

## Option 1
The most common approach I see in various solutions is to utilize the `IHttpContextAccessor` to extract the necessary data from the current request. This works, but it's not ideal. It relies on `AsyncLocal` which can have a negative performance impact and creates a dependency on the "ambient state". I try to avoid this interface as much as possible and use it only as a last resort. The implementation would be as follows.

```csharp
public class CurrentUser : ICurrentUser
{
    public string? UserId { get; }
    public string? Username { get; }

    public CurrentUser(IHttpContextAccessor httpContextAccessor)
    {
        var claimsPrincipal = httpContextAccessor.HttpContext?.User;

        if (claimsPrincipal is null) return;

        UserId = claimsPrincipal.FindFirstValue(ClaimTypes.NameIdentifier);
        UserId = claimsPrincipal.FindFirstValue(ClaimTypes.Name);
    }
}
```

Once we do the following registrations in DI, we may consume the `ICurrentUser` from all our services.
```csharp
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentUser, CurrentUser>();
```

## Option 2
A better approach would be to not utilize `IHttpContextAccessor`, instead, use a custom middleware to extract the information early in the request. Since our `ICurrentUser` is fully immutable, we'll create an additional contract.

```csharp
public interface ICurrentUserInitializer
{
    public string? UserId { get; set; }
    public string? Username { get; set; }
}
```

```csharp
public class CurrentUser : ICurrentUser, ICurrentUserInitializer
{
    public string? UserId { get; set; }
    public string? Username { get; set; }
}
```

Now, we go ahead and implement the middleware. The `ICurrentUserInitializer` is not mandatory to have. It's just convenient for testing where we may inject a faker with pre-populated test user data. That's the reason, we're setting the user data only if they're empty `currentUser.UserId ??`.

```csharp
public static class CurrentUserExtensions
{
    public static IApplicationBuilder UseCurrentUser(this IApplicationBuilder app)
    {
        app.Use(async (context, next) =>
        {
            var user = context.User;
            var currentUser = context.RequestServices.GetRequiredService<ICurrentUserInitializer>();

            currentUser.UserId ??= user.FindFirstValue(ClaimTypes.NameIdentifier);
            currentUser.Username ??= user.FindFirstValue(ClaimTypes.Name);

            await next();
        });

        return app;
    }
}
```
Finally, we need to wire up everything as follows.

```csharp
...
builder.Services.AddScoped<CurrentUser>();
builder.Services.AddScoped<ICurrentUserInitializer, CurrentUser>(x => x.GetRequiredService<CurrentUser>());
builder.Services.AddScoped<ICurrentUser, CurrentUser>(x => x.GetRequiredService<CurrentUser>());
var app = builder.Build();

...
app.UseAuthorization();
app.MapControllers();
app.UseCurrentUser();
app.Run();
```

I hope you found the article useful and happy coding!