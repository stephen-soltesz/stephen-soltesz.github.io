---
layout: post
title:  "Using the Yubikey NEO for SSH"
categories: howto
---

<div id="table_of_content"/>

# Goals

SSH keys are often stored in the local file system. A remote attacker who gains
access to your laptop or workstation can copy your SSH private keys.  Even if
protected by a passphrase, the private keys are loaded unencrypted in memory by
ssh-agent.

If your workstation or laptop were compromised by a remote
attacker, then your SSH keys theft by a remote attacker if your system is
compromised. They may be loaded into ssh-agent This post describes how to use
the Yubikey Neo to store a hardware-secured SSH key. In the end, SSH will use
the Neo for authentication.

* Enable the Yubikey Neo OpenPGP SmartCard applet.
* Load the Neo with a GPG Authentication key.
* Add `gpg-agent` to system-wide startup scripts.
* Configure `gpg-agent` to use the Yubikey Neo for SSH authentication and login.

Non-goals:

* Provide background for PGP or outline best-practices for operational security.
* SmartCards have three keys for signing, encrypting, and authenticating. This
  post is only concerned with the authentication key. 

{% highlight console %}
{% endhighlight %}

# Package Installation & Setup

## Add ppa:yubico/stable Apt Repository

{% highlight console %}
$ sudo add-apt-repository ppa:yubico/stable
 PPA for stable Yubico software.
 More info: https://launchpad.net/~yubico/+archive/ubuntu/stable
Press [ENTER] to continue or ctrl-c to cancel adding it

gpg: keyring `/tmp/tmp6n6fl1ra/secring.gpg' created
gpg: keyring `/tmp/tmp6n6fl1ra/pubring.gpg' created
gpg: requesting key 32CBA1A9 from hkp server keyserver.ubuntu.com
gpg: /tmp/tmp6n6fl1ra/trustdb.gpg: trustdb created
gpg: key 32CBA1A9: public key "Launchpad PPA for Yubico" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
OK
$ sudo apt-get update -qq
{% endhighlight %}

## Install GPG & Yubico Packages

{% highlight console %}
$ sudo apt-get install yubikey-personalization yubikey-neo-manager ykneomgr
$ sudo apt-get install gnupg-agent scdaemon
Reading package lists... Done
Building dependency tree       
Reading state information... Done
...
Setting up gnupg-agent (2.0.22-3ubuntu1.1) ...
Setting up scdaemon (2.0.22-3ubuntu1.1) ...
{% endhighlight %}

## Add udev Rules for Yubikey

NOTE: As of 2015-02-17, these rules are not part of the packages in yubico's PPA. At some point they will be added, and these steps will be unnecessary. Before downloading these files, check to see if they exist in `/etc/udev/rules.d/`.
{% highlight console %}
$ for file in 69-yubikey.rules 70-yubikey.rules ; do 
    wget -q https://raw.githubusercontent.com/Yubico/yubikey-personalization/master/${file}
  done 
$ sudo cp --no-clobber 69-yubikey.rules 70-yubikey.rules /etc/udev/rules.d/
{% endhighlight %}

# GPG Key & Yubikey Initialization

## Enable OpenPGP on Yubikey (CCID Mode)

* Check CCID mode. Unplug & Plug NEO in again.
Use ykpersonalize or neoman. ykneomgr fails.
Before setting the mode that enables the OpenPGP applet, check the current mode.

{% highlight console %}
$ apt-get install yubikey-personalization yubikey-neo-manager ykneomgr
# The ykneomgr seems to not work as of this writing.
$ ykneomgr -s
error: ykneomgr_discover_match (-4): Backend error
$ ykneomgr -l
error: ykneomgr_list_devices (-4): Backend error
# The ykinfo command does not include the current mode.
# The graphical `neoman` gives a way to view and set the CCID mode (e.g. OpenPGP).
$ neoman
# Alternately, you can set the mode explicitly.
$ ykpersonalize -m82:15:60
{% endhighlight %}

{% highlight console %}
# TODO: add examples for ykneomgr.
$ ykneomgr --get-mode
$ ykneomgr --set-mode
$ ykneomgr --list-readers
$ ykneomgr --applet-list
{% endhighlight %}

## GPG card status.
{% highlight console %}
$ gpg --version
gpg (GnuPG/MacGPG2) 2.0.22
libgcrypt 1.5.3
...

$ gpg --card-status
Application ID ...: D2760001240102000006033664820000
Version ..........: 2.0
Manufacturer .....: unknown
Serial number ....: 03366482
Name of cardholder: [not set]
Language prefs ...: [not set]
Sex ..............: unspecified
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: forced
Key attributes ...: 2048R 2048R 2048R
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
{% endhighlight %}

