---
layout: post
title:  "Yubikey NEO for SSH Authentication (Generate Keys: Part 2)"
categories: howto
---

<div id="table_of_content"></div>

# {{ page.title }}

It's 10:00PM; do you know where your private keys are?

In the [previous article][part1], we enabled the Yubikey Neo OpenPGP applet.

This post covers generating and loading the Neo with a PGP Authentication key.

# Create Authentication Key

## Generate Key

While it is possible to generate a key _on the Yubikey NEO_ this method
prohibits making a backup of the key. So, we'll generate the GPG key first then
import them to the NEO. Notes taken from [key import][keyimport] guide.

By default, GPG adds the `Sign` and `Encrypt` capability to new keys. Choose
option *8* so we can add the `Authenticate` capability.

{% highlight console %}
$ gpg --expert --gen-key
gpg (GnuPG) 1.4.16; Copyright (C) 2013 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
Your selection? 8
{% endhighlight %}

Since we only want a key to `Authenticate`, disable the `Sign` and `Encrypt`
actions for the new key. The `Certify` capability cannot be removed.

{% highlight console %}
Possible actions for a RSA key: Sign Certify Encrypt Authenticate 
Current allowed actions: Sign Certify Encrypt 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? a

Possible actions for a RSA key: Sign Certify Encrypt Authenticate 
Current allowed actions: Sign Certify Encrypt Authenticate 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? s

Possible actions for a RSA key: Sign Certify Encrypt Authenticate 
Current allowed actions: Certify Encrypt Authenticate 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? e

Possible actions for a RSA key: Sign Certify Encrypt Authenticate 
Current allowed actions: Certify Authenticate 

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? q
{% endhighlight %}

## 2048-bit Key Size

The Yubikey supports a maximum key size of 2048 bits, so leave the default size.

{% highlight console %}
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 
Requested keysize is 2048 bits
{% endhighlight %}

Consider setting an expiration so that your keys will have to be rotated at
some future date.

{% highlight console %}
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 3m
Key expires at Sat 23 May 2015 12:58:45 AM EDT
Is this correct? (y/N) y
{% endhighlight %}

Add personal information, if you like.

{% highlight console %}
You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

Real name: My Name
Email address: myname@myemail.com
Comment: demo SSH key
You selected this USER-ID:
    "My Name (demo SSH key) <myname@myemail.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
{% endhighlight %}

## Optional: Passphrase

The private key will be removed once it is moved to the Yubikey. The Yubikey
SmartCard requires its own passphrase, so you can safely skip the passphrase
at this point.

{% highlight console %}
You need a Passphrase to protect your secret key.

You don't want a passphrase - this is probably a *bad* idea!
I will do it anyway.  You can change your passphrase at any time,
using this program with the option "--edit-key".
{% endhighlight %}

## Optional: More Random Bytes

If your system cannot generate random bytes quickly, consider installing
[haveged](http://www.issihosts.com/haveged/).

{% highlight console %}
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
.....+++++
.......+++++
gpg: key 3A12A5EB marked as ultimately trusted
public and secret key created and signed.

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   5  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 5u
gpg: next trustdb check due at 2015-05-16
pub   2048R/3A12A5EB 2015-02-22 [expires: 2015-05-23]
      Key fingerprint = 28D9 982D CA44 1DCE 9FD0  01F3 D469 1040 3A12 A5EB
uid                  My Name (demo SSH key) <myname@myemail.com>

Note that this key cannot be used for encryption.  You may want to use
the command "--edit-key" to generate a subkey for this purpose.
{% endhighlight %}

## Optional: Key Backup

This is not necessary since new keys could be generated at any time, but you can
backup your keys.

{% highlight console %}
$ gpg --armor --export-secret-keys 3A12A5EB > auth-secret.asc
$ gpg --armor --export 3A12A5EB > auth-public.asc
{% endhighlight %}

# Move GPG key to Yubikey Neo

To move the key to the Yubikey, we edit the key, `toggle` the key to operate on
the private key, and run the `keytocard` command.

{% highlight console %}
$ gpg --edit-key 3A12A5EB
gpg (GnuPG) 1.4.16; Copyright (C) 2013 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret key is available.

pub  2048R/3A12A5EB  created: 2015-02-22  expires: 2015-05-23  usage: CA  
                     trust: ultimate      validity: ultimate
[ultimate] (1). My Name (demo SSH key) <myname@myemail.com>

gpg> toggle

sec  2048R/3A12A5EB  created: 2015-02-22  expires: 2015-05-23
(1)  My Name (demo SSH key) <myname@myemail.com>

gpg> keytocard
Really move the primary key? (y/N) y
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]

