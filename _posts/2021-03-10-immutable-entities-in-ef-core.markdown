---
# layout: post
# author: Fati Iseni
title: "Immutable entities and value object in EF Core!"
date: 2021-03-10 17:00:00 +0100
description: How to implement and persist immutable entities and value object in Entity Framework Core.
categories: [Software Development]
tags: [EFCore, dotnetcore]
image: /assets/img/pozitron-cover.png
pin: false
# math: true
# toc: true
---
*You can find the code and the full sample [here](https://github.com/fiseni/ImmutabilitySample).*

Immutability is a trendy topic nowadays. The immutable constructs can be handy and can offer many benefits in different scenarios. They can significantly improve your domain model by capturing and mimicking various business processes more accurately. We won't dive into all the pros and cons of adopting the immutability, which will require a separate article. We'll only demonstrate how to implement and persist them in `EF Core` once you have decided to utilize such constructs in your design.

Let's start with a sample model as shown below

```c#
public class Order
{
    public int Id { get; set; }
    public string OrderNo { get; set; }
    public DateTime Date { get; set; }
    public string CustomerFirstName { get; set; }
    public string CustomerLastName { get; set; }
    public string CustomerEmail { get; set; }
    public string Street { get; set; }
    public string City { get; set; }
    public string PostalCode { get; set; }
    public string Country { get; set; }
    public decimal GrandTotal { get; set; }

    public List<OrderItem> OrderItems = new List<OrderItem>();
}

public class OrderItem
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}
```

In this scenario, we have defined `Order` and `OrderItem` models. For now, they're fully anemic models, with no behavior, and no encapsulation applied. Simply said, they're just data objects. In the next sections, we'll improve the model and define the necessary configurations for `EF Core`.

## Designing the domain model

While designing the entities and the domain model in general, it is wise to step back, disconnect yourself from the code and think of the actual business domain in the real world. In our case, first and foremost, is there any concept in the real-world which corresponds to an "empty order"? Imagine you're trying to buy a product, but you oppose sharing any information about the recipient, name, address to be shipped, or any other info. That doesn't make much sense right? The "order" becomes a "thing" once it contains a certain piece of information, otherwise, it's just a meaningless word. Having that in mind, then the question is why would we allow an empty `Order` object in our application? Obviously, our object should utilize a constructor, and accept all required information as parameters. Once instantiated, we want to be sure that the object is not in a corrupted state.

The next step would be to go through all the properties and try to make sense of what they represent and what changes we can introduce to improve the model
- `Id` - This is an identifier of the entity/record. In our case, it's not a business concept but just required technical information. Once the entity is assigned an identifier, there is no reason for the `Id` to change.
- `OrderNo` - This represents the number of the document/order. This should be a unique identifier, composed of some given business rules. Once the document gets its number, no longer we should be able to change it.
- `Date` - It is the date of the document. In this example, we'll require the exact date and time to be set automatically by the system itself, not as an input by the user. Additionally, instead of directly setting the date, we'll accept a given service that can provide the required information. This will enable us to mock the service and write better unit tests.
- `Customer` information - We'd like to group all customer information in a separate construct, and we'll use a value object for that purpose. The customer information should be provided while we create the order. No changes should be allowed, otherwise, that would represent a completely different order.]
- `Address` information - We might group the address information in a separate value object too. It is required information to create an order, but the customer can change their mind and update the desired shipping address.
- `GrandTotal` - It's the sum of the prices of the items in the order. The amount should be automatically updated as we add or delete items from the order.
- `OrderItems` - It's a list of items in the current order. In this example, once we add an item, we can no longer change it, but only delete it. We also want to encapsulate the actions of adding/deleting items within the `Order` entity.

Now that we defined our model, we can refactor the `Order` and `OrderItem` as follows.

