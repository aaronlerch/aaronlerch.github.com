---
layout: post
title: "App Extensibility, Follow Up"
date: 2011-10-09 22:35
categories:
- blog
---

My [last
post]({% post_url 2011-09-08-app-extensibility-in-ruby %})
about introducing an extensibility point
got some good feedback. I realized two things:

1. Following the pattern of rails generators was *huge* overkill
2. I wasn't thinking correctly about the problem to begin with

Rails generators need to support a lot of things. For example, they
need to process command line arguments, consume or produce templates,
and update
existing files in a non-destructive way. The potential complexity in
generators is handled by the complexity in the extensility model
implemented in rails. Arbitrarily duplicating this complexity (magic
class names,
inheritance, etc) is just wrong. Which, of course, is why I
said:

> OK, I’m doing something wrong, I just know it

If I was going to continue down this road (I'm not, see below) I would
take a cue from something like [Sinatra](http://www.sinatrarb.com/) and
take advantage
of global methods to reduce/remove requirements on the configuration
code.

{% highlight ruby %}
configure do |host|
  # do something with 'host' to configure your app
end
{% endhighlight %}

But that's all irrelevent. Jeff Lindsay set me straight in a comment on my previous post.

> The hooks that most SCM use, like post-commit or post-receive,
> are based on running shell scripts. This is my favorite approach
> so far because it's not language specific.

DUH. I love those things that I look back and think "of course."

I was thinking about this as a ruby problem, because localtunnel's
client is written in ruby. But it's not a ruby problem. Localtunnel is a
utility. I can write hooks for Git in anything I want, it doesn't matter
what language Git is written in. The same is true for localtunnel.

Taking this approach *greatly* simplifies the solution. And the simpler
solution is always always always the better solution. In this
case, we can add a quick check followed by a system call to enable users
to provide a shell hook to configure whatever system(s) they want:

{% highlight ruby %}
SHELL_HOOK_FILE = "./.localtunnel_callback"

...

if File.exists?(File.expand_path(SHELL_HOOK_FILE))
  system "#{SHELL_HOOK_FILE} ""#{tunnel['host']}""" if
File.exists?(File.expand_path(SHELL_HOOK_FILE))
  if !$?.success?
    puts "   An error occurred executing the callback hook
#{SHELL_HOOK_FILE}"
    puts "   (Make sure it is executable)"
  end
end
{% endhighlight %}

Bam. Localtunnel now provides an extensibility point that can be
implemented in any language. Well, it will when/if my [pull
request](https://github.com/progrium/localtunnel/pull/27) is
accepted. :)

Thanks, Jeff!
