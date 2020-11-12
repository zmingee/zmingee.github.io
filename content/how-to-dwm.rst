##########
How to dwm
##########

:date: 2020-11-11 20:00
:modified: 2020-11-11 20:00
:category: workstation
:tags: suckless, dwm, linux
:keywords: dwm, dmenu, slstatus, slock
:slug: how-to-dwm
:summary: How to setup dwm and related tools

I have relied on a number of software projects from the suckless team over the
years, especially when it comes to my desktop environment. My workstation setup
leans heavily on ``dwm``, ``dmenu``, ``slstatus``, and ``slock``. Several people
have asked me about my setup, either out of curiosity or because they want to
give a tiling window manager a shot. It can be fairly daunting to jump into it,
especially because the suckless projects do not rely on GUI- or file-based
configuration. All customization is done by editing C header files. That,
combined with not having basics like a straightforward way of setting a
wallpaper, makes for a challenge to anyone wanting to get their feet wet. I
finally decided to write a brief guide on how to setup ``dwm`` and the related
software. Much of this is opinionated, because with a desktop environment like
this, you're going to customize it to your liking down the road. After following
along, you'll at least be able to setup an environment similar to mine, and also
know what you can do next.

I'm going to assume if you're reading this that you're proficient with Linux,
your distro, and the related toolchains. We'll be using ``git``, ``make``,
editing C header files, compiling software... This isn't a guide on how to
compile a C program, so details will be glossed over to focus on the software
we're trying to build, not how to build them. I also won't be covering all of
the intricate details of using ``dwm``. The project page has a good tutorial to
show you the basics of using ``dwm``: https://dwm.suckless.org/tutorial/. But
the best way to learn is by experience. Follow along, install it, and try it
out for yourself.

******
Basics
******

To quote the suckless project directly:

    dwm is a dynamic window manager for X. It manages windows in tiled, monocle
    and floating layouts. All of the layouts can be applied dynamically,
    optimising the environment for the application in use and the task
    performed.

What that means is, ``dwm`` is a windowing manager, not a desktop environment.
This isn't Gnome, or KDE, or XFCE.. It renders "windows", but the comparisons
end there. It does not come with any kind of desktop environment, just tooling
to render windows and interact with them. In fact, ``dwm`` doesn't even render a
toolbar above windows, or an "X" to close them. Much of the interaction with
``dwm`` goes through the keyboard.

That being said, some other features you may consider "basic" aren't provided
out of the box. ``dwm`` supports a "status bar" for displaying system
information or whatever else you may like, but only through an external
interface. Getting data into the status bar requires a separate program such as
``slstatus``. Further, there is no default lock screen or window. This is also
provided via a separate program, such as ``slock``. I'll cover setting up these
as well, just understand that this is the intent of dwm and a result of
following the UNIX philosophy. All ``dwm`` does is provide a window manager. All
other functionalities are out of scope.

********************
Setting up ``dmenu``
********************

First we will setup a program called ``dmenu``. This is a dependency of ``dwm``,
and is used to render the tag and status bar. Most people won't need to do any
configuration, as ``dwm`` will pass it's own configuration values to it at
runtime anyways.

Before we compile ``dmenu``, you should see if your distros package manager
already has a package available. In the case of Arch Linux, ``dmenu`` is already
in the community repository. If it's available, just go ahead and install it
from there since we won't be editing it's configuration anyways. Otherwise, I'll
go over how to build/install from source.

# Install dependencies

    - ``libxinerama``
    - ``fontconfig``

# Clone ``dmenu`` repository

    ::

        git clone https://git.suckless.org/dmenu

# Compile and install

    ::

        cd dmenu
        make
        sudo make install

******************
Setting up ``dwm``
******************

Now that ``dmenu`` is installed, we can setup ``dwm``. This essentially requires
building from source. While some distros do have ``dwm`` available in their
repos, these have no configuration, no patches (for features/tweaks), just the
base code. These packages always require some kind of scaffolding for applying
your changes to the package before installation. Whatever the case may be, it's
going to be a lot simpler to just build from source and manage it directly.

The only dependency you need for ``dwm`` other than ``dmenu`` are the Xlib
header files. Installing "devel" or "header" packages varies depending on the
distro you use. In the case of Arch Linux, you'll want ``libx11`` from the extra
repository, as it contains the header files.

- Arch Linux: ``libx11``
- Ubuntu: ``libx11-dev``
- Fedora/CentOS/RHEL: ``libx11-devel``

Once we have that package installed, we can clone the ``dwm`` repository:

::

    git clone git://git.suckless.org/dwm

Move into the resultant directory and pull up your favorite text editor.

Configuring suckless projects
=============================

Most (if not all) suckless projects are configured by editing a C header file.
In most cases, this is named ``config.def.h``. This file gets copied to
``config.h`` by ``make``, and that file is used during compilation. When you
change a config option, you want to:

::

    rm config.h
    make

