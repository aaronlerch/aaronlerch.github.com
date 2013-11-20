---
layout: post
title: "Spelunking"
date: 2013-11-20 14:57
categories:
  - blog
---

> Writing software is 10% inspiration, 90% grepping through source from
> github trying to find how the broken internals of a third party
> dependency work.
>
> \- me

It seems like a joke, like something that just _couldn't possibly be
true_. But in today's world of increasingly
componentized software units, we find ourselves spending a
disproportionate amount of time
figuring out how and why some 3rd party library works, or doesn't work.

### Can you imagine what a closed-source compiled world would be like?

\*shudder\*<br />
Thank goodness for OSS.

Today's "OSS cave" was [foreman](https://github.com/ddollar/foreman/).
Foreman is great, but we are using it to export
[upstart](http://upstart.ubuntu.com/) configuration files and the config
files it created
result in orphaned processes after a new deploy.

As it turns out this is a long-standing known issue:
[#97](https://github.com/ddollar/foreman/issues/97). In the issue,
[jarthod](https://github.com/jarthod) has a
[resolution](https://github.com/ddollar/foreman/issues/97#issuecomment-19608866)
which involves using a customized template to generate a working 
upstart configuration file. What's non-obvious is how to incorporate
this custom template.

### Customizing foreman upstart templates

The foreman upstart
[exporter](https://github.com/ddollar/foreman/blob/master/lib/foreman/export/upstart.rb)
uses three templates: `upstart/master.conf.erb`,
`upstart/process_master.conf.erb`, and `upstart/process.conf.erb`.
jarthod offered a replacement for `process.conf.erb`.

After looking through the command line options and the source for the
upstart exporter, the answer was fairly easy.

* Create `process.conf.erb` in any folder you like
* Invoke foreman with the `-t` option to specify a custom template
  directory: `foreman export upstart /etc/init -t path/to/templates/`

The [documentation
says](https://github.com/ddollar/foreman/wiki/Custom-exporters) that the foreman exporter [base
class](https://github.com/ddollar/foreman/blob/master/lib/foreman/export/base.rb)
uses the method `export_template` which looks for templates in 3
locations: the `-t` command-line provided option,
`~/.foreman/templates`, and finally the templates shipped with the gem
itself.

The trick is that if you look at the source for the default upstart
exporter, you might be tempted to put the templates in an `upstart/`
subdirectory, but templates from the `-t` command-line option are looked
for just in the root of that directory.

## Here's the frustrating part

After writing all that down, it seems STUPIDLY SIMPLE. Because it is. But yet it took
longer than I wish to find and/or figure all that out. Ugh.
