---
layout: post
title:  "Yubikey NEO for SSH Authentication (Everyday Use: Part 3)"
categories: howto
---

<div id="table_of_content"></div>

# {{ page.title }}

It's 10:00PM; do you know where your private keys are?

In the past two articles, we:

* [Setup the Yubikey Neo for SSH Authentication]().
* [Generated keys for the Yubikey Neo]().

We have a _Yubikey Neo_ with a hardware-secured Authentication key.  In this
post, we will prepare our environment for everyday use.

This post covers the following:

* Enabling `gpg-agent` in system-wide startup scripts.
* Using the Yubikey NEO for SSH authentication.

# Automate gpg-agent

Once the Yubikey Neo has our authentication key, SSH will interact with
`gpg-agent`. So we want to guarantee that gpg-agent is configured and running
when we log into our account.

## Configuration for gpg.conf & gpg-agent.conf

If these files do not already exist, create them.
 
* `~/.gnupg/gpg.conf` -- the `use-agent` directive tells GPG to use the
  gpg-agent and the Xsession startup script uses this directive to decide to
  start gpg-agent.

{% highlight console %}
$ cat <<EOF > .gnupg/gpg.conf 
use-agent
EOF
{% endhighlight %}

* `~/.gnupg/gpg-agent.conf` -- the `enable-ssh-support` directive tells gpg-agent
to support the ssh-agent protocol.

{% highlight console %}
$ cat <<EOF > .gnupg/gpg-agent.conf 
pinentry-program /usr/bin/pinentry-gtk-2
default-cache-ttl 180
max-cache-ttl 600
enable-ssh-support
{% endhighlight %}

If you encounter problems later on, you can enable logging from scdaemon to get
more diagnostics.

{% highlight console %}
$ cat <<EOF > .gnupg/scdaemon.conf
log-file ${HOME}/scdaemon.log
{% endhighlight %}

## Edit Xsession.options

The `gnupg-agent` package includes an Xsession.d script
(`/etc/X11/Xsession.d/90gpg-agent`) that starts gpg-agent automatically when
it finds 'use-agent' in `$HOME/.gnupg/gpg.conf`.

Since gpg-agent will operate as an ssh-agent, edit `/etc/X11/Xsession.options`
and remove the option to `use-ssh-agent`.

## Update .bashrc

The Xsession.d script writes the gpg-agent & ssh-agent environment variables to
`$HOME/.gnupg/gpg-agent-info-${HOSTNAME}`.

Add a command to your `$HOME/.bashrc` that sources and exports these variables.

{% highlight bash %}
GPGENV="${HOME}/.gnupg/gpg-agent-info-${HOSTNAME}"
if [ -f ${GPGENV} ] ; then
  . ${GPGENV}
  export GPG_AGENT_INFO
  export SSH_AUTH_SOCK
fi
{% endhighlight %}

## Verify Environment

Log out, then log back in and verify that gpg-agent is running and that the
`$GPG_AGENT_INFO` from `$HOME/.gnupg/gpg-agent-info-${HOSTNAME}` matches the
running gpg-agent.

## Check ssh-agent

If the yubikey is installed, then you can verify this using ssh-add.

{% highlight console %}
$ ssh-add -l
2048 00:11:22:33:44:55:66:77:88:99:aa:bb:cc:dd:ee:ff cardno:000003333333 (RSA)
{% endhighlight %}

Copy the public key to the `~/.ssh/authorized_keys` file of remote hosts you
would like to access 

{% highlight console %}
$ ssh-add -L
ssh-rsa AAAAB3NzzD2aBABLKJED30...aBABLKJEd30aBABLKJEd30 cardno:000003333333
{% endhighlight %}

## PIN/Password Cache

In a GNOME-base desktop `/usr/bin/pinentry` (the default for gpg-agent) links to
`pinentry-gtk-2`, which relies on GNOME settings. GNOME settings cache
passphrases independently of the cache-ttl settings from `gpg-agent.conf`.

By default, the GNOME cache setting is:

* `org.gnome.crypto.cache gpg-cache-method 'session'`.

This means that you are asked for your PIN/passphrase *once per session*. This
could be days or weeks.

To prompt for a passphrase more often, set `gpg-cache-method` to:

* `idle` -- like a screen saver, the timeout starts counting every use.
* `timeout` -- is a strict limit that starts counting on first use.


{% highlight console %}
$ gsettings set org.gnome.crypto.cache gpg-cache-method 'idle'
$ gsettings set org.gnome.crypto.cache gpg-cache-ttl 300
{% endhighlight %}

# References

These are some of the pages I used when configuring my own NEO.

* [Excellent GPG introduction](http://spin.atomicobject.com/2013/09/25/gpg-gnu-privacy-guard/)
* [Yubikey NEO OpenPGP applet source](https://github.com/Yubico/ykneo-openpgp) -- Unfortunately, the Yubikey hardware is closed. So Yubikey NEO owners can neither update their Yubikey, nor can they verify that this is the code running on their Yubikey.
* [Yubikey NEO Manager source](https://developers.yubico.com/yubikey-neo-manager/) -- Used to enable the OpenPGP applet.
* [Import a GPG key Yubikey NEO](https://developers.yubico.com/ykneo-openpgp/KeyImport.html)
* [GPG keys can be used for SSH!?](https://blog.habets.se/2013/02/GPG-and-SSH-with-Yubikey-NEO)

