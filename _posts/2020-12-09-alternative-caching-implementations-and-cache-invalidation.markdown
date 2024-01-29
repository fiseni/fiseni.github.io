---
# layout: post
# author: Fati Iseni
title: "Alternative caching implementations and cache invalidation!"
date: 2020-12-09 17:00:00 +0100
description: Alternative implementation for caching, and on-demand cache invalidation using EF (on changes only).
categories: [Software Development]
tags: [cache, dotnetcore, design patterns, software architecture]
img_path: '/assets/img/pozitron-cover.png'
pin: false
# math: true
# toc: true
---
Today I'll be talking about caching and various ways to implement the same. We'll also show how to invalidate the cache on demand if using Entity Framework.

The caching feature is almost mandatory for your applications, especially if you have a lot of relatively read-only data. It can drastically boost up the performance, by reducing the roundtrips to your database or any other persistence infrastructure that you might have.

The standard and most common way of implementing caching is by using a decorator pattern (or is it proxy pattern? Everlasting discussion) and invalidating the cache on specific time intervals.

OK, let's imagine a scenario, and try to provide various possible solutions. We have `Customer` data in our application, and we utilize this information very often. In the case of a desktop application, we might have a sort of drop-down or search controls; and in a web application might be a search field with a drop-down suggestion list. Other than that, the application uses customers heavily in various scenarios. The behavior of several features depends on the customer, and we querying this information quite often.

To sum up the requirements:

- We have customers entity/table in our app.

- The customer table contains roughly 1.000 records.

- The records barely change, once per day, or even once per week.

- But, once the data is changed, it's crucial we have the new data and work with the most up-to-date information.

- We require a complete list of customers, for whatever reason.

Based on all information above, we surely want to utilize some caching infrastructure and reduce the number of queries to persistence.

## Solution 1

The common approach, as mentioned above, is to use the decorator pattern. Initially, we have a service/repository (or whatever construct) which queries the persistence and gets the data. Then, we create an additional implementation that contains the same actions but wraps/decorates each of them with caching functionality. The rest of the application consumes the cached variant of the service.

The customer entity

```c#
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Address { get; set; }
    public string Email { get; set; }
}
```

```c#
public interface ICustomerRepository
{
    Task<List<Customer>> GetCustomers();
}

public class CustomerRepository
{
    private readonly AppDbContext dbContext;

    public CustomerRepository(AppDbContext dbContext)
    {
        this.dbContext = dbContext;
    }

    public Task<List<Customer>> GetCustomers()
    {
        return dbContext.Customers.ToListAsync();
    }
}

public class CustomerCachedRepository : ICustomerRepository
{
    private readonly IMemoryCache cache;
    private readonly CustomerRepository customerRepository;

    private readonly string cacheKey = nameof(Customer);
    private readonly TimeSpan cacheDuration = TimeSpan.FromSeconds(1);

    public CustomerCachedRepository(IMemoryCache cache,
                                    CustomerRepository customerRepository)
    {
        this.cache = cache;
        this.customerRepository = customerRepository;
    }

    public Task<List<Customer>> GetCustomers()
    {
        return cache.GetOrCreateAsync(cacheKey, entry =>
        {
            entry.SlidingExpiration = cacheDuration;
            return customerRepository.GetCustomers();
        });
    }
}
```

We'll register the services as scoped, as shown below. Note that the cached repository accepts concrete implementation and not the interface, otherwise we'll end up with a circular dependency.

```c#
services.AddMemoryCache();
services.AddScoped<ICustomerRepository, CustomerCachedRepository>();
services.AddScoped<CustomerRepository>();
```

## Solution 2

The previous solution is quite straightforward and does the job well. The only issue with this approach is the "timer". We are invalidating the cache on strictly defined time intervals. Choosing the right time interval is not as easy as it seems. Define a short interval (a couple of seconds) and you end with too many queries, define a longer one and you risk working with obsolete data.

In our scenario, customers barely change, and yet we end up querying them each second. So, we should define the interval in hours maybe? In that case, when changes occur, we surely will end up corrupting our data.

So, let's try another approach, implement caching manually and invalidate the cache on-demand, on change only. We'll create a new specific interface and name it `ICachedDataService`. We'll modify the `CustomerRepository` as following

