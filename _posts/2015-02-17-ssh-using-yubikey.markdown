---
layout: post
title:  "SSH Using Yubikey NEO"
categories: docs
---

Steps:

* add yubikey udev rules so that user can modify yubikey.
* yk personalize, change mode to support OpenGPG. 0x82
* install gnupg, gnupg-agent, & scdaemon.
* generate gpg key (yubico steps link).
* move keys to neo.
* configure gpg-agent.conf 
* configure scdaemon.conf for logging
* (note that gpg --card-status will return an error when scdaemon is running).
* setup `/etc/X11/Xsession.d/90x11-common_gpg-agent` to startup gpg-agent.
* setup .bashrc to export env variables.
* ssh-add -L 
* ssh to-some-host

* Recovery Tips.
 - key pass phrase vs pin vs admin pin.
 - unblock pin.

{% highlight bash %}
if [ -f $HOME/local/etc/profile/bashrc.sh ] ;
then
  echo "test"
  source $HOME/local/etc/profile/bashrc.sh ;
fi
{% endhighlight %}

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll's GitHub repo][jekyll-gh].

[jekyll-gh]: https://github.com/mojombo/jekyll
[jekyll]:    http://jekyllrb.com
