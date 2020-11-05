---
# layout: post
# author: Fati Iseni
title: "What is a domain model?"
date: 2020-11-05 12:00:00 +0100
categories: [Blogging, Software Development]
tags: [ddd, design patterns, software architecture]
image: /assets/img/pozitron-cover.png
pin: false
# math: true
toc: false
---
I've been asked this question very often, especially by the new developers. So, let's try to explain the term and clear some misconceptions.
The term is being popularized with the adoption of DDD (domain-driven design), but nowadays I think is broadly misunderstood. Usually, I get the following explanations:
- It's just the data objects that map to your DB tables.
- Actually, the entities. If you're doing code-first, the entities are your domain model.
- You must have behavior in your data objects. You shouldn't have anemic objects.

Almost all of the answers I get, usually have to do with some "data constructs". Well, none of that is correct, or at least it's just partially true.
I think, unfortunately, we have overused the term "model"; to a point where it got highly ambiguous meaning. Being used in so many different contexts, developers attach their own meaning to any term containing the word "model".

In the context of `domain model`, it simply means "modeling". See, as the architects for an instance, try to model the layout of the city in a map through drawing; we're trying to model the business rules and processes through code. We try capture each procedure, each rule, each workflow of the particular business; and convert all of that into code. We do that with all the available tools and constructs we have in a certain programming language.

So, `domain model` is a collection of constructs which accurately models one particular business domain. Simply put, it's a collection of entities, enumerations, value objects, exceptions/custom exceptions, interfaces, services, etc. All of that represents the domain model. Anything that has any impact in describing the business domain. On the other hand, any construct that don't take part in this description; the UI code, infrastructure related code, persistence objects, various tools, etc; are considered as third-party/outer concerns in the context of DDD.

The domain model should change only when business requirements change, and never otherwise. Whatever construct fits into this category, is part of your domain model.

