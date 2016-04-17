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

Let's add a `layout.erb` view to our project in a `views` folder:

```erb
<p>It's <%= Time.now.strftime('%H:%M') %>!</p>
```

<br/>
Since we're lucky, the `ERB` class is
[part of ruby Standard Lib](http://ruby-doc.org/stdlib-2.2.3/libdoc/erb/rdoc/ERB.html){:target="_blank"},
so no need for another gem!

Reading erb content isn't that hard in fact. In ruby, reading a file can be written like this:

```ruby
file_path = Dir.pwd + '/views/index.erb'
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
  read_erb_file(Dir.pwd + '/views/index.erb')
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
    - index.erb
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
Let's add more views to our project, I'll just put the files name in their content at this moment
in a dedicated `posts` folder:

```
/my_project
  /views
    - index.erb
  /posts
    - foo.erb
    - bar.erb
 - .ruby_version
 - config.ru
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
  { '/'    => read_erb_file(Dir.pwd + '/views/index.erb'),
    '/foo' => read_erb_file(Dir.pwd + '/posts/foo.erb'),
    '/bar' => read_erb_file(Dir.pwd + '/posts/bar.erb') }
end

run Rack::URLMap.new(routes)
```
<br/>
Seeing this should scream at you that we should do it dynamically. Doing so would not necessitate
to update the code to build its route after its addition to the project. Which would be a considerable
waste of time.

```ruby
ROOT ||= Dir.pwd
ERB_FILES ||= Dir[ROOT + '/posts/**/*.erb']

def read_erb_file(file_path)
  file_content = File.read(File.expand_path(file_path))

  erb_load = ERB.new(file_content).result(binding)

  Proc.new { |env| ['200', {'Content-Type' => 'text/html'}, [erb_load]] }
end

def home
  { '/' => read_erb_file(ROOT + '/views/index.erb') }
end

def relative_file_path(file)
  "/#{file.match(/posts\/(.*).erb/).captures.first}"
end

def routes
  ERB_FILES.each_with_object(home) do |file, response|
    response.merge!({ relative_file_path(file) => read_erb_file(file) })
    response
  end
end

run Rack::URLMap.new(routes)
```
<br/>
We added few constants in order to make it DRYish.
As you can see, the only specific case here is the homepage which is separated in its own method.
The rest of the routes will be mapped to an url matching their path in the `posts` folder, thanks to
the `relative_file_path` method.

This way, we have our homepage on `localhost:9292`, and the two other views are accessible at `/foo`
and `/bar`! But you may wonder why I kept the homepage separated. The next step will make more sense
since we're going to build a layout system!
<hr/>
Step 4: Let's create a layout system!
---
<br/>
First things first, let's add a `layout` file in our views:

```
/my_project
  /views
    - layout.erb
    - index.erb
  /posts
    - foo.erb
    - bar.erb
 - .ruby_version
 - config.ru
 - Gemfile
 - Gemfile.lock
```
<br/>
Now things are getting serious since, as you may have already found, a layout system would basically
rely on the fact that we're going to insert the content of an erb file... in another erb file. It may
sound complicated but it can be summed up quite simply.

```ruby
LAYOUT ||= ROOT + '/views/layout.erb'

def erb(file)
  params = render_partial(file)

  ERB.new(File.read(LAYOUT)).result(params.instance_eval { binding })
end

def render_partial(file)
  return OpenStruct.new(content: nil) if file.nil?

  path = File.expand_path(file)

  OpenStruct.new(content: ERB.new(File.read(path)).result(binding))
end

def load_erb(file)
  render = erb(file)

  Proc.new { |env| ['200', {'Content-Type' => 'text/html'}, [render]] }
end
```
<br/>
For this case, we have to split the rendering logic. The `render_partial` method has been updated, it will
be in charge of loading the post content file and its `.erb` content. We eventually add a guard condition
in its first line in case of a user calling a post that does not exist, preventing a crash of the application.

The `erb` method is really interesting, it'll handle the merge of the two ERBs.
First things first, as you can see in its last line, it'll load the layout file. Then, in this content, we
embed an `instance_eval` of the partial rendering and return the whole result as a `binding`, same as before.
This time the `instance_eval` contains an `OpenStruct` with a key named `content`, this key has the whole
partial content.

A view of the `layout.erb` file may make this clearer as how you'll load this content in your layout:

```
<html>
    <head><head>
    <body>
        <h1>This is the layout.</h1>
        <%= content %>
    </body>
</html>
```
<br/>
As I've said before, even if it sounds complicated, keep in mind that it's just embedding an `erb` rendering
into another. You can now load any post file in your layout and add new posts on the fly with no sweat!

The next step will be dedicated to strenghten our application by preventing unexpected behaviours.
<hr/>
Step 5: Prevent unexpected behaviours and massive cleanup!
---
<br/>
Our main issue is that all of our program routes are defined at its launch. Meaning that, with our actual code,
you can't really redirect a user to a "404 not found" page if this user tries to load an unexisting post. At
this point, you'll only load an empty layout unfortunately: the content of the partial loading being `nil`.

Another issue is that if you check you app logs, you'll see that your browser requests your favicon to display it
and it raises a 404 too.

So, how can we make this routes system more dynamic and more intelligent? How can we send images files like a favicon?
First of all, let's create a new view called `404.erb` in our `views` folder and a `favicon.ico` file of your choice
at the root of the project.

Since it'll be a massive rework, let's move our code to a proper `app.rb` file and update the `config.ru`. We'll add
some namespacing too! (From now on, I'll name my app `Garnet`)

```
/garnet
  /views
    - layout.erb
    - index.erb
    - 404.erb
  /posts
    - foo.erb
    - bar.erb
 - .ruby_version
 - app.rb
 - config.ru
 - favicon.ico
 - Gemfile
 - Gemfile.lock
```
<br/>
The `app.rb` has been reworked a lot:

```ruby
require 'rack'
require 'erb'
require 'ostruct'

module Garnet
  class Application

    ROOT ||= File.expand_path(Dir.pwd)

    SYSTEM_FILES ||= { 'layout'    => ROOT + '/views/layout.erb',
                       'favicon'   => ROOT + '/favicon.ico',
                       'homepage'  => ROOT + '/views/index.erb',
                       'not_found' => ROOT + '/views/404.erb' }

    HEADERS ||= { 'html' => {'Content-Type' => 'text/html' },
                  'favicon' => { 'Content-Type' => 'image/x-icon',
                                 'Content-Length' => File.read(SYSTEM_FILES['favicon']).bytesize.to_s } }

    def erb(file)
      params = build_partial(file)
      layout = File.read(SYSTEM_FILES['layout'])

      ERB.new(layout).result(params.instance_eval { binding })
    end

    def build_partial(file)
      content = ERB.new(File.read(File.expand_path(file))).result(binding)

      OpenStruct.new(content: content)
    end

    def render_view(file_name)
      file = "#{ROOT}/posts/#{file_name}.erb"

      return not_found unless File.exist?(file)

      ['200', HEADERS['html'], [erb(file)]]
    end

    def favicon
      ['200', HEADERS['favicon'], [File.read(SYSTEM_FILES['favicon'])]]
    end

    def homepage
      ['200', HEADERS['html'], [erb(SYSTEM_FILES['homepage'])]]
    end

    def not_found
      ['404', HEADERS['html'], [erb(SYSTEM_FILES['not_found'])]]
    end

    def call(env)
      endpoint_called = Rack::Request.new(env).path

      return favicon  if endpoint_called == '/favicon.ico'
      return homepage if endpoint_called == '/'

      render_view(endpoint_called)
    end
  end
end
```
<br/>
The real focus here is on the `call` method, It'll get each path request and act
as a proper router. There's two guard conditions: the first is related to the favicon
that'll be requested at each call. Doing so allow us to send it as fast as possble in
the whole app process. The second is related to the homepage that can't match our
dynamic route building unfortunately.

Since `call` is in fact a method call by `Rack::Handler::WEBrick` triggered automatically
which renders views for us, we'll cheat a little an use its magic by simply implementing it.
This way we still rely only on `Rack` while doing fancy things.

The rest of the rework, as you can see, is just a cleanup in order to keep our project
as DRYish as possible.

`config.ru` now calls a new instance of Garnet, as expected, binded on the port 3000 now
(which is a personal preference and not really neccessary).

```ruby
require './app'

app = Garnet::Application.new

Rack::Handler::WEBrick.run(app, Port: 3000)
```
<hr/>

Last step: Add some CSS!
---
<br/>
One last thing: loading asset files! Since dealing with blank views is not quite appealing, let's
add some style and update our code base to load asset files from a, `asset` folder at the root of
our app.

Just update `app.rb` to match this, it's as simple as it sounds:

```ruby
def asset_file(name)
  file = [ROOT, name].join

  return not_found unless File.exist?(file)

  ['200', HEADERS['html'], [File.read(file)]]
end

def call(env)
  endpoint_called = Rack::Request.new(env).path

  return favicon if endpoint_called == '/favicon.ico'
  return homepage if endpoint_called == '/'
  return asset_file(endpoint_called) if endpoint_called =~ /\/assets/

  render_view(endpoint_called)
end
```
<br/>
This way, any relative link to a stylesheet can work properly and return all the css you need!
<br/>
<hr/>
There you go! You can now have a little blog working with only one gem in your Gemfile, as promised!
I hope you found this as interesting as it was funny to do for me and if you want to contribute you
can find the source code on [GitHub](https://github.com/Yinfei/garnet){:target="_blank"}.
Thank you for reading, I hope you learnt some things!
