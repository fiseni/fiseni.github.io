---
# layout: post
# author: Fati Iseni
title: "Simplifying Configuration for Temporal Tables in EF Core!"
date: 2023-01-15 12:00:00 +0100
description: Learn how to simplify the configuration of temporal tables in EF Core.
categories: [Blogging, Tutorial, Software Development]
tags: [EFCore]
img_path: '/assets/img/pozitron-cover.png'
pin: false
# math: true
# toc: true
---
When dealing with data that changes over time, you may want to keep track of historical changes. Temporal tables in SQL Server offer an excellent solution to this problem, allowing you to automatically track changes to your data. With temporal tables, you can see what your data looked like at any point in time, making it easier to debug issues and track changes over time. Starting with Entity Framework Core 6, the team added support for this feature. In this blog post, we will discuss how to create an extension method for EF Core that automatically configures temporal tables for all entities of interest.

## Configuring temporal tables for simple entities

The required configuration is quite straightforward. Let's assume we have the following model/entity.

```csharp
public class Customer
{
    public int Id { get; set; }
    public string? Name { get; set; }
}
```

The required configuration is as follows:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>().ToTable(x => x.IsTemporal());
}
```

That's all we need to configure a temporal table for `Customer`. The EF will automatically generate shadow properties `PeriodStart` and `PeriodEnd` for our model. If you're using migrations, the following SQL will be generated

```sql
DECLARE @historyTableSchema sysname = SCHEMA_NAME()
EXEC(N'CREATE TABLE [Customers] (
  [Id] int NOT NULL IDENTITY,
  [Name] nvarchar(max) NULL,
  [PeriodEnd] datetime2 GENERATED ALWAYS AS ROW END HIDDEN NOT NULL,
  [PeriodStart] datetime2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
  CONSTRAINT [PK_Customers] PRIMARY KEY ([Id]),
  PERIOD FOR SYSTEM_TIME([PeriodStart], [PeriodEnd])
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = [' + @historyTableSchema + N'].[CustomersHistory]))');
```

## Configuring temporal tables for entities with owned entity types

In EF Core 6 we lacked support for this feature if the entity has defined owned types mapped to the same table. This was added in EF Core 7. But, configuring them is not intuitive nor simple. Let's assume we have the following model now

```csharp
public class Address
{
    public string? Street { get; set; }
    public string? City { get; set; }
}

public class Customer
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public Address? Address { get; set; }
}
```

We assume that the configuration might be as follows

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>().ToTable(x => x.IsTemporal());
    modelBuilder.Entity<Customer>().OwnsOne(x => x.Address, ownedBuilder =>
    {
        ownedBuilder.ToTable(x => x.IsTemporal());
    });
}
```

Unfortunately, this will result in an exception while building the model. 

```
System.InvalidOperationException
  HResult=0x80131509
  Message=When multiple temporal entities are mapped to the same table, their period start properties must map to the same column. Issue happens for entity type 'Customer' with period property 'PeriodStart' which is mapped to column 'PeriodStart'. Expected period column name is 'Address_PeriodStart'.
```

There is some indication that the generated column names for shadow properties of the owned entities is incorrect. The first guess will be to just define the column names for the owned entity only. But, weirdly, that's not enough. In this case we have to be fully explicit as follows:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>().ToTable(x => x.IsTemporal());
    modelBuilder.Entity<Customer>().Property<DateTime>("PeriodStart").HasColumnName("PeriodStart");
    modelBuilder.Entity<Customer>().Property<DateTime>("PeriodEnd").HasColumnName("PeriodEnd");

    modelBuilder.Entity<Customer>().OwnsOne(x => x.Address, ownedBuilder =>
    {
        ownedBuilder.ToTable(x => x.IsTemporal());
        ownedBuilder.Property<DateTime>("PeriodStart").HasColumnName("PeriodStart");
        ownedBuilder.Property<DateTime>("PeriodEnd").HasColumnName("PeriodEnd");
    });
}
```

Finally, this works, and the generated SQL is correct

```sql
DECLARE @historyTableSchema sysname = SCHEMA_NAME()
EXEC(N'CREATE TABLE [Customers] (
  [Id] int NOT NULL IDENTITY,
  [Name] nvarchar(max) NULL,
  [Address_Street] nvarchar(max) NULL,
  [Address_City] nvarchar(max) NULL,
  [PeriodEnd] datetime2 GENERATED ALWAYS AS ROW END HIDDEN NOT NULL,
  [PeriodStart] datetime2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
  CONSTRAINT [PK_Customers] PRIMARY KEY ([Id]),
  PERIOD FOR SYSTEM_TIME([PeriodStart], [PeriodEnd])
) WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = [' + @historyTableSchema + N'].[CustomersHistory]))');
```

## Applying temporal table configuration globally

As shown above, the configuration is a bit overwhelming. Doing it for all your entities is a tedious task. Luckily, we can extend the `ModelBuilder` capabilities and write an extension method that does this automatically for all entities. Ideally, we'd like to to end up with the following configuration.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>().OwnsOne(x => x.Address);

    // Call the method at the end after all other configurations.
    modelBuilder.ConfigureTemporalTables();
}
```

We also may want the ability to choose which entities should be taken into consideration. Therefore, we may add an interface that will mark all the entities that should be tracked with temporal tables.

```csharp
public interface ITemporalEntity
{
}
```

```csharp
public class Address
{
    public string? Street { get; set; }
    public string? City { get; set; }
}

public class Customer : ITemporalEntity
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public Address? Address { get; set; }
}
```

```csharp
public static class ModelBuilderExtensions
{
    public static void ConfigureTemporalTables(this ModelBuilder modelBuilder)
    {
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (entityType.IsOwned())
            {
                if (!entityType.IsMappedToJson())
                {
                    ConfigureTemporalForOwnedType(entityType, entityType);
                }
            }
            else
            {
                if (typeof(ITemporalEntity).IsAssignableFrom(entityType.ClrType))
                {
                    ConfigureTemporal(entityType);
                }
            }
        }
    }

    private static void ConfigureTemporalForOwnedType(IMutableEntityType entityType, IMutableEntityType parentEntityType)
    {
        var ownership = parentEntityType.FindOwnership();

        if (ownership is null) return;

        var parent = ownership.PrincipalEntityType;

        if (parent is null) return;

        if (parent.IsOwned())
        {
            ConfigureTemporalForOwnedType(entityType, parent);
        }
        else if (typeof(ITemporalEntity).IsAssignableFrom(parent.ClrType))
        {
            ConfigureTemporal(entityType);
        }
    }

    private static void ConfigureTemporal(IMutableEntityType entityType)
    {
        entityType.SetIsTemporal(true);

        var periodStart = entityType.FindDeclaredProperty("PeriodStart");
        if (periodStart is not null)
            periodStart.SetColumnName("PeriodStart");

        var periodEnd = entityType.FindDeclaredProperty("PeriodEnd");
        if (periodEnd is not null)
            periodEnd.SetColumnName("PeriodEnd");
    }
}
```

You may notice, we don't even have to mark our owned types with the interface, they'll be discovered automatically. Marking the parent entities is enough.

So, that's it, we're all set. I hope this is helpful!