```c#
public interface ICachedDataService
{
    Task Reload(AppDbContext dbContext);
}

public interface ICustomerRepository
{
    IEnumerable<Customer> Customers { get; }
}

public class CustomerRepository : ICustomerRepository, ICachedDataService
{
    private static readonly object _instanceLock = new object();

    public IEnumerable<Customer> Customers { get; private set; }

    public async Task Reload(AppDbContext dbContext)
    {
        var customers = await dbContext.Customers.ToListAsync();
        lock (_instanceLock)
        {
            Customers = customers;
        }
    }
}
```

No doubt this construct should have a singleton scope. We want a single instance so the list of customers will act as static information. Surely, you can have a static class here, or implement a singleton pattern manually. But, be aware in that case the consumers will be strongly coupled to the implementation, and you might no be able to easily switch it and unit test the consumers properly. My advice, if you already using a DI container, just let it handle the scope for you, and utilize constructor injection in the consumers.

```c#
services.AddSingleton<ICustomerRepository, CustomerRepository>();
services.AddSingleton<ICachedDataService, CustomerRepository>();
```

And, finally, let's invalidate the cache in the persistence implementation, in our case EntityFramework. Override the SaveChanges as shown below

```c#
public class AppDbContext : DbContext
{
    private readonly ICachedDataService cachedDataService;

    public DbSet<Customer> Customers { get; set; }

    public AppDbContext(DbContextOptions<AppDbContext> options,
                        ICachedDataService cachedDataService)
        : base(options)
    {
        this.cachedDataService = cachedDataService;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.ApplyConfiguration(new CustomerConfiguration());
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var isCustomerChanged = ChangeTracker.Entries<Customer>().Where(x => x.IsAddedOrModifiedOrDeleted()).Count() > 0;

        var response = await base.SaveChangesAsync(cancellationToken);

        if (isCustomerChanged)
        {
            await cachedDataProvider.Reload(this);
        }

        return response;
    }
}
```

Note the extension method. If entities have owned types, you may want to check them for changes as well. There are some improvements in the newer versions, but previously, EF was not flagging the entity as modified if any of the owned types have changed.

```c#
public static class ChangeTrackerExtensions
{
    public static bool IsAddedOrModifiedOrDeleted(this EntityEntry entry) =>
        entry.State == EntityState.Deleted ||
        entry.State == EntityState.Added ||
        entry.State == EntityState.Modified ||
        entry.References.Any(r => r.TargetEntry != null &&
                                    r.TargetEntry.Metadata.IsOwned() &&
                                    (r.TargetEntry.State == EntityState.Added || r.TargetEntry.State == EntityState.Modified));
}
```

Now, whenever the customer information is changed, we'll reload the cached data immediately, and the consumers will work with up-to-date customers.

Initially, we'll read/reload the data on application startup. You can do it as shown here.

```c#
public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateHostBuilder(args).Build();

        using (var scope = host.Services.CreateScope())
        {
            var services = scope.ServiceProvider;
            var dbContext = services.GetRequiredService<AppDbContext>();
            var cachedDataService = services.GetRequiredService<ICachedDataService>();

            await cachedDataService.Reload(dbContext);
        }

        host.Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

## Solution 3

What if we want several entities cached? Let's say we want to cache a list of countries too.

We'll start by creating a few additional abstractions and refactor the implementations. Let's define an interface which will mark the entities that we want to cache.

```c#
public interface ICacheInfo
{
    string CacheKey { get; }
}
public class Customer : ICacheInfo
{
    public string CacheKey => nameof(Customer);

    public int Id { get; set; }
    public string Name { get; set; }
    public string Address { get; set; }
    public string Email { get; set; }
}
public class Country : ICacheInfo
{
    public string CacheKey => nameof(Country);

    public int Id { get; set; }
    public string Name { get; set; }
    public string Code { get; set; }
}
```

And few changes in the repository implementations

```c#

public interface ICachedDataService : ICacheInfo
{
    Task Reload(AppDbContext dbContext);
}

public interface ICustomerRepository
{
    IEnumerable<Customer> Customers { get; }
}

public interface ICountryRepository
{
    IEnumerable<Country> Countries { get; }
}

