---
layout: post
title: "The Problem With Service Objects Part 2: \"ActiveRecord Solutions\""
date: "2019-03-08"
categories: rails
---

In the [previous post][previous], we discussed a common architectural pitfall in Rails apps: creating "cycles" and loops using callbacks to make related records update each other. In this post I will discuss some possible solutions, staying inside the bounds of things you learn from the [Rails Guide][railsguide]. I've tried all of these in the past in various production applications, and have discovered the pros and cons firsthand.

Before I begin, I would like to point out that I was recently reminded that [ActiveRecord actually has a "touch" option for `#belongs_to`][belongs_to]. I would suggest using it normally. I feel that my example can stand in for any number of situations that ActiveRecord does **not** have an option for (and I'm curious if said option will trigger 20 parent updates when inserting 20 children at once).

## Idea 1: Move the callback to ChildrenController

```ruby
class ChildrenController < ApplicationController
  def update
    if @child.update(child_params) && @parent.touch
      redirect_to @parent
    else
      render action: :edit
    end
  end
end
```

This is not too bad, but you probably already noticed that it should be
inside a database transaction.

### Pros:

* Easy change.
* It will only happen in the place you specify.

### Cons:

* Actually increases the complexity of the controller a lot if you do the full-blown database transaction. At that point we are really exceeding the purview of a controller.
* We want this to happen any time a Child is updated for any reason, but it will only happen if the user hits this specific controller. A recipe for future development mistakes.


## Idea 2: Controller code can never modify a Child directly, only the Parent can do that.

```ruby
class Parent < ApplicationRecord
  has_many :children

  def update_child(id, params)
    self.class.transaction do
      child = children.find(id)
      child.update!(params)
      self.touch
      child
    end
  end
end
```

### Pros:

* Clear chain of responsibility. Controller only messes with Parent.

### Cons:

* Risks colliding with ActiveRecord code.
* Arduous and redundant to implement every possible child action.
* Fairly confusing and unexpected.

## Idea 3: Don't update anything! Let the database figure it out!

```ruby
class Parent < ApplicationRecord
  has_many :children

  def updated_at
    most_recent_child = children.order(updated_at: :desc).first
    [updated_at, most_recent_child.updated_at].max
  end
end
```

We could even always join parents with their most recently-updated child with some fancy SQL and default scoping!

### Pros:

* Always going to be the correct date.

### Cons:

* A recipe for N+1 queries.
* Not really any less work, will have subtle bugs.
* Could potentially be slow with large data sets unless you do some work in the database to optimize it.

### Conclusion:

In most real-world applications, there will be a strong case for just updating the column on the parent, to keep the database querying simple. You do not want to be fetching every single thing related to the parent in order to compute a single property of it.

## Why are none of these ideas making us happy?

**We are looking at the problem from the wrong layer of abstraction.**
Our application isn't a "Parent" or "Child", it is a great big ball of parents and children and any other data we choose to store. When an HTTP request comes in, we mutate the application state. **We need to start thinking of Rails controllers as interacting with the application itself**, not an individual model.


[railsguide]: https://guides.rubyonrails.org/
[belongs_to]: https://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html#method-i-belongs_to
[next]: {{ site.baseurl }}
[previous]: {{ site.baseurl }}{% post_url 2019-03-07-the-problem-with-service-objects-part-1 %}
