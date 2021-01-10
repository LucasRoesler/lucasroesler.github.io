---
title: "Customizing the keyboard layout on the Dell XPS 13"
summary: "Fixing the Dell XPS arrow keys using XKB"
date: 2021-01-10T00:00:00+02:00
tags:
  - programming
  - linux
  - keyboard
  - dell xps
draft: false
coverImage: images/cover.jpg
---

In my last post I mentioned that I got a Dell XPS 13 last year. For awhile it was my wife's computer. But, for various reasons, now it is mine! Cue the evil laughter, mwahhahaha. However, I think Dell got the last laugh with the bizarre keyboard layout for PgUp and PgDn being half-hight and just above the left and right arrow keys! 

{{<image src="images/arrow-keys.png" title="The arrow keys on my Dell XPS 13" >}}

Apparently, someone has realized this is a horrible layout because the latest XPS 13 looks like this

{{<image src="images/new-arrow-keys.png" title="The arrow keys on the 2021 Dell XPS 13" >}}

If only I had known, I probably would have waited!


However, [learning a little bit about `XKB` for Linux][damko], it is relatively easy to just fix the current keyboard and avoid making any new e-waste. All we need to do is turn the `PgUp` and `PgDn` into Left and Right keys. 

<!-- more -->

## TLDR
All of the code and an install script for the new layout can be found on Github.   To get the new layout, just use

```sh
git clone git@github.com:LucasRoesler/xkb-xps.git
cd xkb-xps
bash install.sh
```

Afterward, your `PgUp` and `PgDn` will now behave as arrow keys.


## `xmodmap` vs `xkb` Symbols
If you start Googling something like `linux customize keyboard` or `change behavior of <X> key` and you will eventually run into `xmodmap`

> The xmodmap program is used to edit and display the keyboard modifier map and keymap table that are used by client applications to convert event keycodes into keysyms. It is usually run from the user's session startup script to configure the keyboard according to personal tastes.

And this will work, but I started investigating how the keyboard works and how to make the configuration more persistent. One way to do this is to create a custom keyboard layout, which is what I will show you below. 

Diving into `XKB` was overwhelming at first. There are several moving parts: types, geometries, symbols. All I want to do is modify a few keys, so we only need to update the _symbols_.


## Creating a new layout

First we need to create a new symbols file that defines the new behavior for `PgUp` and `PgDn`. It looks likes this:

```txt
partial
xkb_symbols "fix_pg_btns" {
    include "us"
    key <PGUP>  { 
        type="PC_ALT_LEVEL2",
        symbols[Group1] = [ Left, Prior ]
    };
    key <PGDN> { 
        type="PC_ALT_LEVEL2",
        symbols[Group1] = [ Right, Next ]
    };
};
```

Let's dig into what happens here:

The first two lines

```txt
partial
xkb_symbols "fix_pg_btns" {
```
declares a new symbol map named `fix_pg_btns` while `partial` indicates that this map doesn't cover a complete keyboard,, in this case, just two keys.

Next, 

```txt
include "us"
```
ensures that we have the default `us` layout, which is why I normally used and was the layout configured by Ubuntu by default.

Finally, we have the actually key mapping. 

```txt
key <PGUP>  { 
    type="PC_ALT_LEVEL2",
    symbols = [ Left, Prior ]
};
```
This mapping says that `PGUP` has two actions, the default action is `Left`, i.e. the left arrow. The modified action is `Prior`, page up. The type, indicates that the modifier is `Alt`. 

Figuring all of this out took a lot of experimentation. You can use `xev` to discover these action names. In my case, I wanted to know the correct names for the Page Up action and the Left Arrow action, so I ran `xev` and then pressed the keys (I have trimmed the output to just the relevant keys)

```sh
$ xev
#....
KeyPress event, serial 37, synthetic NO, window 0x4a00001,
    root 0x7bb, subw 0x0, time 10216510, (-658,-240), root:(1173,815),
    state 0x8, keycode 112 (keysym 0xff55, Prior), same_screen YES,
    XLookupString gives 0 bytes: 
    XmbLookupString gives 0 bytes: 
    XFilterEvent returns: False

#....
KeyRelease event, serial 37, synthetic NO, window 0x4a00001,
    root 0x7bb, subw 0x0, time 10425065, (-860,-188), root:(971,867),
    state 0x0, keycode 113 (keysym 0xff51, Left), same_screen YES,
    XKeysymToKeycode returns keycode: 112
    XLookupString gives 0 bytes: 
    XFilterEvent returns: False
```

The important parts are `(keysym 0xff55, Prior)` and `(keysym 0xff51, Left)` showing the action names that were used when I pressed my `PgUp` and left arrow button.

## Installing the new layout
In this section I assume you have cloned by git repo

```sh
$ git clone git@github.com:LucasRoesler/xkb-xps.git
$ cd xkb-xps
```

Once we have a new layout file we need to copy or link it to `/usr/share/X11/xkb/symbols`. This is required due to how XKB works:

> An XKB keymap is constructed from a number of components which are compiled only as needed.  The source for all of the components can be found in /usr/share/X11/xkb. 

For simplicity, I use a link from the clone of my git repo.

```sh
$ sudo ln -s `pwd`/symbols/xps /usr/share/X11/xkb/symbols/xps
```

To enable the new layout for the current session use 

```sh
$ setxkbmap xps
```

Try it out, the `PgUp` should now be left arrow button, but you can still page up when you us `Alt+PgUp`. Similarly `PgDn` is now a right arrow and `Alt+PgDn` is the actual page down.

## Persisting the Changes
I have found two suggestion for how to make this setting persist across logins.

1. First, you create/edit your `$HOME/.profile` to add `setxkbmap xps` to the end. This file contains commands that are applied when you session starts.

    ```sh
    $ touch $HOME/.profile
    $ "setxkbmap xps" >> $HOME/.profile
    ```

2. Second, I can not confirm that this works. supposedly, you can modify your settings using `gsettings`

    ```sh
    $ gsettings get org.gnome.desktop.input-sources sources
    [('xkb', 'us')]
    
    $ gsettings set org.gnome.desktop.input-sources sources "[('xkb','xps')]"

    $ gsettings get org.gnome.desktop.input-sources sources
    [('xkb', 'xps')]
    ```

    You can also see and set via `dconf`, but I found `gsettings` easier to use. 

I tried using _just_ option 2, but it didn't work after a reboot. However, it _does_ change my input source in the Settings app to be `xps` instead of `us`. This seems like a good thing, so I have do both (1) and (2). This way, if anything tries to modify the keyboard settings and uses this value, it will get the correct value.


{{<image src="images/language-settings.png" title="New layout shown in the settings." >}}


## References:
Much of this content is consolidated from various pages of documentation and blog posts, including: 

* [A Guide on the XKB Configuration Files](https://www.charvolant.org/doug/xkb/html/node5.html)
* [A simple, humble but comprehensive guide to XKB for linux][damko]
* [Creating a custom xkb layout](https://niklasfasching.de/posts/custom-keyboard-layout/)
* [This AskUbuntu post](https://askubuntu.com/questions/369276/add-remove-keyboard-layout-by-console-command)
* [The Unix StackExchange](https://unix.stackexchange.com/questions/99085/save-setxkbmap-settings)



[damko]: https://medium.com/@damko/a-simple-humble-but-comprehensive-guide-to-xkb-for-linux-6f1ad5e13450