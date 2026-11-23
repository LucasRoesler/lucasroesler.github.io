---
title: "Have your Debian cake and eat it too"
description: "Using unstable packages in Debian"
date: 2023-09-16T00:00:00+02:00
tags:
  - debian
  - firefox
  - apt
  - linux
draft: false
coverImage: images/jordane-mathieu-k7VgPnWFcx8-unsplash.jpg
---

I recently switched to Debian because I wanted a rock solid plain Gnome environment for my daily driver / work laptop. The main downside is that even Debian testing is a bit more conservative with updates than I would like for certain applications. Normally I would use and recommend Flatpak, but there are a few edge cases where I need to use the Debian package, which leads me to today's post. 

---

I am really excited because the the Firefox [finally removed the microphone and camera indicator](https://www.mozilla.org/en-US/firefox/117.0/releasenotes/)!!!!  I really hated their floating indicator and this update brings me a lot of joy. Like I said in the intro,  I would normally install/update [Firefox via Flatpak](https://flathub.org/apps/org.mozilla.firefox), but this doesn't really work for me because I use [1password](https://1password.com/) and I really like the integration of the Firefox extension and the desktop app. This integration throws a small kink into the the install story due to sandboxing in Flatpak not currently [supporitng a native messaging portal](https://1password.community/discussion/121404/issues-with-firefox-native-messaging-need-suggestions) (see [also](https://github.com/flatpak/xdg-desktop-portal/issues/655)).

Fortunately, Debian apt just installs the standalone non-sandboxed Firefox. This means 1password integration works as expected. However, Debian stable and testing only ship the [Firefox Extended Support Release (ESR)](https://support.mozilla.org/en-US/kb/switch-to-firefox-extended-support-release-esr)! This is _not_ the latest. But Debian unstable (ie Sid) *does* package the latest Firefox releases. 

## Manual approach
At first, I just [manually downloaded](https://packages.debian.org/sid/firefox) and installed the `.deb` using `dpkg -i <deb file>`.

This naturally has an issue that I need to manually track and update it when a new version is released.

## Apt approach

But I found a better approach that allows the installation to be updated via apt. Apparently the source list can have multiple overlapping sources _and_ you can set a preference for packages to prefer a specific source!  I found some [nice instructions here](https://www.addictivetips.com/ubuntu-linux-tips/how-to-run-unstable-packages-on-debian-stable/)

First, you need to edit the `/etc/apt/sources.list` to add this section 

```txt
deb http://deb.debian.org/debian/ sid main non-free-firmware non-free contrib
deb-src http://deb.debian.org/debian/ sid main non-free-firmware non-free contrib
```

Next, I created two files in `/etc/apt/preferences.d`

```txt
# /etc/apt/preferences.d/debian
Package: *
Pin: release a=trixie
Pin-Priority: 500

Package: *
Pin: release a=unstable
Pin-Priority: 100
```
This file tells debian to prefer packages in the Testing repository. 

Then I created

```txt
# /etc/apt/preferences.d/firefox
Package: firefox
Pin:release a=unstable  
Pin-Priority: 1000  
```

Which overrides the previous file and tells apt to prefer the Unstable repository for the `firefox` package.

Finally, we can update and install Firefox

```sh
sudo apt update
sudo apt install firefox
```

## Conclusion

I am now running the latest Firefox _and_ have the standard update flow with `apt`. However, I would be very cautious about using this approach with other applications and would certainly avoid using it for any of the core system packages like gnome, systemd, etc. [Don't make a FrankenDebian](https://wiki.debian.org/DontBreakDebian#Don.27t_make_a_FrankenDebian)!

Once the desktop portal for native messaging is finalized, I will just switch to Flatpak. But until then, this is a nice solution for me.


---
Cover Photo by <a href="https://unsplash.com/@mat_graphik?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Jordane Mathieu</a> on <a href="https://unsplash.com/photos/k7VgPnWFcx8?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  