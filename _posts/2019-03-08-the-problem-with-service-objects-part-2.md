---
layout: post
title: "The Problem With Service Objects Part 2: \"ActiveRecord Solutions\""
date: "2019-03-07"
categories: rails
---

In the [previous post][previous], we discussed a common architectural pitfall in Rails apps: creating "cycles" and loops using callbacks to make related records update each other. In this post I will discuss some possible solutions, staying inside the bounds of things you learn from the [Rails Guide][railsguide].

## Idea 1: Move the callback to ChildrenController

```ruby
class ChildrenController < ApplicationController
  def update
    
  end
end
```

[railsguide]: https://guides.rubyonrails.org/
[previous]: {{ site.baseurl }}{% post_url 2019-03-07-the-problem-with-service-objects-part-1 %}
