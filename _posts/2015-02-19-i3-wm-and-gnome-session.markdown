---
layout: post
title: "Add GNOME-panel to i3 Window Manager"
categories: howto
---

# {{ page.title }}

## Dilemma

Tiling window managers promise simplicity, elegance, and efficiency. And, 
conventional desktop environments, like GNOME, provide mature and convenient 
applets for Wi-Fi configuration, battery status, volume control, screen lock, 
etc.

How can I get the benefits of a tiling window manager without losing the 
convenience built into modern desktops?

## Options

Ubuntu includes native packages for several popular tiling window managers.

* [Awesome window manager][awesomewm] -- uses Lua for configuration.
* [XMonad window manager][xmonad] -- uses Haskell for configuration.
* [i3 window manager][i3wm] -- uses a simple, custom configuration language.

Each are primarily window managers. Even so, each also includes a minimal status 
/ widget bar. e.g. i3bar, awesome widgets, xmobar. Though, these status bars 
require extra configuration for features that are standard in
[gnome-panel][gnome-panel].

For example, i3 with gnome-panel:

![gnome-panel with i3]({{site.url}}/images/i3-with-gnome-panel.png =500x)

## Configuration

Thankfully, most of the hard work is already done by an ArchLinux package. So,
download the tarball for the ArchLinux [i3-gnome package][archlinux-i3gnome].

Edit `i3-gnome.session` and add `gnome-panel` to the `RequiredComponents`.

{% highlight bash %}
RequiredComponents=gnome-settings-daemon;gnome-panel;i3-gnome;
{% endhighlight %}

Install the desktop and session files:

{% highlight console %}
# install -D -m 644 ./i3-gnome-xsession.desktop /usr/share/xsessions/i3-gnome.desktop
# install -D -m 644 ./i3-gnome.session /usr/share/gnome-session/sessions/i3-gnome.session
# install -D -m 644 ./i3-gnome-app.desktop /usr/share/applications/i3-gnome.desktop
# install -D -m 755 ./i3-gnome /usr/bin/i3-gnome
# update-desktop-database -q
{% endhighlight %}

Finally, log out of your current X session, then choose the "i3 + GNOME"
session.

Enjoy i3 with Gnome panel!

# References

* [i3 Tiling Window Manager][i3wm]
* [ArchLinux i3-gnome package][archlinux-i3gnome]


[i3wm]: https://i3wm.org
[awesomewm]: http://awesome.naquadah.org/
[xmonad]: http://xmonad.org/
[gnome-panel]: http://en.wikipedia.org/wiki/GNOME_Panel
[archlinux-i3gnome]: https://aur.archlinux.org/packages.php?ID=58675