The file is set as a target in the ``Makefile``, but ``config.def.h`` isn't a
dependency. This means that ``make`` won't overwrite the ``config.h`` file by
itself; you will have to remove the file yourself.

If you open the ``config.def.h`` file in your text editor, you'll see a basic C
header file. If you don't have any experience with C, this may look rather
strange. I won't get into the weeds of header file syntax. Much of it is rather
self-explanatory for our purposes.

In the first section, you'll see appearance settings. These control border
sizes, bar location, theming, etc. Then come tagging, layouts, keys, shortcuts,
and button mappings. These sections are specific to ``dwm``, but are a good
outline of how suckless software is generally configured.

Configuring dwm
===============

I'll go through some of the settings in ``dwm`` you may be interested in.

Appearance
----------

To really be comfortable with ``dwm``, you're going to want to theme and style
it to your liking. I personally like the Nord theme, and I've used it throughout
my environment. I'll reference those settings here, but feel free to change it
to your liking.

Borders
^^^^^^^

The default border size is a little hard to see on 2K+ monitors, so you may want
to bump up the size:

::

    static const unsigned int borderpx = 2;

Bar Location
^^^^^^^^^^^^

If you prefer your tag/status bar to be on the bottom, you can configure this:

::

    static const int topbar = 0;

Fonts
^^^^^

In theory, your fonts will be configured sanely in your OS, and you can depend
on that for setting your default font. Setting fonts can be a pain though
depending on your distro. If you have issues getting ``dwm`` to load your
preferred font, you can hardcode this in the settings:

::

    static const char *fonts[]          = { "Ubuntu Mono:style=Regular:size=14:antialias=true:autohint=true" };
    static const char dmenufont[]       = "Ubuntu Mono:style=Regular:size=14:antialias=true:autohint=true";

Theme
^^^^^

You can style this to your liking, but these are the settings I use for the Nord
theme:

::

    static const char col_gray1[]       = "#2E3440";
    static const char col_gray2[]       = "#2E3440";
    static const char col_gray3[]       = "#D8DEE9";
    static const char col_gray4[]       = "#2E3440";
    static const char col_cyan[]        = "#88C0D0";

Tagging
-------

The first letter in ``dwm`` stands for "dynamic". Each monitor has it's own set
of tags Each application window exists on a single tag, and you can view
multiple tags at a time. These tags show up on the left side of the "bar". By
default, these are rendered as the index number of the tag. You can set them to
any character(s) you'd like though. Some people set these to unicode symbols,
some set them to strings, or you could be like me and just keep the defaults:

::

    static const char *tags[] = { "1", "2", "3", "4", "5", "6", "7", "8", "9" };

::

    static const char *tags[] = {
        "main",
        "comms",
        "media",
        "devel",
        "ref",
        "6", "7", "8", "9"
    };

Below the tag array, you can configure default rules for applications. This is
useful in a few scenarios. You may want Slack to always open on your tag for
communication apps, or open Spotify/VLC/etc on your media tag. Maybe you want
some applications to default to your second or third monitor. Or, you could use
something like Pidgin that has a small contact window that doesn't display
properly as a full tile so it needs to stay floating. Whatever the case, the
rules array lets you customize the defaults.

::

    /* class      instance    title       tags mask     isfloating   monitor */
    { "Gimp",     NULL,       NULL,       0,            1,           -1 },
    { "Firefox",  NULL,       NULL,       1 << 8,       0,           -1 },

The default config file has Gimp and Firefox as examples. Gimp is actually a
very good example, because it uses multiple windows for displaying swatches or
tools. Displaying these as full tiles is going to make using Gimp difficult, so
it defaults to having them float.

In order for ``dwm`` to select the correct window, you need to supply it with a
combination of window class, instance, and title. The config file references
``xprop`` for getting this information but not actually how to use it. All you
have to do is run:

::

    xprop | grep -E "WM_(CLASS|NAME)"

Your cursor will turn into a cross selector. Click the window you'd like to
customize, and the command will return the information you need. ``WM_CLASS``
will have two values, ``instance`` and ``class`` respectively. ``WM_NAME`` will
have a single value. If the application you're customizing has dialog boxes or
separate windows it opens, you'll want to check the properties for the main
window as well as any child windows. If you want all of the windows to have the
same setting, use less specifiers. If you only want a single of it's dialog
windows to be customized, use the all the specifiers for that dialog window.
Here are a few examples of what I'm using:

::

    /* class            instance            title           tags mask   isfloating  monitor */
    { "Claws-mail",     "claws-mail",       NULL,           1 << 1,     0,          1 },
    { "Gajim",          "org.gajim.Gajim",  "Gajim",        1 << 1,     0,          1 },
    { "Pidgin",         "Pidgin",           "Buddy List",   1 << 1,     1,          1 },
    { "Pidgin",         "Pidgin",           NULL,           1 << 1,     0,          1 },
    { "Slack",          NULL,               NULL,           1 << 1,     0,          1 },
    { "Zeal",           NULL,               NULL,           1 << 3,     0,          1 },
