---
layout: post
title:  Rack and nothing else
date:   2016-03-01 21:38:44
categories: dev
image: /assets/article_images/2016-03-01-rack-and-nothing-else/banner.png
comments: true
---
The main objective of this blogpost will be to build a Ruby werbserver with Rack as the only gem of our Gemfile and see how far we can go in our app complexity.

The setup
---

First of all, we'll need at the root of our project a `Gemfile` file with those few lines:

```ruby
source 'https://rubygems.org'

gem 'rack'
```

Keep in mind that a `.ruby-version` file would be really appreciated by your Ruby version manager too, just type in there: `2.2.3`, or any other Ruby version of your choice.

Now just run


$ bundle install
```

Our project, at this step:

```
/my_project
 - .ruby_version
 - Gemfile
 - Gemfile.lock
```

Step 1: Hello world!
---

As [Rack website](http://rack.github.io/) explains, at the root of our project, a `config.ru` file like this:

```ruby
run Proc.new { |env| ['200', {'Content-Type' => 'text/html'}, ['Hello world!']] }
```

can run in a Rack server with a `bundle exec rackup config.ru` command in your terminal.  A quick check at `localhost:8080` in your browser will show you this :

< screenshot >

It works !

Now let's see in details the content of the array our `Proc` instance gave to Rack:

```ruby
['200', {'Content-Type' => 'text/html'}, ['Hello world!']]
```

The first item is the HTTP code sent by our server, the second are the headers sent for the response and the last is the body of the response.
