+++
title = "It's year 2022. Is Wayland ready for use?"
date = "2022-03-26 00:20:14"
authors = ["茨月"]
+++

TL;DR:

For KDE Plasma & Hi-DPI users, no.

<!-- more -->

# Why Wayland?

The day before yesterday, I redecorated my workstation and borrowed an idle 1920×1080, 21.5' monitor. My laptop screen is 2880×1800, 14', and I wanted to set up dual-screen collaboration <del>copying code is really convenient XD</del>.

If each screen is set to its optimal scaling rate, the monitor should be at 100% scaling, while the laptop screen should be at 200% scaling. The problem is that X11-based desktop environments struggle to configure scaling rates per display (I heard from c10s that this is because the drawing part in X.org’s implementation is a single large buffer, not sure if it’s true >_<). You can only set a global scaling rate—either 100%, making the text on the laptop screen too small to read, or 200%, causing the monitor to overflow.

Wayland, on the other hand, was designed with multi-monitor support in mind, allowing each display to have its own resolution. <del>Plus, it’s a more modern, future-oriented display protocol, so switching to Wayland is also part of the historical process.</del>

# How to Wayland?

For Manjaro + KDE users, installing the Wayland desktop environment is relatively simple:

```
sudo pacman -Sy --needed plasma-wayland-session xorg-xwayland libxcb egl-wayland
```

After logging out of the current session, you can select the Wayland session and log in.

# Why *NOT* Wayland?

In the Wayland session, I encountered two major issues, and either one makes the system *unbearable*:

- `xorg-xwayland` allows running X windows, but it ***does not yet support*** high DPI, so these windows become very blurry at 200% scaling.
    - Firefox is broken
    - VS Code is broken
    - After these two broke, I didn’t want to test further

- The touchpad only supports scrolling, not clicking.

Since I wanted to focus on solving the first issue, I temporarily bypassed the second one by plugging in a mouse. I then found a package called `xorg-xwayland-hidpi-git` in the aur and archlinuxcn repositories. Judging by the name, this seemed to be what we were looking for.

Then I checked the [AUR comments section](https://aur.archlinux.org/packages/xorg-xwayland-hidpi-git#comment-809202)—

> @modnoob This adds an Xorg extension that needs to be supported by the compositor. The only
> implementation is in wlroots-hidpi-git, so only wlroot-based compositors can use it. AFAIK the only
> compositor using these patches on wlroots is sway-hidpi-git. No chance that Plasma (it's KDE, right?) can
> use these patches.

Ah, it’s broken.

So I didn’t even bother trying to solve the touchpad issue and quickly rolled back to the X.org happy place.

# So...... When Wayland, exactly?

Actually, KDE mentioned the possibility of multi-monitor support and Wayland fully replacing X11 in their [roadmap](https://pointieststick.com/2022/01/03/kde-roadmap-for-2022/) released in January this year... Well, it’s just a possibility.

In fact, this issue has been hanging around [here](https://community.kde.org/Plasma/Wayland_Showstoppers) for a long time, but it’s marked as "blocked due to waiting for upstream." So, until `xorg-xwayland-hidpi-git` is merged into the mainline `xorg-xwayland`, we probably won’t see any progress. Based on this, I’m very skeptical about using it this year...

# What about the workstation issue?

The current workaround is to set the global scaling to 100%, place CLion and the browser (with normal font size) on the monitor, and place VS Code and the terminal (with enlarged fonts) on the laptop screen.
<del>If the boss sees this, maybe they’ll get me a Hi-DPI monitor.</del>

# An Unexpected Solution

Later, I discovered that X can provide display-level "scaling" using the `xrandr` command.

First, adjust the global scaling rate to 200% to ensure the laptop display is normal and sharp. Then, use `xrandr --output <interface> --scale <scale>` to adjust the scaling for a specific output. For example, I set it to `1.75x1.75`, so X thinks the monitor’s resolution is 1.75 times 1920×1080—i.e., 3360×1890—and then downscales the output to 1920×1080 for display.

The downside is that the downscaled result is a bit blurry, but it’s the only way I know of to make both displays work at normal DPI under X. <del>Still, 4K is the way to go.</del>