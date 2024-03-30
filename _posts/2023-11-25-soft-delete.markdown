---
# layout: post
# author: Fati Iseni
title: "Global soft delete in EF Core!"
date: 2023-11-25 12:00:00 +0100
description: How to implement a global soft delete in EF Core.
categories: [Software Development]
tags: [dotnet efcore]
pin: false
#math: false
#toc: false
image:
  path: /assets/img/posts/778542/cover.png
---
In this article, I'll describe how to implement soft delete in EF Core. We would like to apply a global configuration and not on a per-entity basis. The implementation consists of two parts, applying soft delete to the items while saving changes and configuring a global query filter to be applied to the queries.

We'll start by defining an interface `ISoftDelete`. The interface should be implemented by all entities that require this feature.

```csharp
public interface ISoftDelete
{
    public bool IsDeleted { get; }
}
```

## Applying soft delete

EF Core is under active development and many new features have been added with each new version. I'll provide not only the final implementation but also its evolution over time. With each new version, I've been updating the soft-delete implementation to account for the new features and changes in EF Core.

### Option 1
In the beginning, this was simple. We loop through all the deleted items in the tracker and update the state to `Modified`.

```csharp
public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    foreach (var entry in ChangeTracker.Entries<ISoftDelete>())
    {
        if (entry.State != EntityState.Deleted) continue;

        entry.State = EntityState.Modified;
        entry.CurrentValues[nameof(ISoftDelete.IsDeleted)] = true;
    }

    return base.SaveChangesAsync(cancellationToken);
}
```

### Option 2
With the introduction of owned entity types, the above implementation no longer works. In the case of OwnsOne, the type might be mapped to the same parent table. Considering that owned types internally are designed as fully blown entities (with generated PK/FK shadow properties), then for each parent, we must find the owned types and change their state as well. Failing to do so will result in an exception.

```csharp
public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    foreach (var entry in ChangeTracker.Entries<ISoftDelete>())
    {
        if (entry.State != EntityState.Deleted) continue;

        entry.State = EntityState.Modified;
        entry.CurrentValues[nameof(ISoftDelete.IsDeleted)] = true;

        var ownedEntries = entry.References
            .Where(x => x.TargetEntry is not null && x.TargetEntry.Metadata.IsOwned());

        foreach (var ownedEntry in ownedEntries)
        {
            if (ownedEntry.TargetEntry is not null)
            {
                ownedEntry.TargetEntry.State = EntityState.Modified;
            }
        }
    }

    return base.SaveChangesAsync(cancellationToken);
}
```

### Option 3
In Option2, we didn't have to account for OwnsMany since thos entities won't be mapped to the same table anyway. But, with the introduction of JSON arrays in EF Core 8, that is possible now. The `entry.References` don't include collections, and searching through `entry.Collections` or `entry.Navigations` is not straightforward either. So, we have to come up with an alternative approach. We'll fetch all owned entries in the tracker, and then check whether they're owned by a given deleted EntityEntry.

```csharp
public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    List<EntityEntry>? ownedEntries = null;

    foreach (var entry in ChangeTracker.Entries<ISoftDelete>())
    {
        if (entry.State != EntityState.Deleted) continue;

        entry.State = EntityState.Modified;
        entry.CurrentValues[nameof(ISoftDelete.IsDeleted)] = true;

        ownedEntries ??= ChangeTracker.Entries()
            .Where(x => x.State == EntityState.Deleted && x.Metadata.IsOwned())
            .ToList();

        foreach (var ownedEntry in ownedEntries)
        {
            if (ownedEntry.Metadata.IsInOwnershipPath(entry.Metadata.ContainingEntityType))
            {
                ownedEntry.State = EntityState.Modified;
            }
        }
    }

    return base.SaveChangesAsync(cancellationToken);
}
```

## Configuring query filter
The EF Core offers a handy API to define a query filter for a given entity. All we need is to apply the following configuration.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Product>().HasQueryFilter(x=>x.IsDeleted);
}
```

Doing this for every entity is a tedious task. And we always run the risk of forgetting to add the configuration whenever we add a new entity. Instead, let's try to implement this globally. We'll create an extension as follows.

```csharp
public static class EFCoreExtensions
{
    private static readonly MethodInfo _softDeleteFilterMethodInfo = typeof(EFCoreExtensions)
            .GetMethod(nameof(GetSoftDeleteFilter), BindingFlags.NonPublic | BindingFlags.Static)!;

    private static LambdaExpression GetSoftDeleteFilter<TEntity>() where TEntity : class, ISoftDelete
    {
        Expression<Func<TEntity, bool>> filter = x => !x.IsDeleted;
        return filter;
    }

    public static void ConfigureSoftDelete(this ModelBuilder modelBuilder)
    {
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            // In case we have inherited types, it's important to add it only for roots.
            var isRootType = entityType.GetRootType() == entityType;

            if (isRootType && typeof(ISoftDelete).IsAssignableFrom(entityType.ClrType))
            {
                // The property is immutable (contains only a getter)
                // By default will be ignored by EF, so we need to add it to the model explicitly.
                modelBuilder.Entity(entityType.Name, x => x.Property(nameof(ISoftDelete.IsDeleted)));

                var methodToCall = _softDeleteFilterMethodInfo.MakeGenericMethod(entityType.ClrType);
                var filter = methodToCall.Invoke(null, Array.Empty<object>());

                entityType.SetQueryFilter((LambdaExpression?)filter);
                entityType.AddIndex(entityType.FindProperty(nameof(ISoftDelete.IsDeleted))!);
            }
        }
    }
}
```

Now, we can apply the query filter for all entities (that implement the `ISoftDelete` interface) in our model.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ConfigureSoftDelete();
}
```

I hope you found the article useful and happy coding!