---
# layout: post
# author: Fati Iseni
title: "How to implement auditing on your entities!"
date: 2020-11-28 17:00:00 +0100
last_modified_at: 2020-11-28 17:00:00 +0100
description: A guide on how to implement auditing for your entities using Entity Framework.
categories: [Software Development]
tags: [EFCore, Auditing]
pin: false
#math: false
#toc: false
image:
  path: /assets/img/pozitron-cover.png
---
Not rarely we require to apply some basic auditing information for our entities. Although we may configure full audit features on the Database side, sometimes it's handy to have this information as part of your entities.
Of course, we want this feature to be processed behind the scenes, automatically, and not deal with it manually. If you're using Entity Framework (EF6, EFCore), implementation is quite straightforward.

## Auditable Entities

Let's create a base class that will hold the audit information. Once created, use it as a base class for all entities that you want to apply auditing.

```c#
public class AuditableEntity
{
    public DateTime? AuditCreatedTime { get; set; }
    public string AuditCreatedByUserId { get; set; }
    public string AuditCreatedByUsername { get; set; }
    public DateTime? AuditModifiedTime { get; set; }
    public string AuditModifiedByUserId { get; set; }
    public string AuditModifiedByUsername { get; set; }
}
```

I like to prefix all the properties (and DB columns) with "Audit". It helps me easily distinct these values, and I like having them grouped when shown by IntelliSense. You can have these properties named differently, and still, have specific names configured for the DB columns.
Other than that, on top of `UserID` I usually tend to persist the `Username` of the user too. This way, if you want to utilize this information and display it on the UI, you won't have to query the Identity persistence each time.

## User information provider

If you're using ASP.NET Core, you can access the authenticated user's information through `HttpContextAccessor`. But, you may want to keep your persistence infrastructure in a separate project, in which case you won't have direct access to this property.
Let's create a simple class that will encapsulate the current user's information, and provide it wherever is required.

```c#
public class CurrentUserProvider : ICurrentUserProvider
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CurrentUserProvider(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public string UserId => _httpContextAccessor.HttpContext?.User?.FindFirstValue(ClaimTypes.NameIdentifier);
    public string Username => _httpContextAccessor.HttpContext?.User?.FindFirstValue(ClaimTypes.Name);
}
```

Also, you should register these services in your DI container as following

```c#
services.AddSingleton<IHttpContextAccessor, HttpContextAccessor>();
services.AddScoped<ICurrentUserProvider, CurrentUserProvider>();
```

## Implement auditing

Now that we have the supporting infrastructure ready, let's make a few modifications in the `DbContext` class.

First, modify the constructor and inject the `ICurrentUserProvider` implementation.

```c#
private readonly ICurrentUserProvider currentUserProvider;

public MyDbContext(DbContextOptions<MyDbContext> options,
                    ICurrentUserProvider currentUserProvider)
    : base(options)
{
    this.currentUserProvider = currentUserProvider;
}
```

Then, override the `SaveChangesAsync` method and add the actual implementation for the auditing.

```c#
public async override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    var addedEntries = ChangeTracker.Entries<AuditableEntity>().Where(x => x.IsAdded());
    var modifiedEntries = ChangeTracker.Entries<AuditableEntity>().Where(x => x.IsModified());

    foreach (var entry in addedEntries)
    {
        entry.CurrentValues[nameof(AuditableEntity.AuditCreatedTime)] = DateTime.Now;
        entry.CurrentValues[nameof(AuditableEntity.AuditCreatedByUserId)] = currentUserProvider?.UserId;
        entry.CurrentValues[nameof(AuditableEntity.AuditCreatedByUsername)] = currentUserProvider?.Username;
    }

    foreach (var entry in modifiedEntries)
    {
        entry.CurrentValues[nameof(AuditableEntity.AuditModifiedTime)] = DateTime.Now;
        entry.CurrentValues[nameof(AuditableEntity.AuditModifiedByUserId)] = currentUserProvider?.UserId;
        entry.CurrentValues[nameof(AuditableEntity.AuditModifiedByUsername)] = currentUserProvider?.Username;
    }

    return await base.SaveChangesAsync(cancellationToken);
}
```

You may notice, I'm using `IsAdded` and `IsModified` extensions, instead of directly utilizing the `EntityState` enum. This is important if your entities hold owned types (e.g. value objects). You want to consider the whole entity as modified if the owned types are added/modified.

```c#
public static class ChangeTrackerExtensions
{
    public static bool IsAdded(this EntityEntry entry) =>
        entry.State == EntityState.Added;

    public static bool IsModified(this EntityEntry entry) =>
        entry.State != EntityState.Added &&
        (entry.State == EntityState.Modified ||
        entry.References.Any(r => r.TargetEntry != null && 
                                    r.TargetEntry.Metadata.IsOwned() && 
                                    (r.TargetEntry.State == EntityState.Added || r.TargetEntry.State == EntityState.Modified)));
}
```

And that's all. On each save, the auditing information will be persisted for your chosen entities.