---
layout: post
title: "Defining an Entity: Hibernate for ActiveRecord users"
date: "2020-07-01"
categories: ["java", "orms", "activerecord", "hibernate", "rails"]
---

I have been learning Spring Boot and JPA+Hibernate at a new job. This experience has been frustrating and humbling. The goal of this series is to assist anyone making the same transition, and also to attempt to put my deep frustration with the JPA+Hibernate philosophy into words. If you feel I am giving bad advice or unfair criticism, please direct your hate mail to: [will@willmakley.dev](will@willmakley.dev). I would truly like to know your opinion!

This blog post will briefly cover some of the conceptual basics of Models vs Entities. 

## Assume the following MySQL tables:

```sql
CREATE TABLE contacts (
  id          INT(11) NOT NULL AUTO_INCREMENT,
  client_id   INTEGER NOT NULL,
  first_name  VARCHAR(255) NOT NULL,
  last_name   VARCHAR(255) NOT NULL,
  CONSTRAINT contacts_pk PRIMARY KEY (id)
);

CREATE TABLE clients (
  id    INT(11) NOT NULL AUTO_INCREMENT,
  name  VARCHAR(255) NOT NULL,
  CONSTRAINT clients_pk PRIMARY KEY (id)
);

CREATE TABLE addresses (
  id          INT(11) NOT NULL AUTO_INCREMENT,
  contact_id  INTEGER NOT NULL,
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

Note that we do not mention all of the columns on the table anywhere. Rails finds these out through introspection of the tables, and by making assumptions. A "Contact" model must have a "contacts" table. Every table must have an "id" primary key column. Etc.

## Hibernate / JPA

In Hibernate you define an "Entity" class for the table, and you have to tell it about every column too.

```java
@Entity
@Table( name = "contacts" )
public class Contact {

  @Id
  // "IDENTITY" means auto-increment - Hibernate has many fun head-scratchers!
  @GeneratedValue( strategy = GenerationType.IDENTITY )
  @Column( name = "id" )                               
  private Long id;

  @NotBlank
  @Column( name = "first_name", length = 100, nullable = false )
  private String firstName;

  @NotBlank
  @Column( name = "last_name", length = 100, nullable = false )
  private String lastName;

  // Relationships:

  // Same ActiveRecord "belongs_to"
  @ManyToOne
  @JoinColumn( name = "client_id", nullable = false )
  private Client client;

  // Same as ActiveRecord "has_many".
  //
  // Unfortunately we have to choose which collection interface to use
  // and manually initialize it, leaving ample opportunity to make the
  // wrong choice or be confused by stockholme syndrome blog posts on
  // the topic. 'Set' ensures no duplicates at least, so we'll go with
  // that.
  @OneToMany( mappedBy = "contact" )
  private Set<Address> addresses;

  public Contact() {
    this.addresses = new HashSet<>();
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

The difference in verbosity is always funny (seriously, who would rather write the Java class?), but _that really isn't the point I want to get across here_.

**These are fundamentally different approaches.**

In Hibernate/JPA, the Entity is a dumb container of data. You must ask the framework gods to "hydrate" it with data through repository objects, JPQL, etc. The @Annotations tell the framework where to put the data, and provide some validations. It is a surprisingly passive way of working, and decidedly odd using Java to define a database schema.

In ActiveRecord, not only does the model also acts as a container for a row in the database, but it also includes all the validation, querying, and "CRUD" methods through inheritence or code generation. Those "presence" validation rules? All happening in the model. You can override them if you want, they are just methods. Query methods? Model. Saving and updating? Model.

In my opinion, there is nothing intrinsically wrong with either approach. Honestly as a Ruby guy I have to admit the Java way of working is architecturally cleaner. Large ActiveRecord models can easily become quite confusing and encompass too many concerns. **Where things go wrong for JPA+Hibernate is in the details.** Hoo boy.
