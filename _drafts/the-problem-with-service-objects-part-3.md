---
layout: post
title: "The Problem With Service Objects Part 3: \"Why Service Objects\""
date: "2019-03-20"
categories: rails
---
In the [previous post][previous], we went over various ActiveRecord solutions to orchestrating multiple models, and why they all fail to please. In this post I will show how and why we arrive at service objects.

## What is a web application?

**A web application is a bundle of state that can be mutated by HTTP requests.**


[railsguide]: https://guides.rubyonrails.org/
[previous]: {{ site.baseurl }}{% post_url 2019-03-08-the-problem-with-service-objects-part-2 %}
