---
layout: post
title:  "SSH Using Yubikey NEO"
categories: howto
---

Steps:

* - Add Yubico ppa repositories.
* - Add yubikey udev rules so that user can modify yubikey.
* - yk personalize, change mode to support OpenGPG. 0x82
* - install gnupg, gnupg-agent, & scdaemon.
* - generate gpg key (yubico steps link) &  move keys to neo.
* - configure gpg-agent.conf & scdaemon.conf for logging
* (note that gpg --card-status will return an error when scdaemon is running).
* - setup `/etc/X11/Xsession.d/90x11-common_gpg-agent` to startup gpg-agent.
* - setup .bashrc to export env variables.
* ssh-add -L 
* ssh to-some-host

* Recovery Tips.
 - key pass phrase vs pin vs admin pin.
 - unblock pin.

Setup Yubikey Neo OpenGPG Smart Card applet as SSH agent.

## Introduction x

* http://spin.atomicobject.com/2013/09/25/gpg-gnu-privacy-guard/
* https://github.com/Yubico/ykneo-openpgp
* https://developers.yubico.com/yubikey-neo-manager/
* https://developers.yubico.com/ykneo-openpgp/CardEdit.html

{% highlight console %}
{% endhighlight %}


## Yubico PPA repository

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

## Add Yubico udev rules

NOTE: As of 2015-02-17, these rules are not part of the packages in yubico's PPA. At some point they will be added, and these steps will be unnecessary. Before downloading these files, check to see if they exist in `/etc/udev/rules.d/`.
{% highlight console %}
$ for file in 69-yubikey.rules 70-yubikey.rules ; do 
    wget -q https://raw.githubusercontent.com/Yubico/yubikey-personalization/master/${file}
  done 
$ sudo cp --no-clobber 69-yubikey.rules 70-yubikey.rules /etc/udev/rules.d/
{% endhighlight %}

## ykpersonalize

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
$ ykpersonalize -m82
{% endhighlight %}

## Install gnupg-agent & scdaemon

{% highlight console %}
$ sudo apt-get install gnupg-agent scdaemon
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following packages were automatically installed and are no longer required:
  libconfig-tiny-perl libfile-touch-perl
Use 'apt-get autoremove' to remove them.
The following NEW packages will be installed:
  gnupg-agent scdaemon
0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
Need to get 0 B/393 kB of archives.
After this operation, 1,488 kB of additional disk space will be used.
Selecting previously unselected package gnupg-agent.
(Reading database ... 201128 files and directories currently installed.)
Preparing to unpack .../gnupg-agent_2.0.22-3ubuntu1.1_amd64.deb ...
Unpacking gnupg-agent (2.0.22-3ubuntu1.1) ...
Selecting previously unselected package scdaemon.
Preparing to unpack .../scdaemon_2.0.22-3ubuntu1.1_amd64.deb ...
Unpacking scdaemon (2.0.22-3ubuntu1.1) ...
Processing triggers for man-db (2.6.7.1-1ubuntu1) ...
Setting up gnupg-agent (2.0.22-3ubuntu1.1) ...
Setting up scdaemon (2.0.22-3ubuntu1.1) ...
{% endhighlight %}

## Enable CCID Mode

* Install the yubikey-neo-manager.
* Check CCID mode. Unplug & Plug NEO in again.

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

## Configure Personal gpg-agent

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


## Create Xsession script to start gpg-agent

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


## Verify SSH Key is Present

Log out and log back in again (or perform equivalent steps based on the commands above).

Verify that gpg-agent is running and that the `$GPG_AGENT_INFO` matches the running gpg-agent.

If the yubikey is installed, then you can verify this using ssh-add.

{% highlight console %}
$ ssh-add -l
2048 00:11:22:33:44:55:66:77:88:99:aa:bb:cc:dd:ee:ff cardno:000003333333 (RSA)
$ ssh-add -L
ssh-rsa AAAAB3NzzD2aBABLKJED30...aBABLKJEd30aBABLKJEd30 cardno:000003333333
{% endhighlight %}



Example link [Jekyll docs][jekyll] is great.

[jekyll]:    http://jekyllrb.com
