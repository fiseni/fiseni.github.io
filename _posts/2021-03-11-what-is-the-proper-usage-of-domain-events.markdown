---
# layout: post
# author: Fati Iseni
title: "What is the proper usage of domain events?"
date: 2021-03-11 17:00:00 +0100
description: What is the proper usage of domain events, and when not to use them!
categories: [Software Development]
tags: [ddd, design patterns, dotnetcore]
image: /assets/img/pozitron-cover.png
pin: false
# math: true
# toc: true
---
Events are a powerful concept and they're quite a useful mechanism in various scenarios. They enable us to notify other parties of our internal activities, simply put, broadcasting our actions. We all have used events in one form or another. They are very common in desktop application development, where the "framework" fires events on various user interactions with UI, and we respond to these events by doing something useful. Anyhow, that's not the only place where events can be utilized. If you are fond of Domain-Driven Design (DDD), it's common to implement domain events as a communication mechanism between the aggregates, or just passing some information to the outer "world". It offers some form of loose coupling of different modules, which usually act quite independently within their own boundaries. But, it's very easy and it's very common to misuse the concept of events.

Let's examine the following sample, and try to figure out if we're doing the correct thing.

```c#
public class OrderItem
{
    public int Id { get; }
    public string Name { get; }
    public decimal Quantity { get; private set; }
    public decimal Price { get; }
	
	public int OrderId { get; }

    public event EventHandler QuantityChanged;

    public OrderItem(string name, decimal quantity, decimal price)
    {
        if (string.IsNullOrEmpty(name)) throw new ArgumentNullException(nameof(name));

        this.Name = name;
        this.Quantity = quantity;
        this.Price = price;
    }

    public void UpdateQuantity(decimal quantity)
    {
        this.Quantity = quantity;

        this.QuantityChanged?.Invoke(this, new EventArgs());
    }
}

public class Order
{
    public int Id { get; }
    public decimal GrandTotal { get; private set; }

    private readonly List<OrderItem> _orderItems = new List<OrderItem>();
    public IEnumerable<OrderItem> OrderItems => _orderItems.AsEnumerable();

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

    public void Order_QuantityChanged(object sender, EventArgs e)
    {
        CalculateGrandTotal();
    }

    private void CalculateGrandTotal()
    {
        this.GrandTotal = this.OrderItems.Sum(x => x.Price);
    }
}
```

In this case, whenever the `OrderItem` is changed, it raises an appropriate event. On the other hand, the `Order`, the aggregate root in this sample, listens to this event and updates the `GrandTotal` information on each change of its items. For simplicity, we'll assume the subscribing action happens in some service (e.g., in `OrderService`). This implementation is fairly common, and I see it all the time. I have used this pattern very often in the past too. So, what's wrong here, if anything?

If we try to generalize, we might identify 3 types of communication mechanisms or types of interactions in our system
- Queries - I want to know something. Please provide me this information.
- Commands - I want something to happen. Please do this/that.
- Events - I'm just doing my business. I'm talking loudly, but I really don't care if anyone is listening.

In our sample, the `CalculateGrandTotal` is not a trivial or an optional operation. On contrary, it's a crucial action that must happen to preserve the consistency of the aggregate. If we miss this action we'll certainly end up with corrupted/incorrect data, the `GrandTotal` will contain an invalid value. Having this in mind, the `OrderItem` is not raising the event carelessly, but with a specific intention, it expects something specific to happen as a result. If we can paraphrase, this is what `OrderItem` is shouting 

*"I want to notify the world that I have been changed. But, hey `Order`, it's important you listen to this, and it's important that you do that action. Hey `Order`... `Order`..."*

It's quite clear that we're doing something wrong here. We clearly want something to happen as a side effect of our actions. This no longer should be an event, but it's a command. Instead of notifying someone and hoping that the necessary action will happen, we can simply demand that action. Let's modify the sample according to this.

```c#
public class OrderItem
{
    public int Id { get; }
    public string Name { get; }
    public decimal Quantity { get; private set; }
    public decimal Price { get; }

    public int OrderId { get; }
    public Order Order { get; }

    public OrderItem(string name, decimal quantity, decimal price)
    {
        if (string.IsNullOrEmpty(name)) throw new ArgumentNullException(nameof(name));

        this.Name = name;
        this.Quantity = quantity;
        this.Price = price;
    }

    public void UpdateQuantity(decimal quantity)
    {
        this.Quantity = quantity;

        this.Order?.CalculateGrandTotal();
    }
}

public class Order
{
    public int Id { get; }
    public decimal GrandTotal { get; private set; }

    private readonly List<OrderItem> _orderItems = new List<OrderItem>();
    public IEnumerable<OrderItem> OrderItems => _orderItems.AsEnumerable();

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

    public void CalculateGrandTotal()
    {
        this.GrandTotal = this.OrderItems.Sum(x => x.Price);
    }
}
```

Now, we still keep the calculation logic in `Order`, but its items can demand this action in a non-ambiguous way. The intent is communicated more clearly.

The argument against this approach, usually is "you have to remember to call the method". But, in no form that's any different from raising the event, you still have to remember to raise the event. We can summarize as follows:
- Command case - I should remember to call this function here
- Event case - I should remember to raise the event here. Also, I will need you specifically to subscribe to it.

It's obvious (for me at least), what's the correct thing to do here. The other argument is that we shouldn't have a navigation back to the aggregate. That no longer is an issue, and it's quite safe to use it as it is. But, you can always just accept the aggregates' method as an `Action` parameter, and achieve the same.