---
layout: post
title:  Rack and nothing else
date:   2016-03-01 21:38:44
categories: dev
image: /assets/article_images/2016-03-01-rack-and-nothing-else/banner.png
comments: true
---
The main objective of this blogpost will be to build a Ruby werbserver with Rack
as the only gem of our Gemfile and see how far we can go in our app complexity.
<hr/>
The setup
---
<br/>
I strongly recommend having a Ruby Version manager installed, I personnaly use
[Rbenv](https://github.com/rbenv/rbenv){:target="_blank"}.

First of all, we'll need at the root of our project a `Gemfile` file with those
few lines:

```ruby
source 'https://rubygems.org'

gem 'rack'
```
<br/>
Keep in mind that a `.ruby-version` file would be really appreciated by your
Ruby version manager too, just type in there: `2.2.3`, or any other Ruby
version of your choice.

Now just run the `bundle install` command in your terminal. Our project, at
this step, should look like this:

```
/my_project
 - .ruby_version
 - Gemfile
 - Gemfile.lock
```
<hr/>

Step 1: Hello world!
---
<br/>
As [Rack website](http://rack.github.io/) explains, at the root of our project,
a `config.ru` file like this:

```ruby
require 'rack'

run Proc.new { |env| ['200', {'Content-Type' => 'text/html'}, ['Hello world!']] }
```
<br>
can run in a Rack server with a `bundle exec rackup config.ru` command in your
terminal.  A quick check at `localhost:9292` (9292 being rack default port) in
your browser will show you this :

![It works!](/assets/article_images/2016-03-01-rack-and-nothing-else/hello_world.png)

Now let's see in details the content of the array our
[Proc](http://ruby-doc.org/core-2.2.3/Proc.html){:target="_blank"} instance gave
to Rack:

```ruby
['200', {'Content-Type' => 'text/html'}, ['Hello world!']]
```
<br/>
The first item is the HTTP code sent by our server, the second is the headers
sent by the server for the response and the last is the body of the response
itself: in this case plain text.

The next step will be to read an `.erb` file! Our project at this point:

```
/my_project
 - .ruby_version
 - config.ru
 - Gemfile
 - Gemfile.lock
```
<hr/>

Step 2: Let's read some .erb !
---
<br/>
While we're working with ruby, it would be cool to load erb views. This way we
could use ruby code in it!

Let's add a `layout.erb` view to our project in a `views`:

```erb
<p>It's <%= Time.now.strftime('%H:%M') %>!</p>
```

<br/>
Since we're lucky, the `ERB` class is
[part of ruby Standard Lib](http://ruby-doc.org/stdlib-2.2.3/libdoc/erb/rdoc/ERB.html){:target="_blank"},
so no need for another gem!

Reading erb content isn't that hard in fact. In ruby, reading a file can be written like this:

```ruby
file_path = Dir.pwd + '/views/layout.erb'
file_content = File.read(File.expand_path(file_path))
```
<br/>
Render its erb content afterwards would just be something like this:

```ruby
ERB.new(file_content).result(binding)
```
<br/>
In this case `binding` will be the result of the erb interpretation done by the `result` method call.

Now let's extract the body response in its own method and implement all of this in our `config.ru` :

```ruby
require 'rack'
require 'erb'

def read_erb_file(file_path)
  file_content = File.read(File.expand_path(file_path))

  erb_load = ERB.new(file_content).result(binding)

  Proc.new { |env| ['200', {'Content-Type' => 'text/html'}, [erb_load]] }
end

def body
  read_erb_file(Dir.pwd + '/views/layout.erb')
end

run body
```
<br/>
A `bundle exec rackup config.ru` command and a look at `localhost:9292` will show you this :

![We load ERB files!](/assets/article_images/2016-03-01-rack-and-nothing-else/erb_loaded.png)

Our project at this point is getting bigger:

```
/my_project
  /views
    - layout.erb
 - .ruby_version
 - config.ru
 - Gemfile
 - Gemfile.lock
```
<br/>
The next step will be to create few more views and dynamically serve them!
<hr/>

Step 3: More routes
---
<br/>
Let's add more file to our project, we'll make a directory named `posts`
with two files in it, I'll just put the files name in their content at this moment :

```
/my_project
  /views
    - layout.erb
  /posts
    - foo.rb
    - bar.rb
 - .ruby_version
 - Gemfile
 - Gemfile.lock
```
<br/>
First of all, if we're to load multiple Procs, we'll need the
[Rack::URLMap](http://www.rubydoc.info/github/rack/rack/Rack/URLMap){:target="_blank"}
class. It would allow us to load a hash of routes linked to Procs. So, let's update
our `config.ru` file to load those additional views this way. The ugly way would
be to simply update the `body` method to declare thoses routes like this :

```ruby
def routes
  { '/'    => read_erb_file(Dir.pwd + '/views/layout.erb'),
    '/foo' => read_erb_file(Dir.pwd + '/posts/foo.erb'),
    '/bar' => read_erb_file(Dir.pwd + '/posts/bar.erb') }
end

run Rack::URLMap.new(routes)
```
<br/>
But we should do it dynamically. This way, adding an erb view would not neccessitate to update the code to
build its route after its addition to the project.

<hr/>
Step 4: Let's create a layout system!
---

<hr/>
Step 5: Prevent unexpected behaviours
---

Our main issue is that all of our program routes are defined at its launch.
