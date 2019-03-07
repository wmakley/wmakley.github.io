---
layout: post
title: "The Problem With Service Objects: Part 1"
date: "2019-03-07"
categories: rails
---
Since I started working on Rails applications, I have been on an architectural journey to find the ideal service object.

## The Problem

Skip this section if you are familiar with the pitfalls of ActiveRecord.

Almost any monolith eventually grows in complexity to the point where any given user interaction could affect numerous tables and have a large number of "side effects" on the application state, or really what I call **"desired ancillary effects"**. If you naively use ActiveRecord to handle all form submissions, using callbacks and "concerns" organically as needed, you eventually run into some well-known issues.

### Callbacks can create "order of operations" problems when dealing with multiple related models.

Say you have these models:

```ruby
class Parent < ApplicationRecord
  has_many :children
end

class Child < ApplicationRecord
  belongs_to :parent
end
```

And then you have some forms to create parents, and then controllers and forms to view each parent and create children for them:

* app/controllers/parents_controller.rb
* app/controllers/parents/children_controller.rb
* app/views/parents/...
* app/views/parents/children/...

So far so good! The problem comes when things start getting complicated.

#### New requirement 1: Update the parent's updated_at timestamp when a child changes.

A trivial example that I've actually had happen. (Let's say that fetching the most recently-updated child is not an option for performance/complexity reasons, and we need to update the parent.)

**Obvious callback solution:**

```ruby
class Child < ApplicationRecord
  belongs_to :parent

  after_save :update_parent_timestamp

  def update_parent_timestamp
    parent.touch!
  end
end
```

So far so good. (Although it already has a subtle bug... can you see it?)

Uh oh, now the customer is complaining that it takes too long to add children!

#### New requirement 2: Allow user to add infinite children directly on the **parent** form.

Forgetting about the form building nightmare (why does Rails not help more with this?), lets say we just do a "bad thing" and put all the logic in the controller:

```ruby
class ParentsController < ApplicationController
  def create
    @parent = Parent.new(parent_params)
    child_names_from_form.each do |child_name|
      @parent.children.build(name: child_name)
    end

    if @parent.save
      redirect_to # somewhere
    else
      render action: :new
    end
  end

  def parent_params
    params.require(:parent).permit(:name)
  end

  def child_names_from_form
    [ "Fred", "Wilma" ]
  end
end
```

Uh oh, all we did was do things the most obvious way, and now our form is completely broken! Here are some of the bugs:

1. If a child is invalid, the form will re-render but not display an error message.
2. The parent will be "touched" by every child's callback.

To solve the first bug, we could always use `accepts_nested_attributes_for` and `fields_for`, but we'll ignore that magical mess for now. The real issue is bug #2. Consider an application with thousands of "children", or non-trivial work in the callback,or - **even worse** -- an application where the parent runs a callback after it has been "touched" by its children: This could be a *massive* amount of wasted trips to the database.

There isn't an easy fix. There is no possible way to check inside the callback if the parent was *"just saved"*, and turning *off* the callback during the "create" action is just an even bigger mess. **So we did something completely wrong architecturally, but ActiveRecord callbacks made it seem right at first.**

## Taking a step back

Where did we go wrong? We missed the fact that the **Parent** *owns* its **Child** objects, and created a `ChildrenController` class that reached in and messed with them, bypassing the parent, treating the child as "owning" the parent. Because `ChildrenController` existed, adding the callback to `Child` seemed to make total sense! And it worked just fine until we had to implement requirement #2.

