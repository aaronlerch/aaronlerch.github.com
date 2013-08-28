---
layout: post
title: "App Extensibility in Ruby"
date: 2011-09-08 18:13
categories:
- blog
---

I ran into an interesting problem recently. There is a utility called [localtunnel](http://progrium.com/localtunnel/) which connects a public URL to a local port. Extremely useful when it comes to developing an app that leverages services that expose web hooks.

The problem is that the public URL is only valid while the script is running. Which means every time you start it, you potentially need to go update the web hook URL in whatever service you're using. For example, with [Twilio](http://www.twilio.com/) you configure an app that is available at a particular URL. With GitHub you can add [post-receive hooks](http://help.github.com/post-receive-hooks/).

What needs to happen is that localtunnel needs to give you a chance to run some custom code after it registers and connects up ports, but before it sits and waits.

It needs an extension point.

[One fork](https://github.com/snay2/localtunnel/) seeks to do this by having localtunnel call a web hook. It works, even if it feels a bit ironic. ;) But that involves maintaining another service simply to configure your first service, which is less than ideal. I took a different approach and I'd love to get feedback here, since I'm still so new to this.

Most stuff on the web about extending ruby has to do with class design, or monkeypatching, or something else that assumes all the code is already loaded and executing. In this case, there's an existing app (not framework) and I want to have it load *my* code dynamically at runtime. In my code, I want to configure whatever service I need to.

Being a n00b, I'm not the most widely read yet, so I looked at an example of something that I already knew exposed an extension point in a similar way: [rails generators](http://guides.rubyonrails.org/generators.html).

[You can see my code here](https://github.com/progrium/localtunnel/pull/23) and the [updated readme
here](https://github.com/aaronlerch/localtunnel/), but here's how it works.

Specify the config to run
-------------------------

This part was easy, I just extended the existing command line processing to add a "-c NAME" argument specifying which configuration you want to run. For example:

{% highlight bash %}
$ localtunnel -c twilio 9292
{% endhighlight %}

will create a public URL connecting to local port 9292, and will look
for an auto configuration implementation named "twilio" and execute it.

Discovery via magic names, I mean, convention
---------------------------------------------

This is a nice, but sometimes frustrating thing about Rails. There's
lots of stuff that magically happens if you organize or name your code a
certain way.

In this case, to make it easy to specify via the command line, I opted
for a similar "magic name" approach. Name your file
foo_auto_config.rb and when you specify "-c foo" I'll look for your
file under a 'localtunnel' subdir. This is a way to try and avoid
loading a lot of code unnecessarily.

{% highlight ruby %}
https://github.com/aaronlerch/localtunnel/blob/master/lib/localtunnel/autoconfig.rb
def self.lookup(name)
  including_current = $LOAD_PATH.dup
  including_current << '.'
  including_current.each do |base|
    Dir[File.join(base, "localtunnel", "#{name}_auto_config.rb")].each do |path|
      begin
        require path
      rescue Exception => e
        puts "   [Warning] Could not load autoconfig #{path.inspect}. Error: #{e.message}.\n#{e.backtrace.join("\n")}"
      end
    end
  end
end
{% endhighlight %}

This has a few issues with it. Because this is running as a script, if
you're using bundler, the $LOAD_PATH won't include all the gems unless
Bundler.require has been called. So looking in the load path for auto
configuration files is probably pointless. Not that gems would include a
custom auto configuration anyway. I should probably remove this.
(Thoughts?) Secondly, I manually added
the local directory because ruby 1.9.2 [took it out of $LOAD_PATH by
default](http://stackoverflow.com/questions/2900370/why-does-ruby-1-9-2-remove-from-load-path-and-whats-the-alternative)
due to security reasons.
And if you consider the use-case, this is typically executed from within
~/MyApp and not randomly throughout the directory structure. Again,
similar to rails generators, you run it from the root of your
application directory.

Discovery via class name + base class
-------------------------------------

Now that we've figured out which files to load, we still don't know what
code to call. We need to be able to pass in parameters such as the new
URL that is reserved, so we probably want a method to call and not just
loading a file.

Ruby gives us a feature that allows you to know when a new base class is
created: the
[inherited](http://www.ruby-doc.org/core/classes/Class.html#M000177)
method.
It gets called when a new subclass is created.

I created a base class, LocalTunnel::AutoConfig::Base, which keeps track
of all subclasses:

{% highlight ruby %}
https://github.com/aaronlerch/localtunnel/blob/master/lib/localtunnel/autoconfig.rb
class Base
  def self.inherited(base)
    super

    if base.name && base.name !~ /Base$/
      LocalTunnel::AutoConfig.subclasses << base
    end
  end
end
{% endhighlight %}

When someone specifies "-c foo" on the command line, I look for any
subclass of LocalTunnel::AutoConfig::Base that is named FooAutoConfig
and has a method called "configure". If I find that, then that's the
auto configuration code that will be called.

{% highlight ruby %}
https://github.com/aaronlerch/localtunnel/blob/master/lib/localtunnel/autoconfig.rb
def self.find(name)
  lookup(name)

  names = Hash[subclasses.map { |klass| [autoconfig_name(klass).downcase, klass] }]
  klass = names[name]
  return nil if klass.nil?

  configurator = klass.new
  if configurator.respond_to? :configure
    configurator
  else
    nil
  end
end
{% endhighlight %}

Then I can call that method on an instance of the matching class, and
pass it the new URL.

{% highlight ruby %}
https://github.com/aaronlerch/localtunnel/blob/master/lib/localtunnel/tunnel.rb
if !@autoconfig.nil?
  configurator = LocalTunnel::AutoConfig.find(@autoconfig)
  if configurator
    configurator.configure(tunnel['host'])
  else
    puts "   [Warning] Unable to find an automatic configuration plugin for '#{@autoconfig}'"
  end
end
{% endhighlight %}

Do your custom configuration
----------------------------

The configuration code can then run and do whatever it wants. Usually
it's
specific per app. Here's an example of me using this to configure
Twilio.

{% highlight ruby %}
require 'rubygems'
require 'localtunnel/autoconfig'
require 'twilio-ruby'
require 'uri'

class TwilioAutoConfig < LocalTunnel::AutoConfig::Base

  TWILIO_ACCOUNT_SID = # my account Sid
  TWILIO_AUTH_TOKEN = ENV['TWILIO_AUTH_TOKEN']
  TWILIO_APP_SID = ENV['TWILIO_APP_SID']

  def configure(host)
    # set up a client to talk to the Twilio REST API
    #     client = Twilio::REST::Client.new(TWILIO_ACCOUNT_SID,
          TWILIO_AUTH_TOKEN)
    app = client.account.applications.get(TWILIO_APP_SID)

    # Grab the current voice_url and status_callback and swap out the
      host
    voice = URI.parse(app.voice_url)
    voice.host = host
    status_callback = URI.parse(app.status_callback)
    status_callback.host = host

    app.update({:voice_url => voice.to_s, :status_callback =>
status_callback.to_s})
    puts "   Configured twilio app #{app.friendly_name} for new host
#{host}"
  end
end
{% endhighlight %}

OK, I'm doing something wrong, I just know it
---------------------------------------------

I am sure I either overcomplicated this, or just plain did it wrong.
Some thoughts I have (in hindsight now) are that perhaps I could've
exposed a global method for the autoconfig code to call. Then, instead
of a base class and magic class name, I'd just load the specified file
(magic name is still nice) and in that file I could call "host" to get
the new host name.

Leave a comment, create a gist, [fork my
fork](https://github.com/aaronlerch/localtunnel/),
or [tweet at me](http://twitter.com/aaronlerch/), but somehow let me
know how I could
make this better.