public class CustomerRepository : ICustomerRepository, ICachedDataService
{
    private static readonly object _instanceLock = new object();

    public string CacheKey { get; } = nameof(Customer);

    public IEnumerable<Customer> Customers { get; private set; }

    public async Task Reload(AppDbContext dbContext)
    {
        var customers = await dbContext.Customers.ToListAsync();
        lock (_instanceLock)
        {
            Customers = customers;
        }
    }
}

public class CountryRepository : ICountryRepository, ICachedDataService
{
    private static readonly object _instanceLock = new object();

    public string CacheKey { get; } = nameof(Country);

    public IEnumerable<Country> Countries { get; private set; }

    public async Task Reload(AppDbContext dbContext)
    {
        var countries = await dbContext.Countries.ToListAsync();
        lock (_instanceLock)
        {
            Countries = countries;
        }
    }
}
```

Now let's modify the SaveChanges and the Reload implementations as well.

```c#
public class AppDbContext : DbContext
{
    private readonly IEnumerable<ICachedDataService> cachedDataServices;

    public DbSet<Customer> Customers { get; set; }
    public DbSet<Country> Countries { get; set; }

    public AppDbContext(DbContextOptions<AppDbContext> options,
                        IEnumerable<ICachedDataService> cachedDataServices)
        : base(options)
    {
        this.cachedDataServices = cachedDataServices;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.ApplyConfiguration(new CustomerConfiguration());
        modelBuilder.ApplyConfiguration(new CountryConfiguration());
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var keys = ChangeTracker.Entries<ICacheInfo>()
                                .Where(x => x.IsAddedOrModifiedOrDeleted())
                                .Select(x => x.Entity.CacheKey)
                                .Distinct()
                                .ToList();

        var response = await base.SaveChangesAsync(cancellationToken);

        foreach (var key in keys)
        {
            await cachedDataServices.FirstOrDefault(x => x.CacheKey.Equals(key))?.Reload(this);
        }

        return response;
    }
}
```

In `EF Core 5`, we can use interceptors or events, so no need to overload the `SaveChangesAsync` method. But, the above approach will work in almost all versions, and that's why I'm using it here.

Based on the new modified infrastructure, we have to be careful about how we register the services in the DI container. For an instance, we want both `ICustomerRepository` and `ICachedDataService` to return the same instance. If we do the following, we'll get two singleton instances for each interface separately. And, that's not what we want.

```c#
services.AddSingleton<ICustomerRepository, CustomerRepository>();
services.AddSingleton<ICachedDataService, CustomerRepository>();
```

In order to mitigate this, we can simply create the instance manually and provide it to the DI container.

```c#
var customerRepository = new CustomerRepository();
services.AddSingleton<ICustomerRepository>(customerRepository);
services.AddSingleton<ICachedDataService>(customerRepository);

var countryRepository = new CountryRepository();
services.AddSingleton<ICountryRepository>(countryRepository);
services.AddSingleton<ICachedDataService>(countryRepository);
```

Finally, we'll modify the startup code as well

```c#
public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateHostBuilder(args).Build();

        using (var scope = host.Services.CreateScope())
        {
            var services = scope.ServiceProvider;
            var dbContext = services.GetRequiredService<AppDbContext>();
            var cachedDataServices = services.GetRequiredService<IEnumerable<ICachedDataService>>();

            foreach (var cachedDataService in cachedDataServices)
            {
                await cachedDataService.Reload(dbContext);
            }
        }

        host.Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

## Summary

You can find the full sample in the following repo [here](https://github.com/fiseni/CachingSample). In the sample, "Cities" are implemented through `IMemoryCache` and decorator pattern, while the "Customer" and "Countries" with the alternative approach.

The standard caching implementation serves us well in various use cases. Using that approach it's really easy to cache various sets based on the input filters (paginated results, applied criteria, etc).

In specific use cases, where we don't have a large set of data, and we want to hold the whole collection; then the described alternative approaches might be a better solution.

Next time, I'll describe how to improve this infrastructure even further. We'll create better abstractions to remove some boilerplate code, and will provide the ability to cache various filtered sets of the collection. Practically, on cache invalidation, we'll invalidate all caches for that particular entity, and then cache various filtered data on the next first request.
