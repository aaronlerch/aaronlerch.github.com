---
layout: post
title: "Sinatra, Bundler, and the global namespace"
date: 2013-10-26 23:04
categories:
  - blog
---

Either my google-fu is waning or I ran into an issue most people
haven't so I want to quickly document it.

The scenario
------------

A Rails 4 app mounting a modular-style Sinatra app to service a
particular route, kind of like this.

routes.rb
{% highlight ruby %}
MyApp::Application.routes.draw do
  mount MySubApp => 'subapp'
end
{% endhighlight %}

my_sub_app.rb
{% highlight ruby %}
require 'sinatra/base'

class MySubApp < Sinatra::Base
  # stuff
end
{% endhighlight %}

The problem
-----------

The Gemfile looks like what you'd expect if you just added gems
'normally'.

{% highlight ruby %}
source 'https://rubygems.org'

# Bundle edge Rails instead: gem 'rails', github: 'rails/rails'
gem 'rails', '4.0.0'

# yadda yadda yadda

gem "sinatra", "~> 1.4.3"
gem "sinatra-contrib", "~> 1.4.1"
{% endhighlight %}

The problem comes because Bundler is really helpful and requires the
gems for you automatically, using the gem name by default. Unfortunately
this means that Bundler will end up running `require 'sinatra'` (and
`require 'sinatra-contrib'`) and when
used this way Sinatra adds its DSL to the global namespace.

That means that when you run rake, for example, you could see odd error
messages like "rake aborted! undefined method 'task' for Sinatra::Application:Class"

In my case, sinatra-contrib was intercepting rake's DSL and obviously
failing.

The solution
------------

The solution is to fix the Gemfile to instruct Bundler how to correctly
require the sinatra gems.

{% highlight ruby %}
gem "sinatra", "~> 1.4.3", require: 'sinatra/base'
gem "sinatra-contrib", "~> 1.4.1", require: false
{% endhighlight %}

This change results in Sinatra being required correctly for modular app use,
and turns off automatic requiring for sinatra-contrib so you can
require just the bits you need at the place you need them, since
sinatra-contrib is just a set of helpful modules.


It's not complex or difficult, but the error message (and backtrace) was
a little non-obvious.