```c#
public class Customer : ValueObject
{
    public string FirstName { get; }
    public string LastName { get; }
    public string Email { get; }

    public Customer(string firstName, string lastName, string email)
    {
        if (string.IsNullOrEmpty(firstName)) throw new ArgumentNullException(nameof(firstName));
        if (string.IsNullOrEmpty(lastName)) throw new ArgumentNullException(nameof(lastName));

        this.FirstName = firstName;
        this.LastName = lastName;
        this.Email = email;
    }

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return this.FirstName;
        yield return this.LastName;
        yield return this.Email;
    }
}

public class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string PostalCode { get; }
    public string Country { get; }

    public Address(string street, string city, string postalCode, string country)
    {
        if (string.IsNullOrEmpty(street)) throw new ArgumentNullException(nameof(street));
        if (string.IsNullOrEmpty(city)) throw new ArgumentNullException(nameof(city));
        if (string.IsNullOrEmpty(postalCode)) throw new ArgumentNullException(nameof(postalCode));
        if (string.IsNullOrEmpty(country)) throw new ArgumentNullException(nameof(country));

        this.Street = street;
        this.City = city;
        this.PostalCode = postalCode;
        this.Country = country;
    }

    protected override IEnumerable<object> GetAtomicValues()
    {
        yield return this.Street;
        yield return this.City;
        yield return this.PostalCode;
        yield return this.Country;
    }
}

public class OrderItem
{
    public int Id { get; }
    public string Name { get; }
    public decimal Price { get; }

    public OrderItem(string name, decimal price)
    {
        if (string.IsNullOrEmpty(name)) throw new ArgumentNullException(nameof(name));

        this.Name = name;
        this.Price = price;
    }
}

public class Order
{
    public int Id { get; }
    public string OrderNo { get; }
    public DateTime Date { get; }
    public Customer Customer { get; }
    public Address Address { get; private set; }
    public decimal GrandTotal { get; private set; }

    private readonly List<OrderItem> _orderItems = new List<OrderItem>();
    public IEnumerable<OrderItem> OrderItems => _orderItems.AsEnumerable(); 

    public Order(IDateTime dateTimeService, string orderNo, Customer customer, Address address)
    {
        _ = dateTimeService ?? throw new ArgumentNullException(nameof(dateTimeService));
        _ = customer ?? throw new ArgumentNullException(nameof(customer));
        if (string.IsNullOrEmpty(orderNo)) throw new ArgumentNullException(nameof(orderNo));

        //this.Id = if we decide to use DB generated value no action required.
        
        this.OrderNo = orderNo;
        this.Date = dateTimeService.Now;
        this.Customer = customer;

        UpdateAddress(address);
    }

    public void UpdateAddress(Address address)
    {
        _ = address ?? throw new ArgumentNullException(nameof(address));

        if (this.Address is null || !this.Address.Equals(address))
        {
            this.Address = address;
        }
    }

    public OrderItem AddItem(OrderItem orderItem)
    {
        _ = orderItem ?? throw new ArgumentNullException(nameof(orderItem));

        _orderItems.Add(orderItem);

        CalculateGrandTotal();

        return orderItem;
    }

    public void DeleteOrderItem(int orderItemId)
    {
        var orderItem = OrderItems.FirstOrDefault(x => x.Id == orderItemId);
        _ = orderItem ?? throw new KeyNotFoundException($"The order item with Id: {orderItemId} is not found!");

        _orderItems.Remove(orderItem);

        CalculateGrandTotal();
    }

    private void CalculateGrandTotal()
    {
        this.GrandTotal = this.OrderItems.Sum(x => x.Price);
    }
}
```

We captured all the rules and modeled the entities accordingly. As you can notice, some of the properties do not expose even private setters. There is no reason for those properties to change within or out of the boundaries of the class. It's important to emphasize that the compiler will generate readonly shadow backing fields for these properties, so they are purely immutable. We also defined the `OrderItem`, `Customer`, and `Address` objects as fully immutable.

## EF Core configuration

Now that we have properly designed our domain model, the question is how to persist them in this form. As the encapsulation is concerned, `EntityFramework Core` has great support. If configured, it can work directly with private backing fields allowing better control of what we publicly expose. We can even define automatic properties, and as long they expose at least private setters, `EF Core` can automatically work with the private shadow fields. This usage is quite common and works quite well. But, it's a less known fact that `EF Core` can do much better and utilize even constructors while creating instances of the given entities. By leveraging this feature, we can design purely immutable entities.
Before defining the configuration, I'd like to emphasize few topics which can save you in the troubleshooting process.
- EF by default excludes/ignores all the properties with getters only. It assumes they are calculated properties. For those properties, we should explicitly configure EF to include them in the model.
- If we have included readonly properties in the model, EF will require a constructor (it may be marked as private) with matching parameters for these properties. You must have a constructor which accepts parameters only for these properties, not including the mutable ones.
- The constructor parameters should be named exactly as the properties. Excluding the first character, the naming is case-sensitive. For a given `FirstName` property, you may define the parameter as `FirstName` or `firstName`, but not as `firstname`. This is a common mistake.
- If you have defined immutable owned types (e.g., value objects) as part of the entity, you might be compelled to add them as constructor parameters. Doing so will be an incorrect configuration, and you'll face runtime exceptions. `EF Core` creates instances of these objects separately and has a different internal mechanism of mapping these objects to the entity.

Finally, the entities will have the following form (`Customer` and `Address` value objects remain unchanged):

