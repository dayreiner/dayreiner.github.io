---
layout: blog/post
title: "Sending Bash and ZSH Commands to Syslog"
date: 2016-02-21 22:07:13
image: '/assets/img/'
description:
tags:
categories:
twitter_text:
---
Sending Bash and ZSH Commands to Syslog
=======================================

 Your bash/zsh history is great if its complete, but it doesn't capture commands across all users, sudo's, root commands etc. In particular with test environments, someone may perform a "one-off" procedure and then months later it needs to be repeated. It would be nice to be able to look up what the user did at the time, and searching through multiple, possibly truncated history files is a pain.

Tools like [typescript](http://man7.org/linux/man-pages/man1/script.1.html) are great if you're actively documenting, but not something you would use all the time in practice and capture more than just a history of your commands. There are third-party tools like [rootsh](https://sourceforge.net/projects/rootsh/) and [Snoopy](https://github.com/a2o/snoopy) that can accomplish this, but third-party tools can be overkill if all you want is a quick reference in a relatively controlled environment. 

If you need an "official" record of all user actions, you should be using process accounting or better-yet [auditd](https://www.digitalocean.com/community/tutorials/understanding-the-linux-auditing-system-on-centos-7) instead (which can be configured to [record all user commands](http://serverfault.com/questions/470755/log-all-commands-run-by-admins-on-production-servers)). This method is just a trivial way to get a searchable record of all user actions on a box for convenience -- **not** for security or auditing purposes. 

## Syslog Setup

Assuming your distribution uses rsyslog, in `/etc/rsyslog.d` create a file called `commands.log` with the following contents:

{% highlight bash %}
local6.*    /var/log/commands.log
{% endhighlight %}

This will log anything going to local6 in to a local file called commands.log. You can adjust this as you want, or send everything to a central syslog server or logstash instance.

## Bash and ZSH Setup

This method takes advantage of the built-in [PROMPT_COMMAND](http://www.tldp.org/HOWTO/Bash-Prompt-HOWTO/x264.html) variable in bash and the [precmd](http://zsh.sourceforge.net/Doc/Release/Functions.html) function in ZSH. Both of these are used to execute something with every command entered in the shell, making them perfect for capturing command input and piping it to syslog.


### Global Bash Profile Setup

Add the following line to the bottom of the global `/etc/bashrc` file on your system:

{% highlight bash %}
# Log commands to syslog for future reference
export PROMPT_COMMAND='RETRN_VAL=$?;logger -p local6.debug "$(whoami) [$$]: $(history 1 | sed "s/^[ ]*[0-9]\+[ ]*//" ) [$RETRN_VAL]"'
{% endhighlight %}
Alternately, you can just add this to the end of your .bashrc if you just want to do this for a single user for some reason.

### Global ZSH Profile Setup

Add the following line to the bottom of the global `/etc/zshrc` file on your system:

{% highlight bash %}
# Log commands to syslog for future reference
precmd() { eval 'RETRN_VAL=$?;logger -p local6.debug "$(whoami) [$$]: $(history | tail -n1 | sed "s/^[ ]*[0-9]\+[ ]*//" ) [$RETRN_VAL]"' }
{% endhighlight %}
Alternately, you can just add this to the end of your .zshrc if you just want to do this for a single user for some reason.

## Log Format

From now on, whenever someone logs into the system using bash or zsh, you'll get a nice log in order of all the commands that were executed on the box and by whom:

{% highlight bash %}
Feb 16 16:12:44 myhost root: jim [16285]: sudo yum update [1]
Feb 16 16:16:32 myhost bob: bob [10033]: docker ps  [0]
Feb 16 16:16:36 myhost satin: satin [10033]: docker ps -a [0]
{% endhighlight %}
