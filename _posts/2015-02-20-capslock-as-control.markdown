---
layout: post
title:  "Linux CapsLock as Control in X11 & Console"
categories: howto
---

# {{ page.title }}

Keyboards vary the [`Control` key location][location]. But, I fell in love with
the "Unix" layout early on (where `Control` is just left of `a`). Ever since,
I remap the `CapsLock` key as a `Control` key on every system I use regularly.

I'm not alone. There are *[hundreds][search]* of articles with hints for
changing the CapsLock key into something else. Unfortunately, most options are
incomplete.

# X11 or Console?

For Linux, in X11 some options are:

* Find the option in your window manager's keyboard configuration (if supported).
* Call `setxkbmap` from some X11 session startup script.
* Call `xmodmap` if `setxkbmap` is not available.

Unfortunately, these only work while you are *in* X11. When you switch to the
console, the CapsLock key is once again a CapsLock key.

For the console, an option is:

* Use `/etc/kbd/remap` to change the keyboard translation table. (Or, do this 
  manually with `dumpkeys` & `loadkeys`.)

Unfortunately, this approach only works while you are *at* the console. X11
manages an independent key map.

# All Together

Fortunately, both X11 and the console depend on the Linux kernel. The kernel 
receives raw scancodes from the keyboard. The kernel maps scancodes to keycodes.
And finally, these keycodes are received by higher level systems, like X11 or 
the console.

We can change the mapping from scancode to keycode using `setkeycodes`.

## Discover Scancodes

First, save the current scancode mapping to compare with the final map when
we're done.

{% highlight console %}
$ sudo getkeycodes > keycodes.before
{% endhighlight %}

Next, start `showkey --scancodes` and press the CapsLock key. The first number
is the raw scan code. The second number is the key released (8th bit high). We
need the first number. Your keyboard may return a different scancode.

{% highlight console %}
$ sudo showkey --scancodes
[ if you are trying this under X, it might not work
since the X server is also reading /dev/console ]

press any key (program terminates 10s after last keypress)...
0x3a
0xba
{% endhighlight %}

## Discover Keycodes

Now, start `showkey` again (without options) and press the `Control` key. This
is the keycode we want to map to the `CapsLock` scancode found above.

{% highlight console %}
$ sudo showkey
[ if you are trying this under X, it might not work
since the X server is also reading /dev/console ]

press any key (program terminates 10s after last keypress)...
keycode  29 press
keycode  29 release
{% endhighlight %}

## Map CapsLock Scancode to Control Keycode

Since we know the raw scancode for `CapsLock` and the preferred keycode for
`Control`, we can set this mapping with `setkeycodes <scancode> <keycode>`.

{% highlight console %}
$ sudo setkeycodes 3a 29
{% endhighlight %}

Since this alters the kernel's scancode mapping, it takes effect immediately in
both the console and X11.

To better understand what has changed, compare the new scancode map with the original.

{% highlight console %}
$ sudo getkeycodes > keycodes.after
$ diff keycodes.before keycodes.after
3a4,6
 >  0x38:   56  57  29  59  60  61  62  63
 >  0x40:   64  65  66  67  68  69  70  71
 >  0x48:   72  73  74  75  76  77  78  79
{% endhighlight %}

Note the first row third value. This is scancode position 0x3a with a value of
29. Notice all the other values from 56 to 79. Low-valued scancodes are mapped
to the identical valued keycodes. For example, 0x38 (56 dec) is mapped to 56
(0x38), etc. Since we added a code that is not identitical, the other values
are reported for completeness.

## Add to /etc/rc.local

Finally, add the setkeycodes command run above to `/etc/rc.local`. Now the change will take effect on every boot.

[location]: http://en.wikipedia.org/wiki/Control_key#Location_of_the_key
[search]: https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=linux%20change%20caps%20lock
