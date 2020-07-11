---
layout: post
title: "Defining an Entity: JPA for ActiveRecord Users"
date: 2020-07-01
categories: ["java", "jpa", "spring", "activerecord", "hibernate", "rails"]
---

I have been learning Spring Boot and JPA+Hibernate at a new job. This experience has been frustrating and humbling. The goal of this series is to assist anyone making the same transition, and also to attempt to put my deep frustration with the JPA+Hibernate philosophy into words. If you feel I am giving bad advice or unfair criticism, please direct your hate mail to: [will@willmakley.dev](will@willmakley.dev). I would truly like to know your opinion!

This blog post will briefly cover some of the conceptual basics of Models vs Entities. 

> Important terms:
> * JPA = Java Persistence API; an API specification.
> * Hibernate = A JPA implementation.
>
> When writing a standard "Spring Boot" app, it seems to me that one usually
> works as much as possible with JPA classes and constructs. Occasionally one
> may need to use a Hibernate or Spring-specific feature, but this can usually
> be avoided.

## Assume the following MySQL tables:

(I prefer PostgreSQL, but we're sticking with the lowest common denominator here.)

```sql
CREATE TABLE contacts (
  id          INT(11)      NOT NULL AUTO_INCREMENT,
  client_id   INTEGER      NOT NULL,
  first_name  VARCHAR(255) NOT NULL,
  last_name   VARCHAR(255) NOT NULL,
  CONSTRAINT contacts_pk PRIMARY KEY (id)
);

CREATE TABLE clients (
  id    INT(11)      NOT NULL AUTO_INCREMENT,
  name  VARCHAR(255) NOT NULL,
  CONSTRAINT clients_pk PRIMARY KEY (id)
);

CREATE TABLE addresses (
  id          INT(11)      NOT NULL AUTO_INCREMENT,
  contact_id  INTEGER      NOT NULL,
  line_1      VARCHAR(255),
  /* additional varchar fields... */
  CONSTRAINT addresses_pk PRIMARY KEY (id)
);
```

## ActiveRecord

In ActiveRecord you define "models" by extending `ActiveRecord::Base`. Relationships and validation rules are defined using macros. Example:

```ruby
class Contact < ActiveRecord::Base
  # "belongs_to" and "has_many" are class methods defined in
  # ActiveRecord::Base. They create several instance methods
  # and probably some configuration data structure.

  belongs_to :client # The contacts table has a foreign key
                     # pointing to the clients table.

  has_many :addresses # The addresses table has a foreign
                      # key pointing to contacts.

  validates :first_name, presence: true
  validates :last_name, presence: true
end
```

Note that we do not mention all of the columns on the table anywhere. Rails finds these out through introspection of the tables, and by making assumptions. A "Contact" model must have a "contacts" table. Every table must have an "id" primary key column. Etc. (You may of course change these settings, but these are the defaults.)

## JPA

In JPA you define an "Entity" class for the table, and you have to tell it about every column. This serves as an authoritative schema for the table, and may be used to generate schema migrations. (Whereas in Rails, the auto-generated "schema.rb" file is the source of truth.)

Note that we must manually spoon feed to JPA the schema of the tables we already created in detail. While this can be relatively satisfying busywork, I am skeptical of the overall productivity of this approach.

One nice thing is the built-in validation annotations.

```java
@Entity
@Table( name = "contacts" )
public class Contact {

  @Id
  // "IDENTITY" means auto-increment - JPA has many fun head-scratchers!
  @GeneratedValue( strategy = GenerationType.IDENTITY )
  @Column( name = "id" )                               
  private Long id;

  @Column( name = "first_name", length = 100 )
  @NotBlank // in Spring Boot, this implies "not null"
  private String firstName;

  @Column( name = "last_name", length = 100 )
  @NotBlank
  private String lastName;

  // Relationships:

  // Same ActiveRecord "belongs_to"
  @ManyToOne
  @JoinColumn( name = "client_id" )
  @NotNull
  private Client client;

  // Same as ActiveRecord "has_many".
  //
  // Unfortunately we have to choose which collection interface+
  // implementation to use and manually initialize it, leaving ample
  // opportunity to make the wrong choice or to be confused by stockholm
  // syndrome blog posts on the topic. 'Set' ensures no duplicates at
  // least, so we'll go with that (more for ease of manipulating the data
  // than because Hibernate will load duplicates, although I have seen
  // that happen too!).
  @OneToMany( mappedBy = "contact" )
  private Set<Address> addresses;

  public Contact() {
    // LinkedHashSet is an ordered set, meaning we get both no
    // duplicates, and preservation of order.
    this.addresses = new LinkedHashSet<>();
  }

  // I <3 Java boilerplate:

  public Long getId() { return this.id; }
  public void setId( Long id ) { this.id = id; }

  public String getFirstName() { return this.firstName; }
  public void setFirstName( String firstName ) { this.firstName = firstName; }

  public String getLastName() { return this.lastName; }
  public void setNameName( String lastName ) { this.lastName = lastName; }

  public Client getClient() { return this.client; }
  public void setClient( Client client ) { this.client = client; }

  public Set<Address> getAddresses() { return this.addresses; }
  public void setAddresses( Set<Address> addresses ) { this.addresses = addresses; }
}
```

## Conclusion

The difference in verbosity is always funny (seriously, who would rather write the Java class?), but that really isn't the point I want to get across here.

**These are fundamentally different approaches.**

In Hibernate/JPA, the Entity is a dumb container of data. You must ask the framework gods to load them, create them, save them, etc. You have little direct control over how that happens. It is a surprisingly passive way of working, and decidedly odd using Java @Annotations to define a database schema. (In Ruby, this would be most similar to the "[Hanami][hanami]" framework.)

In ActiveRecord, the model class itself contains all of the querying, persistence rules, and even business logic (by default). It works very hard to save you from having to repeat yourself, or manually define boring stuff such as the schema (which it will go and discover for you, like a helpful servant).

In my opinion, history bears out that there is nothing intrinsically wrong with either approach. Honestly as a Ruby guy I have to admit the Java way of working is architecturally clean. Large ActiveRecord models can easily become quite confusing and encompass too many concerns. **Where things go wrong for JPA+Hibernate is in the details.**

[hanami]: https://hanamirb.org/
