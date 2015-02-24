---
layout: post
title:  "Yubikey NEO for SSH Authentication (Everyday Use: Part 3)"
categories: howto
---

<div id="table_of_content"></div>

# {{ page.title }}

It's 10:00PM; do you know where your private keys are?

In the last two articles, we:

* [Enabled the OpenPGP applet on the Yubikey Neo (Part 1)][part1]
* [Generated keys for the Yubikey Neo Using GPG (Part 2)][part2]

Now, we will prepare our environment for everyday use, including:

* Enabling `gpg-agent` in system-wide startup scripts.
* Using the Yubikey NEO for SSH authentication.

# Auto-start gpg-agent

Our authentication key is loaded on the Yubikey Neo and `gpg-agent` can talk to
the Yubikey. Ultimately, SSH will talk to `gpg-agent` using the `ssh-agent`
protocol. So, `gpg-agent` should be configured to run as soon as we login.

## Configuration for gpg.conf & gpg-agent.conf

If these files do not already exist, create them.
 
GPG configuration: `~/.gnupg/gpg.conf`

  * `use-agent` instructs GPG to use the gpg-agent, and the Xsession
    startup script uses this setting when deciding to start gpg-agent.

{% highlight console %}
$ cat <<EOF >> .gnupg/gpg.conf
use-agent
EOF
{% endhighlight %}

GPG Agent configuration: `~/.gnupg/gpg-agent.conf`

 * `enable-ssh-support` tells `gpg-agent` to support the ssh-agent protocol.

{% highlight console %}
$ cat <<EOF >> .gnupg/gpg-agent.conf
pinentry-program /usr/bin/pinentry-gtk-2
default-cache-ttl 180
max-cache-ttl 600
enable-ssh-support
{% endhighlight %}

Optional: `~/.gnupg/scdaemon.conf`

 * If you encounter problems later on, you can enable logging from scdaemon to get
   more diagnostics.

{% highlight console %}
$ cat <<EOF > .gnupg/scdaemon.conf
log-file ${HOME}/scdaemon.log
{% endhighlight %}

## Edit Xsession.options

The `gnupg-agent` package includes an Xsession.d script to start `gpg-agent`.

During user login, `/etc/X11/Xsession.d/90gpg-agent` performs these steps:

 * Does `$HOME/.gnupg/gpg.conf` have `use-agent` option? If so:

   - Start `gpg-agent`.
   - `gpg-agent` writes environment variables to
     `$HOME/.gnupg/gpg-agent-info-$HOSTNAME`.

`gpg-agent` will operate as our `ssh-agent`. So, *you must remove*
"use-ssh-agent" from `/etc/X11/Xsession.options` to prevent other Xsession
scripts from starting a separate `ssh-agent`.

## Update .bashrc

`gpg-agent` writes environment variables to `$HOME/.gnupg/gpg-agent-info-$HOSTNAME`.

So, add a command to your `$HOME/.bashrc` to sources and exports these
variables.

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
would like to access.

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

This means that you are asked for your PIN/passphrase *once per X11 session*.
This could be days or weeks.

To prompt for a passphrase more often, set `gpg-cache-method` to either:

* `idle` -- like a screen saver, the ttl starts counting down after every use.
* `timeout` -- is a strict ttl that starts counting on first use.

The `gpg-cache-ttl` is in seconds.

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

[part1]: {% post_url 2015-02-16-yubikey-neo-for-ssh-authentication-part1 %}
[part2]: {% post_url 2015-02-17-yubikey-neo-for-ssh-authentication-part2 %}