## Generate PGP key

Generate gpg key and import to NEO.

* https://developers.yubico.com/ykneo-openpgp/KeyImport.html

{% highlight console %}
$ gpg --gen-key
$ gpg --expert --edit-key <keyid>
gpg> addkey
$ gpg --edit-key <keyid>
gpg> toggle
gpg> keytocard
gpg> quit
{% endhighlight %}

# Automate gpg-agent

## Configure gpg-agent

The most important options are:

 * write-env-file - 
 * enable-ssh-support - 

{% highlight console %}
$ cat .gnupg/gpg-agent.conf 
write-env-file /home/<user>/.gpg-agent.sh
pinentry-program /usr/bin/pinentry-gtk-2
default-cache-ttl 86400
max-cache-ttl 86400
enable-ssh-support
{% endhighlight %}

Only if you encounter problems, enable logging from scdaemon to get more diagnostics:

{% highlight console %}
$ cat .gnupg/scdaemon.conf
log-file /home/<user>/scdaemon.log
{% endhighlight %}

## Create Xsession Startup Script for gpg-agent

* Create `/etc/X11/Xsession.d/90x11-common_gpg-agent`.
* Edit `/etc/X11/Xsession.options` to include use-gpg-agent and disable use-ssh-agent.

{% highlight bash %}
# This file is sourced by Xsession(5), not executed.
STARTGPG=
GPGAGENT=/usr/bin/gpg-agent
GPGAGENTARGS="--daemon" 

if has_option use-gpg-agent; then
  if [ -x "$GPGAGENT" ] && [ -z "$GPG_AGENT_INFO" ]; then
    STARTGPG=yes
  fi
fi

if [ -n "$STARTGPG" ]; then
  STARTUP="$GPGAGENT $GPGAGENTARGS ${TMPDIR:+env TMPDIR=$TMPDIR} $STARTUP"
fi
{% endhighlight %}

Add to your `$HOME/.bashrc`

{% highlight bash %}
if [ -f ~/.gpg-agent.sh ] ; then
  . ~/.gpg-agent.sh
  export GPG_AGENT_INFO
  export SSH_AUTH_SOCK
fi
{% endhighlight %}


## Verify SSH Key Works

Log out and log back in again (or perform equivalent steps based on the commands above).

Verify that gpg-agent is running and that the `$GPG_AGENT_INFO` matches the running gpg-agent.

If the yubikey is installed, then you can verify this using ssh-add.

{% highlight console %}
$ ssh-add -l
2048 00:11:22:33:44:55:66:77:88:99:aa:bb:cc:dd:ee:ff cardno:000003333333 (RSA)
$ ssh-add -L
ssh-rsa AAAAB3NzzD2aBABLKJED30...aBABLKJEd30aBABLKJEd30 cardno:000003333333
{% endhighlight %}

# Tips

## One process can open the SmartCard at a time.
If `scdaemon` is running, then `gpg --card-status` will produce an error.

{% highlight console %}
$ gpg --card-status
gpg: pcsc_list_readers failed: unknown PC/SC error code (0x8010002e)
gpg: card reader not available
gpg: OpenPGP card not available: general error
{% endhighlight %}

## Passwords and PINs, Oh My

If GPG keys are generated outside of the key, then you will be asked to assign the key a passphrase. When moving the key to the card, only the PIN and Admin PINs are relevant.

Note: the PINs do not need to be numbers only.
Reset the PINs using:

{% highlight console %}
{% endhighlight %}

If you happen to enter your key PIN incorrectly it will be blocked, and you need to unblock it using the Admin PIN.
 - unblock pin.

# References

These are some of the pages I used when configuring my own NEO.

* [Excellent GPG introduction](http://spin.atomicobject.com/2013/09/25/gpg-gnu-privacy-guard/)
* [Yubikey NEO OpenPGP applet source](https://github.com/Yubico/ykneo-openpgp) -- Unfortunately, the Yubikey hardware is closed. So Yubikey NEO owners can neither update their Yubikey, nor can they verify that this is the code running on their Yubikey.
* [Yubikey NEO Manager source](https://developers.yubico.com/yubikey-neo-manager/) -- Used to enable the OpenPGP applet.
* [Import a GPG key Yubikey NEO](https://developers.yubico.com/ykneo-openpgp/KeyImport.html)
* [GPG keys can be used for SSH!?](https://blog.habets.se/2013/02/GPG-and-SSH-with-Yubikey-NEO)


Example link [Jekyll docs][jekyll] is great.

[jekyll]:    http://jekyllrb.com
