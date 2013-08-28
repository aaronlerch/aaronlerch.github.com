---
layout: post
title: "Preventing Rack::Session on a Per-Request Basis"
date: 2012-01-14 22:55
categories:
- blog
---

This is admittedly a rare need, but the other day I found myself needing
to conditionally enable and disable sessions in Sinatra on a per-request
basis. Sinatra uses the Rack::Session middleware to manage sessions. I
searched around for a while but couldn’t find out how to do it. There
are options for the session that can be set per-request, like this:

{% highlight ruby %}
get '/' do
  request.session_options[:renew] = true
end
{% endhighlight %}

The
[documentation](http://rack.rubyforge.org/doc/Rack/Session/Abstract/ID.html) shows a :defer option that seems to do what I want,
but the terminology didn’t make it abundantly clear whether it did or
not.

> \:defer will not set a cookie in the response.

Actually that does sound like what I want, sorta, but mentally I was
thinking “turn off” or “disabling” and not “deferring the setting of a
cookie.”

With most ruby projects, or for that matter, any interpreted language
that isn’t compiled , I’ve found that I end up spending a decent amount
of time reading the code from my dependencies. Some unscientific polling
of ruby devs I know showed that to be a pretty standard practice.

Doing a little digging through the Rack::Session middleware showed an
option that didn’t appear in the documentation:

> \:skip will not a set a cookie in the response nor update the session state

Now that matches the mental model of what I was looking for, and the
behavior as well. Turns out it’s not in the docs because it was added
[recently](https://github.com/rack/rack/pull/277) from a Rails core team member to support the new asset pipeline
in Rails 3.1, and the docs haven’t been updated yet.

So if, on a per-request basis, you want to disable using sessions
entirely, simply do:

{% highlight ruby %}
request.session_options[:skip] = true unless use_session?
{% endhighlight %}

(where “use_session?” is your method that figures out whether or not to
use sessions)

What’s the diff between :defer and :skip, you ask? Well the answer is in
the pull request description

> This will not send a cookie back nor change the session state.
>
> The :defer option did not send the cookie back but did change the
> session state in the backend.
> 
> This is useful for assets requests that still go through the rack stack
> but do not want to cause any change in the session (for example
> accidentally expiring flash messages).
