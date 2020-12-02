---
# layout: post
# author: Fati Iseni
title: "Open-Close Principle and runtime DI configurations!"
date: 2020-12-02 17:00:00 +0100
description: How to improve your design and adhere to Open-Close Principle by using dynamic/runtime DI configuration.
categories: [Software Development]
tags: [dependency injection, dotnetcore, design patterns, software architecture]
image: /assets/img/pozitron-cover.png
pin: false
# math: true
# toc: true
---
I just recently published a Nuget package that offers some extensions to the .NET Core's built-in DI container, practically extensions to `IServiceCollection`. The extensions provide the ability for dynamic/runtime DI configuration, through external config files. You can find the repo [here](https://github.com/fiseni/PozitronDev.DIConfiguration).

This is not a new approach at all, and in some other platforms is being heavily used. If anyone used Java with Spring, at some point, you surely have utilized XML based injection. On the other hand in .NET this is actively discouraged, for a good reason. Giving up the compile-time error proofing not always is a bright idea. Of course, there is a difference in how "linking" and "reflection" works in Java and .NET, and that affects the design decisions. But, generally, strongly typed and compile-time configurations are always favored in .NET world.

Then, what all this is about? What's the benefit here, if any?

## Motivation

Shortly said, such a design offers the ability to switch implementations on the fly, on runtime, not requiring to re-compile the solution; restarting the application is all you need. But, do we really need that? We already have other mechanisms and patterns on how to provide some form of flexibility in this context. Before going any further, let's just shortly remind ourselves of Open-Close Principle, and what that means.

As a brief recap, the OCP predicates that we should have constructs that are open to extensions and close to changes. This means if we need to add a "behavior" to a solution, or even to a class; we should be able to do that without changing the existing constructs. Even more simplified, if you have switch statements and too many conditional logic; it might be a sign that the behavior is too much hardcoded, and might be refactored in a better way.

The main logic and the driver here is that by modifying existing constructs, you're increasing the likelihood to introduce new bugs and issues to the existing features. If you find yourself refactoring the class and its methods over and over again, you'll likely end up with some inconsistencies. Also, you'll sneak in, and update/refactor your existing unit tests to reflect the new reality you created, which is not good practice at all. By striving to design an architecture, where you can add new requirements by just creating new constructs, you'll get a more robust and error-prone solution. That's all what OCP means. 

Once that said, now let's imagine a scenario and try to offer various solutions to the problem in hand. Imagine you have an enterprise application (e.g. ERP solution), and you offering it to various clients. One of the requirements of the solution is to calculate some price for some particular workflow. As the number of your clients increases, they all want some minor, or major changes in how this calculation is done. The rest of the workflow remains the same, but the price calculation varies significantly depending on the customer.

## Solution 1

Let's start with the most simple solution. We have one single method which holds all the logic and handles all the requirements.

```c#
public class MyService
{
    private readonly decimal someBaseValue;

    public MyService(decimal someBaseValue)
    {
        this.someBaseValue = someBaseValue;
    }

    public decimal GetCalculatedPrice(string clientType)
    {
        if (clientType.Equals("clientType1"))
        {
            return Math.Round(this.someBaseValue * 10, 2);
        }
        else if (clientType.Equals("clientType2"))
        {
            return Math.Round(this.someBaseValue * 10 + 3, 0);
        }
        else if (clientType.Equals("clientType3"))
        {
            return (this.someBaseValue - 2) * 5;
        }

        throw new NotSupportedException();
    }
}
```

Obviously, this is not the best option. If we get a new client who requires new calculation rules, we'll end up updating the same method. We'll be forced to update/modify the unit test for this method as well. Not a great place to be.

## Solution 2

In order to improve this, first and foremost, we can start by extracting the calculation logic for each customer into separate methods. 

But, let's take a moment and re-think it. Why methods? Why not separate classes? Why not classes which implement a particular interface?


```c#
public interface IPriceCalculator
{
    decimal GetPrice(decimal someBaseValue);
}

public class Type1Calculator : IPriceCalculator
{
    public decimal GetPrice(decimal someBaseValue)
    {
        return Math.Round(someBaseValue * 10, 2);
    }
}

public class Type2Calculator : IPriceCalculator
{
    public decimal GetPrice(decimal someBaseValue)
    {
        return Math.Round(someBaseValue * 10 + 3, 0);
    }
}

public class Type3Calculator : IPriceCalculator
{
    public decimal GetPrice(decimal someBaseValue)
    {
        return (someBaseValue - 2) * 5;
    }
}

public class MyService
{
    private readonly decimal someBaseValue;

    public MyService(decimal someBaseValue)
    {
        this.someBaseValue = someBaseValue;
    }

    public decimal GetCalculatedPrice(string clientType)
    {
        IPriceCalculator calculator = null;

        if (clientType.Equals("clientType1"))
        {
            calculator = new Type1Calculator();
        }
        else if (clientType.Equals("clientType2"))
        {
            calculator = new Type2Calculator();
        }
        else if (clientType.Equals("clientType3"))
        {
            calculator = new Type3Calculator();
        }

        if (calculator != null)
        {
            return calculator.GetPrice(this.someBaseValue);
        }   
        
        throw new NotSupportedException();
    }
}
```

