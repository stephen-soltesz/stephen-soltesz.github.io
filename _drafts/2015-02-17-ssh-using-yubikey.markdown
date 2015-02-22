---
layout: post
title:  "Using the Yubikey NEO for SSH"
categories: howto
---

<div id="table_of_content"/>

It's 10:00PM; do you know where your private keys are?

Even if protected by a passphrase, private keys are unencrypted once loaded in 
memory. As an example, you can dump the process memory of `ssh-agent`,
[to extract the private keys](https://blog.netspi.com/stealing-unencrypted-ssh-agent-keys-from-memory/).

This post describes how to use the _Yubikey Neo_ for hardware-secured SSH 
authentication. This post walks through the following steps:

* Enabling the Yubikey Neo OpenPGP SmartCard applet.
* Loading the Neo with a GPG Authentication key.
* Enabling `gpg-agent` in system-wide startup scripts.
* Using the Yubikey NEO for SSH authentication.

Non-goals:

* Provide background for OpenPGP or detail additional best-practices for 
  operational security.
* OpenPGP SmartCards have three keys for signing, encrypting, and 
  authenticating. This post only focuses on the authentication key. 


# Package Installation & Setup

## Add ppa:yubico/stable Apt Repository

Yubico publishes open source libraries and utilities for managing Yubikeys.  
Their packages are distributed by many Linux distributions, but the most current 
versions are always available from the Yubico PPA.

{% highlight console %}
$ sudo add-apt-repository ppa:yubico/stable
 PPA for stable Yubico software.
 More info: https://launchpad.net/~yubico/+archive/ubuntu/stable
Press [ENTER] to continue or ctrl-c to cancel adding it
gpg: keyring '/tmp/tmp6n6fl1ra/secring.gpg' created
gpg: keyring '/tmp/tmp6n6fl1ra/pubring.gpg' created
gpg: requesting key 32CBA1A9 from hkp server keyserver.ubuntu.com
gpg: /tmp/tmp6n6fl1ra/trustdb.gpg: trustdb created
gpg: key 32CBA1A9: public key "Launchpad PPA for Yubico" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
OK
$ sudo apt-get update -qq
{% endhighlight %}

## Install GPG & Yubico Packages

Install the gnupg and Yubikey management packages.

{% highlight console %}
$ sudo apt-get install gnupg-agent scdaemon yubikey-personalization
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

To enable the OpenPGP applet and keep using the OTP & U2F features of your NEO, 
you need to use the mode `-m6`. Other options are available, including an 
auto-eject, OpenPGP-only mode. See the [yubikey-personalization(1)](https://github.com/yubico/yubikey-personalization) man page for more details.

{% highlight console %}
$ ykpersonalize -m6
Firmware version 3.3.6 Touch level 1541 Program sequence 1
.
The USB mode will be set to: 0x6
.
Commit? (y/n) [n]: y
{% endhighlight %}

Unplug & plug the NEO in again.

## GPG card status.

The following steps depend on GPG version 2.0.22+.

{% highlight console %}
$ gpg-agent --version
gpg-agent (GnuPG) 2.0.22
libgcrypt 1.5.3
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
{% endhighlight %}

Verify that you can check the card status.

{% highlight console %}
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

While it is possible to generate a key _on the Yubikey NEO_ this method
prohibits making a backup of the key. So, we'll generate the GPG key first then
import them to the NEO. Notes taken from [key import](https://developers.yubico.com/ykneo-openpgp/KeyImport.html) guide.

{% highlight console %}
$ gpg --gen-key
gpg (GnuPG) 1.4.16; Copyright (C) 2013 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
.
Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 
Requested keysize is 2048 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 3m
Key expires at Fri 22 May 2015 01:31:55 AM EDT
Is this correct? (y/N) y
.
You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"
.
Real name: fake key
Email address: whatever@myemail.com
Comment: test key for howto.
You selected this USER-ID:
    "fake key (test key for howto.) <whatever@myemail.com>"
.
Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
You need a Passphrase to protect your secret key.
.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
..+++++
......+++++
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
..+++++
......+++++
gpg: key 454B28A0 marked as ultimately trusted
public and secret key created and signed.
.
gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   3  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 3u
gpg: next trustdb check due at 2015-05-16
pub   2048R/454B28A0 2015-02-21 [expires: 2015-05-22]
      Key fingerprint = FB17 D887 29FF 0A29 D1C0  3DC0 9C01 A757 454B 28A0
uid                  fake key (test key for howto.) <whatever@myemail.com>
sub   2048R/4EEBB175 20
{% endhighlight %}

{% highlight console %}
$ gpg --expert --edit-key <keyid>
gpg> addkey
.
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


