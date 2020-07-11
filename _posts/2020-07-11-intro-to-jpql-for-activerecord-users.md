---
layout: post
title: "Intro to JPQL for ActiveRecord Users"
date: 2020-07-11
categories: ["jpa", "spring", "java", "hibernate"]
---

## Introduction

In the [previous blog post][previous], I demonstrated how to define a JPA entity, and how it differs from an ActiveRecord model. In this and subsequent posts I will show some ways to load entities, and when to, and - more importantly - when *not* to use them.

To recap, querying ActiveRecord models looks like this:

```ruby
# Example of fetching multiple Contacts by last name,
# and pre-loading each contact's Client to avoid N+1
# queries.
contacts = Contact.where(last_name: "Flintstone")
                  .includes(:client)

contacts.each do |contact|
  puts "#{contact.last_name}: #{contact.client.name}"
end
```

Basically you use the model class directly, and a simple DSL that you can chain as much as you like. It is "lazy", so nothing will be loaded until you call "load", "each", "map", etc. This will more or less generate the SQL you expect, as at any time you may call `#to_sql` to see what will happen.

Here are the JPA choices for achieving the same result that I will discuss:

* **JPQL - this post**
* Repositories
* QueryDSL

Here are the choices I will *not* discuss, because they are an over-complicated waste of time:

* Criteria Query API (Google it yourself and cry)

**Already you may be asking:** Why do we need three separate APIs, with three separate blog posts? This is a very good question!!! If you have an answer as to why this sucks so much, please [let me know][me]!

## What is JPQL?

JPQL is an SQL-like language for querying entities (not tables), defined by the JPA specification.

Yes, you read that correctly: **entities, not tables**.

## Detour: Getting an Entity Manager instance

Before we can do anything with JPQL, we need an instance of the EntityManager. In Spring Boot you can have this auto-injected into your class as follows:

```java
@PersistenceContext
private EntityManager entityManager;
```

This will handle threading properly. Don't ask me how it works, but apparently it *is* possible for the framework to modify private variables through Java's reflection API. Who knew! (It makes me feel slightly queezy.)

## JPQL Simple Example

```java
List<Contact> contacts = entityManager.createQuery(
    "SELECT c FROM Contact c WHERE c.lastName = :lastName",
    Contact.class
  )
  .setParameter("lastName", "Flintstone")
  .getResultList();

// Example of looping through them to print their names:
for (Contact contact : contacts) {
  System.out.println( contact.getLastName() );
}
```

Not too bad, right? You might be thinking it looks type-safe, fairly intuitive, and succinct.

### Lazy Loading

Now that we have our Contact entities, what happens if we try to access relationships?

```java
for (Contact contact : contacts) {
  System.out.println( contact.getLastName() + ": " +
                      contact.getClient().getName() );
}
```

As you might expect, the Client object will be lazily loaded. How, you ask? The Contacts are actually wrapped by "proxy" objects that do the lazy loading. `getClient()` returns a proxy, which only goes to the database if you call a method such as `client.getId()`. (This is yet another piece of Hibernate magic that doesn't seem possible in the Java type system, but there you are.)

This could be a whole blog post itself, so we will leave it at that for now.

### Preventing N+1

Since that way of accessing clients has an N+1 issue, let's modify this query to pre-load each Contact's Client relationship, like we did in Ruby:

```java
List<Contact> contacts = entityManager.createQuery(
    "SELECT c FROM Contact c " +
    "JOIN FETCH c.client cl " +
    "WHERE c.lastName = :lastName",
    Contact.class
  )
  .setParameter("lastName", "Flintstone")
  .getResultList();
```

JPQL will figure out the joins for you, and has a special "fetch" keyword to signify pre-loading. You also have to reference "client" as a property of "contact": `c.client`. Again, in this query language we are dealing with entities, not tables.

> Aside: The blogosphere on avoiding N+1 issues in JPA seems woefully lacking. Many posts treat the reader like they are five years old, and maybe aren't ready to tackle such "complex" issues, or spend pages beating about the bush. Documentation rarely offers straightforward solutions to common problems. **Note that ActiveRecord's `.includes(...)` method is obvious and easy to use, and may be explained in five seconds.** This is typical. Do I sound frustrated?

## When to use it

* When you have a custom one-off query of some complexity, with an un-changing structure.

## When NOT to use it

* When the query structure needs to change based on user input. (Most search queries.)
* Any time there is an equally clear and succinct alternative.

## Comparison to ActiveRecord

* `ActiveRecord::Relations` are chainable and composable (see example in introduction). JPQL is not (hence why it is bad for searches).
* If you want to pre-load any relationships, you will need to specify the join type (left, inner, outer, cross, etc.).
* `JOIN FETCH` actually does a join. ActiveRecord `#includes(...)` will usually result in a separate query.

## What I think

I think JPQL is ridiculous. It combines all the downsides of writing SQL by hand with all the downsides of writing in a language that *looks* like SQL, but isn't actually SQL. I have lost count of the number of things I thought should work, but didn't, and the number of utterly incomprehensible *runtime* error messages I have had to google. One of the main things I want from Java (in exchange for increased verbosity) is compile-time type-safety. JPQL takes it away. :(

I also find the entire idea of thinking in objects instead of tables to be mind-boggling. No wonder everyone has been complaining about the "object-relational impedence mismatch" for years, going so far as to call it "[the Vietnam of Computer Science][vietnam]" in some cases.

How does ActiveRecord not fall into this trap? It provide a simple API that *helps you to write lots of syntactically correct SQL quickly and succinctly*, instead of forcing you to learn a new object-querying language.

Unfortunately JPQL is unavoidable, and most things you do with JPA will hearken back to it in some way. Sigh.


[me]: mailto:will@willmakley.dev
[vietnam]: https://blog.codinghorror.com/object-relational-mapping-is-the-vietnam-of-computer-science/
[previous]: {{ site.baseurl }}{% post_url 2020-07-01-defining-an-entity-jpa-for-active-record-users %}