We extracted the logic for each client into a separate class. We defined a contract/interface to standardize the output of the operation. We still have the ugly conditional logic, but this seems much better. If we have to change the calculation for any client, we'll change the corresponding class, and nothing else.

Note: In the newer versions, in `C#8` and `C#9`, we can simplify the conditions with pattern matching. For the sake of simplicity, we'll stick to the traditional IFs.

## Solution 3

Would be nice to get rid of the "selector" logic completely. What if each calculator holds the information for whom they're meant to?

```c#
public interface IPriceCalculator
{
    string ClientType { get; }
    decimal GetPrice(decimal someBaseValue);
}

public class Type1Calculator : IPriceCalculator
{
    public string ClientType { get; } = "clientType1";

     public decimal GetPrice(decimal someBaseValue)
    {
        return Math.Round(someBaseValue * 10, 2);
    }
}

public class Type2Calculator : IPriceCalculator
{
    public string ClientType { get; } = "clientType2";

    public decimal GetPrice(decimal someBaseValue)
        {
        return Math.Round(someBaseValue * 10 + 3, 0);
    }
}

public class Type3Calculator : IPriceCalculator
{
    public string ClientType { get; } = "clientType3";

    public decimal GetPrice(decimal someBaseValue)
        {
        return (someBaseValue - 2) * 5;
    }
}

public class MyService
{
    private readonly decimal someBaseValue;
    private readonly IEnumerable<IPriceCalculator> priceCalculators;

    public MyService(decimal someBaseValue,
                     IEnumerable<IPriceCalculator> priceCalculators)
    {
        this.someBaseValue = someBaseValue;
            this.priceCalculators = priceCalculators;
    }

    public decimal GetCalculatedPrice(string clientType)
    {
        var calculator = priceCalculators.FirstOrDefault(x => x.ClientType.Equals(clientType));

        if (calculator == null)
        { 
            throw new NotSupportedException();
        }

        return calculator.GetPrice(this.someBaseValue);
    }
}
```
Also, you should register all implementations in your DI container

```c#
services.AddScoped<IPriceCalculator, Type1Calculator>();
services.AddScoped<IPriceCalculator, Type2Calculator>();
services.AddScoped<IPriceCalculator, Type3Calculator>();
```

This is way better, right? We have no hard-coded conditional logic at all. Even if we have to create a new calculator, we can simply add a new class (new implementation) and wire it up to our DI container.
The downside of this approach is that we're creating instances we don't actually need. We need one single calculator, and yet we're instantiating all of them.

## Solution 4

In this last option, we'll try to mitigate the last issue (not an issue per se). We want to inject one single implementation, the required one for the corresponding client. And this is the case where dynamic/runtime DI configuration comes really handy. 

Configuration in the `appsetting.json`

```json
{
  "Bindings": {
    "binding1": {
      "service": "SampleLibrary.IPriceCalculator, SampleLibrary",
      "implementation": "SampleLibrary.Type1Calculator, SampleLibrary",
      "scope": "scoped"
    }
  }
}
```

Wiring it up in DI container

```c#
public class Startup
{
    public IConfiguration Configuration { get; }

    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddBindings(Configuration);
    }
}
```

No more we need the type identifier in the interface

```c#
public interface IPriceCalculator
{
    decimal GetPrice(decimal someBaseValue);
}
```

And the actual implementation in our service

```c#
public class MyService
{
    private readonly decimal someBaseValue;
    private readonly IPriceCalculator priceCalculator;

    public MyService(decimal someBaseValue,
                     IPriceCalculator priceCalculator)
    {
        if (priceCalculator == null)
            throw new ArgumentNullException(nameof(priceCalculator));

        this.someBaseValue = someBaseValue;
        this.priceCalculators = priceCalculators;
    }

    public decimal GetCalculatedPrice()
    {
        return calculator.GetPrice(this.someBaseValue);
    }
}
```

I do believe this is quite a clean solution. No longer instantiating and injecting unnecessary objects, no longer you have hard-coded selector logic, and no longer you end up modifying the same constructs over and over again. For each client, you create and deploy a different configuration file, injecting the correct and requested implementation.

## Conclusion

Having the option to switch the implementations easily and modify the behavior of the solution on the fly, is quite a handy and nice feature. The only caveat is that we gave up the commodity of the compile-time checks. Although, it's not that the configuration issues/error will just sneak into our code. If there is a misconfiguration we'll get a runtime exception during the startup, so it's easy to spot and rectify the issues.

Anyhow this is a love/hate game :) The important point is having a choice, and then you go and choose your own design and whatever fits you most!

Cheers!