Please select where to store the key:
   (3) Authentication key
Your selection? 3

sec  2048R/3A12A5EB  created: 2015-02-22  expires: 2015-05-23
                     card-no: 0006 03036906
(1)  My Name (demo SSH key) <myname@myemail.com>

gpg> quit
Save changes? (y/N) y
{% endhighlight %}

Verify that the key is now present on the Yubikey SmartCard.

{% highlight console %}
$ gpg --card-status
Application ID ...: D2760001240102000006030369060000
Version ..........: 2.0
Manufacturer .....: unknown
Serial number ....: 03036906
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
Authentication key: 28D9 982D CA44 1DCE 9FD0  01F3 D469 1040 3A12 A5EB
      created ....: 2015-02-22 04:59:38
General key info..: pub  2048R/3A12A5EB 2015-02-22 My Name (demo SSH key) <myname@myemail.com>
sec>  2048R/3A12A5EB  created: 2015-02-22  expires: 2015-05-23
                      card-no: 0006 03036906
{% endhighlight %}

## Change PIN / Passphrase

While OpenPGP uses the term 'PIN', all characters are supported. And, the
Yubikey Neo can support passphrases up to 127 characters long. 

{% highlight console %}
$ gpg --change-pin
gpg: OpenPGP card no. D2760001240102000006030369060000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? q
{% endhighlight %}

If a PIN is entered incorrectly too many times, the PIN will be blocked. You can
unblock a PIN. However, if you enter the admin PIN incorrectly too many times,
the device will self-destruct.

# Troubleshooting

While preparing my Yubikey Neo, on several occasions unexpected errors occurred.

The one thing these errors seem to have in common is two or more processes
trying to talk to the SmartCard at the same time.

For example, the `scdaemon` & `gpg --card-status` may produce an error like.

{% highlight console %}
$ gpg --card-status
gpg: pcsc_list_readers failed: unknown PC/SC error code (0x8010002e)
gpg: card reader not available
gpg: OpenPGP card not available: general error
{% endhighlight %}

Or, `gpg-agent` & `gpg --card-edit`.

{% highlight console %}
gpg: sending command `SCD SETATTR' to agent failed: ec=4.21393
gpg: error clearing forced signature PIN flag: general error
{% endhighlight %}

You are now ready for [part three][part3].

# References

These are some of the pages I used when configuring my own NEO.

* [Excellent GPG introduction](http://spin.atomicobject.com/2013/09/25/gpg-gnu-privacy-guard/)
* [Yubikey NEO OpenPGP applet source](https://github.com/Yubico/ykneo-openpgp) -- Unfortunately, the Yubikey hardware is closed. So Yubikey NEO owners can neither update their Yubikey, nor can they verify that this is the code running on their Yubikey.
* [Yubikey NEO Manager source](https://developers.yubico.com/yubikey-neo-manager/) -- Used to enable the OpenPGP applet.
* [Import a GPG key Yubikey NEO](https://developers.yubico.com/ykneo-openpgp/KeyImport.html)
* [GPG keys can be used for SSH!?](https://blog.habets.se/2013/02/GPG-and-SSH-with-Yubikey-NEO)

[part1]: {% post_url 2015-02-16-yubikey-neo-for-ssh-authentication-part1 %}
[part3]: {% post_url 2015-02-18-yubikey-neo-for-ssh-authentication-part3 %}
[keyimport]: https://developers.yubico.com/ykneo-openpgp/KeyImport.html