```c#
public class OrderItem
{
    public int Id { get; }
    public string Name { get; }
    public decimal Price { get; }

    public int OrderId { get; }

    private OrderItem(int id, string name, decimal price, int orderId)
    {
        this.Id = id;
        this.Name = name;
        this.Price = price;
        this.OrderId = orderId;
    }

    public OrderItem(string name, decimal price)
    {
        if (string.IsNullOrEmpty(name)) throw new ArgumentNullException(nameof(name));

        this.Name = name;
        this.Price = price;
    }
}

public class Order
{
    public int Id { get; }
    public string OrderNo { get; }
    public DateTime Date { get; }
    public Customer Customer { get; }
    public Address Address { get; private set; }
    public decimal GrandTotal { get; private set; }


    private readonly List<OrderItem> _orderItems = new List<OrderItem>();
    public IEnumerable<OrderItem> OrderItems => _orderItems.AsEnumerable();

        
    // Just to demostrate calculated properties.
    public string GrandTotalNormalized => this.GrandTotal.ToString("n2");

        
    private Order(int id, string orderNo, DateTime date)
    {
        this.Id = id;
        this.OrderNo = orderNo;
        this.Date = date;
    }

    public Order(IDateTime dateTimeService, string orderNo, Customer customer, Address address)
    {
        _ = dateTimeService ?? throw new ArgumentNullException(nameof(dateTimeService));
        _ = customer ?? throw new ArgumentNullException(nameof(customer));
        if (string.IsNullOrEmpty(orderNo)) throw new ArgumentNullException(nameof(orderNo));

        //this.Id = if we decide to use DB generated value no action required.
            
        this.OrderNo = orderNo;
        this.Date = dateTimeService.Now;
        this.Customer = customer;

        UpdateAddress(address);
    }

    /// It's important not to update the Address if there are no changes.
    /// If updated, since it's a new object, EF tracker will mark it as New.
    /// This can have implications on "Auditing" logic, if you have implemented one. 
    public void UpdateAddress(Address address)
    {
        _ = address ?? throw new ArgumentNullException(nameof(address));

        if (this.Address is null || !this.Address.Equals(address))
        {
            this.Address = address;
        }
    }

    public OrderItem AddItem(OrderItem orderItem)
    {
        _ = orderItem ?? throw new ArgumentNullException(nameof(orderItem));

        _orderItems.Add(orderItem);

        CalculateGrandTotal();

        return orderItem;
    }

    public void DeleteOrderItem(int orderItemId)
    {
        var orderItem = OrderItems.FirstOrDefault(x => x.Id == orderItemId);
        _ = orderItem ?? throw new KeyNotFoundException($"The order item with Id: {orderItemId} is not found!");

        _orderItems.Remove(orderItem);

        CalculateGrandTotal();
    }

    private void CalculateGrandTotal()
    {
        this.GrandTotal = this.OrderItems.Sum(x => x.Price);
    }
}
```

The required EF configuration for these entities is as following

```c#
public class OrderItemConfiguration : IEntityTypeConfiguration<OrderItem>
{
    public void Configure(EntityTypeBuilder<OrderItem> builder)
    {
        builder.ToTable(nameof(OrderItem));

        builder.Property(x => x.Name).IsRequired().HasMaxLength(100);
        builder.Property(x => x.Price).HasPrecision(18, 2);

        // Even though we're not configuring anything specific for OrderId property,
        // it's crucial to write the following line, and let the EF know that we want to include this property.
        // EF by default, excludes all properties with getters only. It assumes they are calculated properties.
        builder.Property(x => x.OrderId);

        builder.HasKey(x => x.Id);
    }
}

public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable(nameof(Order));

        // Even though we're not configuring anything specific for Date property,
        // it's crucial to write the following line, and let the EF know that we want to include this property.
        // EF by default, excludes all properties with getters only. It assumes they are calculated properties.
        builder.Property(x => x.Date);


        builder.Property(x => x.OrderNo).IsRequired().HasMaxLength(10);
        builder.Property(x => x.GrandTotal).HasPrecision(18, 2);

        builder.OwnsOne(x => x.Customer, o =>
        {
            o.WithOwner();

            o.Property(x => x.FirstName).IsRequired().HasMaxLength(100);
            o.Property(x => x.LastName).IsRequired().HasMaxLength(100);
            o.Property(x => x.Email).HasMaxLength(100);
        });

        builder.OwnsOne(x => x.Address, o =>
        {
            o.WithOwner();

            o.Property(x => x.Street).IsRequired().HasMaxLength(250);
            o.Property(x => x.City).IsRequired().HasMaxLength(100);
            o.Property(x => x.PostalCode).IsRequired().HasMaxLength(10);
            o.Property(x => x.Country).IsRequired().HasMaxLength(100);
        });

        builder.Metadata.FindNavigation(nameof(Order.OrderItems))
                .SetPropertyAccessMode(PropertyAccessMode.Field);

        builder.HasKey(x => x.Id);
    }
}
```
Once you have this in place, `EF Core` will be able to do its magic and your application with run smoothly.
Hopefully, this article will be of help to you, and you'll be able to improve your domain model and embrace immutability whenever is required.

You can find the full sample [here](https://github.com/fiseni/ImmutabilitySample).