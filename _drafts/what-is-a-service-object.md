---
layout: post
title: "Just use a service object?"
date: "2019-03-20"
categories: ["java", "orms", "activerecord", "hibernate", "rails"]
---
In the [previous post][previous], I covered a common problem with callbacks in Rails applications: order of operations issues when you have spiderwebs of records modifying each other. This is the point at which many experienced developers on the Internet will give the advice: "Use a service object!" The problem with this advice because it lacks almost any relevant details. In this post I am going to enumerate what I think makes a good service object, and whether you should use them at all.

## What is a service object?

The main issue with the "service object" is that everyone's definition is different. Here are a few examples I have seen in the wild:

1. Simple objects with a single `call` method. (Basically just a function, as seen in many blog posts like [this one][call-example].)
2. Fancy DSL's for orchestrating multiple functions, such as in [Trailblazer][trailblazer] operations or [Dry-Transaction][dry-txn].
3. An unholy hodgepodge of a class that simply keeps the nightmare contained. (What most developers actually end up with.)

## Which one is "best"?

I think we need to take a step back and think about the big picture. Forget about Rails. At the end of the day, a web application is a bundle of state to be mutated by HTTP requests. **A service object is a "button" that facilitates a single logical user interaction with that state.**

A good service object should:

1. **Be the single obvious piece of code to change with regard to its function.**
2. Handle failure cases clearly and concisely.
3. Assist with orchestrating lower-level services, such as the database.
4. Keep side effects contained and clearly labeled.


## 1. Be the single obvious piece of code to change with regard to its function.

Service objects can be surprising in a Rails code-base. ActiveRecord is expected to do most of the heavy lifting, and `app/models` will be the first place other developers look when trying to understand your code. **If you are going to use service objects, they need to be clearly visible.**

Lets take a very simple example of a "Task" model, and a service that notifies recipients after saving a task:

```ruby
# The model
class Task < ActiveRecord::Base
end

# The mailer
class TaskMailer < ActionMailer::Base
  def assignment_notification
    @task = params[:task]
    mail(to: @task.recipient, subject: "A new task has been assigned to you!")
  end
end

# A simple service that combines them
class TaskAssignment
  def call(task)
    task.save or return false
    TaskMailer.with(task: task).assignment_notification.deliver_now
  end
end
```

This already raises questions:

* Where do I put `TaskAssignment`?
* How do future programmers know to use `TaskAssignment` instead of the `Task` model?

One popular idea is to put `TaskAssignment` in `app/services/task_assignment.rb`. If app/services exists, hopefully it will raise a flag to a moderately experienced Rails developer to look there when they want to do something.

In my experience, this starts to break down in a large application when you have hundreds of models and many more services. How do you name your services? Do you have a service for every single operation on every single model? (Hint: NO) Are you allowed to keep using a plain old `Task` if you don't need to send an email?

A small improvement might be to put the service in a module. In this case, `Task` itself could be a namespace:

```ruby
# app/services/task/assign.rb
class Task::Assign
  def call(task)
    # ...
  end
end
```

I really like how Trailblazer suggests creating `app/concepts/task/operation/assign.rb`. This makes it quite clear that a Task is a *concept*, not just a database table. If the concepts directory exists, you know what you are dealing with, and its structure allows you to grow to a large number of models and services (or "operations", as Trailblazer calls them).

**Recommendation if you don't need very many service objects:** Just add a method to your model. Nobody is going to be surprised when a model has business logic methods.

```ruby
class Task < ActiveRecord::Base
  def save_and_send_notifications
    save or return false
    TaskMailer.with(task: task).assignment_notification.deliver_now
  end
end
```

**Summary: Pick a convention, document it, and stick to it.** The bad news is there is no one right answer. If you feel the siren song of service objects, consider making an entire service layer and keeping business logic out of your models. It's cumbersome, but clear. That way, you always look at the same file to see what happens when the user clicks something. Otherwise, consider whether just adding a method to your model is "good enough".


## 2. Handle failure cases clearly and concisely.

The ability to clearly and concisely handle complex failure cases within database transactions is where service objects have the potential to really shine compared to ActiveRecord. Don't get me wrong, [`ActiveModel::Validations`][am-validations] is great for simple use cases, but it is absolutely terrible if you need to save multiple models at once. [`#accepts_nested_attributes_for` and `#fields_for`][nested-attrs] are hilariously difficult to work with and &mdash; having become an expert at them over many years of frustration &mdash; I recommend pretending they don't exist (unless you are desperate).


```ruby
class TasksController
  def create
    @task = 

[previous]: {{ site.baseurl }}{% post_url 2019-03-07-the-problem-with-service-objects-part-1 %}
[railsguide]: https://guides.rubyonrails.org/
[trailblazer]: http://trailblazer.to/
[dry-txn]: https://dry-rb.org/gems/dry-transaction/
[call-example]: https://naturaily.com/blog/ruby-on-rails-design-patterns
[single-responsibility]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[am-validations]: https://api.rubyonrails.org/classes/ActiveModel/Validations.html
[nested-attrs]: https://api.rubyonrails.org/classes/ActiveRecord/NestedAttributes/ClassMethods.html
