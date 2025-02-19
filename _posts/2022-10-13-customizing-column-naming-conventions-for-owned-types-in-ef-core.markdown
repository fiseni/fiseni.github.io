---
# layout: post
# author: Fati Iseni
title: "Customizing Column Naming Conventions for Owned Types in EF Core"
date: 2022-10-13 12:00:00 +0100
last_modified_at: 2022-10-13 12:00:00 +0100
description: Learn how to customize column naming conventions for owned types in EF Core.
categories: [Tech, Software Development]
tags: [EFCore, dotnet]
pin: false
#math: false
#toc: false
image:
  path: /assets/img/posts/231321/cover.webp
  alt: An image generated by ChatGPT. I can't do any better :)
---
EF Core allows you to model entity types that can only ever appear on navigation properties of other entity types. These are called owned entity types. By default, the owned entity types are mapped to the same table as the top-level parent. You can explicitly change this behavior in the configuration. Since the parent and owned types may contain properties with the same name, the generated column names for the owned types will contain the navigation name as a prefix by default. Let's see this in action.

```csharp
public class Foo
{
    public string? Bar { get; set; }
}

public class Address
{
    public string? Street { get; set; }
    public string? City { get; set; }
    public Foo Foo { get; set; }
}

public class Customer
{
    public int Id { get; set; }
    public string? Name { get; set; }
    public Address Address { get; set; }
}
```

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>().OwnsOne(x => x.Address, 
        addressBuilder => addressBuilder.OwnsOne(x => x.Foo));
}
```

For the above model and corresponding configuration, the generated SQL statement is as follows.

```sql
CREATE TABLE [Customers] (
  [Id] int NOT NULL IDENTITY,
  [Name] nvarchar(max) NULL,
  [Address_Street] nvarchar(max) NULL,
  [Address_City] nvarchar(max) NULL,
  [Address_Foo_Bar] nvarchar(max) NULL,
  CONSTRAINT [PK_Customers] PRIMARY KEY ([Id])
);
```

Although this convention is necessary to avoid name conflicts, many teams dislike prefixing the column names. A more compelling argument is when you're trying to match your EF model to an existing database. We can easily achieve that by explicitly setting the column names.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>().OwnsOne(x => x.Address, addressBuilder =>
    {
        addressBuilder.Property(x => x.Street).HasColumnName("Address");
        addressBuilder.Property(x => x.City).HasColumnName("City");
		
        addressBuilder.OwnsOne(x => x.Foo, fooBuilder =>
        {
            fooBuilder.Property(x => x.Bar).HasColumnName("Bar");
        });
    });
}
```

As you may imagine, for large codebases with many entities, overriding all column names might be a tedious task. Luckily, we can configure this convention globally for our EF model.

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>().OwnsOne(x => x.Address,
        addressBuilder => addressBuilder.OwnsOne(x => x.Foo));

    // Call the method at the end, after all other configurations
    modelBuilder.ConfigureOwnedTypeColumnNames();
}
```

```csharp
public static class ModelBuilderExtensions
{
    public static void ConfigureOwnedTypeColumnNames(this ModelBuilder modelBuilder)
    {
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (!entityType.IsOwned()) continue;

            var ownership = entityType.FindOwnership();

            if (ownership is null) continue;

            var properties = entityType.GetProperties().Where(x => !x.IsShadowProperty());

            foreach (var property in properties)
            {
                var tableName = entityType.GetTableName();

                if (tableName is null) continue;

                var columnName = property.GetColumnName(StoreObjectIdentifier.Table(tableName, null));
                var columnNameDefault = property.GetDefaultColumnName(StoreObjectIdentifier.Table(tableName, null));

                if (columnName is null || columnNameDefault is null) continue;

                if (columnName.Equals(columnNameDefault))
                {
                    var columnNameBase = property.GetColumnName();

                    property.SetColumnName(columnNameBase);
                }
            }
        }
    }
}
```

Once we apply this configuration, the generarated SQL statement matches our desired convention.

```sql
CREATE TABLE [Customers] (
  [Id] int NOT NULL IDENTITY,
  [Name] nvarchar(max) NULL,
  [Street] nvarchar(max) NULL,
  [City] nvarchar(max) NULL,
  [Bar] nvarchar(max) NULL,
  CONSTRAINT [PK_Customers] PRIMARY KEY ([Id])
);
```

I hope you found this article useful. Happy coding!