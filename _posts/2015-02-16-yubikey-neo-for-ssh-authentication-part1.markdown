---
layout: post
title:  "Yubikey NEO for SSH Authentication (Setup: Part 1)"
categories: howto
---

<div id="table_of_content"></div>

# {{ page.title }}

It's 10:00PM; do you know where your private keys are?

Even if protected by a passphrase, private keys are unencrypted once loaded in 
memory. For example, you can dump the process memory of `ssh-agent`
[and extract the private keys](https://blog.netspi.com/stealing-unencrypted-ssh-agent-keys-from-memory/).

This post describes how to setup the _Yubikey Neo_ for hardware-secured SSH 
authentication. This post walks through the following steps:

* Enabling the Yubikey Neo OpenPGP SmartCard applet.

Non-goals:

* Provide background or best-practices for using OpenPGP. There are other
  excellent introductions online.
* OpenPGP SmartCards support three keys for signing, encrypting, and 
  authenticating. This post only focuses on the authentication key. 

# Install Packages

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
$ sudo apt-get install gnupg-agent scdaemon yubikey-personalization yubikey-neo-manager
Reading package lists... Done
Building dependency tree       
Reading state information... Done
...
Setting up gnupg-agent (2.0.22-3ubuntu1.1) ...
Setting up scdaemon (2.0.22-3ubuntu1.1) ...
{% endhighlight %}

## Check Package Version

Yubikey OpenPGP applet requires GPG agent & `scdaemon` version 2.0.22+.

{% highlight console %}
$ gpg-agent --version
gpg-agent (GnuPG) 2.0.22
libgcrypt 1.5.3
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
.
$ /usr/lib/gnupg2/scdaemon --version
scdaemon (GnuPG) 2.0.22
libgcrypt 1.5.3
libksba 1.3.0
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
{% endhighlight %}

# Configure Yubikey

## Confirm the Yubikey is Present

Verify that the yubikey is present. The `ykinfo` utility should report some
basic information about your yubikey.

{% highlight console %}
$ ykinfo -a
serial: 3012345
serial_hex: 345571
serial_modhex: efideg
version: 3.3.6
touch_level: 1285
programming_sequence: 1
slot1_status: 1
slot2_status: 0
vendor_id: 1050
product_id: 116
{% endhighlight %}

## Enable OpenPGP Applet on Yubikey (CCID Mode)

The OpenPGP applet is not accessible by default on Yubikey NEOs. The OpenPGP
applet requires that CCID mode is enabled.

You can use the `neoman` graphical:

![neoman mode change]({{ site.url }}/images/neoman-modes.png =500x)

Enable the OpenPGP applet and keep using the OTP & U2F features of your NEO, 
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

## Check GPG Card Status

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

You are now ready for [part two](http://192.168.1.120:4000/howto/2015/02/22/using-yubikey-neo-for-ssh-authentication.html)
<!-- { post_url using-yubikey-neo-for-ssh-authentication } ). -->

# References

These are some of the pages I used when configuring my own NEO.

* [Excellent GPG introduction](http://spin.atomicobject.com/2013/09/25/gpg-gnu-privacy-guard/)
* [Yubikey NEO OpenPGP applet source](https://github.com/Yubico/ykneo-openpgp) -- Unfortunately, the Yubikey hardware is closed. So Yubikey NEO owners can neither update their Yubikey, nor can they verify that this is the code running on their Yubikey.
* [Yubikey NEO Manager source](https://developers.yubico.com/yubikey-neo-manager/) -- Used to enable the OpenPGP applet.
* [Import a GPG key Yubikey NEO](https://developers.yubico.com/ykneo-openpgp/KeyImport.html)
* [GPG keys can be used for SSH!?](https://blog.habets.se/2013/02/GPG-and-SSH-with-Yubikey-NEO)


