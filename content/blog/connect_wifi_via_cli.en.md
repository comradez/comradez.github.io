+++
title = "Connect Wi-Fi in CLI via wpa_supplicant"
date = "2022-03-15 15:52:09"
authors = ["茨月"]
+++

This post discusses how to use the `wpa_supplicant` command line to control a Raspberry Pi connecting to Wi-Fi.

<!-- more -->

# Background

A few days ago, I dug out my Raspberry Pi 4B (4G RAM version), which had been gathering dust for over two years, to set up an MC server for fun. The system from two years ago was outdated, and I couldn’t remember the username and password, so I decided to reinstall the system.

Initially, I installed Raspberry Pi OS (formerly Raspbian), the official Debian-based distribution. However, I quickly discovered that the dependency management of this distribution was downright bizarre—when I tried to use `ntpdate` to sync the time, I found that `ntp` was missing. When I attempted `sudo apt install ntp`, it tried to remove my entire desktop environment. Trying to install `neofetch` (a pure shell script) required removing the pre-installed `git`... After these two beatings, the young Ciyue couldn’t take it anymore and decided to switch the system to Archlinux on ARM.

Archlinux itself doesn’t include any desktop environment, so connecting to a wireless network must be done via the command line.

# Process

## WPA_SUPPLICANT

When in doubt, consult the Arch Wiki. I found [this page](https://wiki.archlinux.org/title/Network_configuration/Wireless). Since the campus network uses WPA-EAP (PEAP), I checked the corresponding entry and found the [wpa_supplicant usage page](https://wiki.archlinux.org/title/Wpa_supplicant).

First, check the name of the interface:

```sh
ip a
```

The wireless network card on the Raspberry Pi 4B is called `wlan0`.

`wpa_supplicant` usually runs as a daemon in the background and connects to the wireless network using the configuration in the config file. The default configuration file path is `/etc/wpa_supplicant/wpa_supplicant.conf`.

So, first, edit this file and write the following:

```
ctrl_interface=/run/wpa_supplicant
update_config=1
```

Next, if it’s just a simple WPA-PSK (i.e., WPA personal wireless network that doesn’t require additional server authentication), you can follow the tutorial on the page to use `wpa_cli` to interactively fill in the SSID and Passphrase, or use `wpa_passphrase` to generate the configuration. However, the problem is that our campus network uses the more complex WPA-EAP (i.e., WPA enterprise), and the specific configuration method isn’t mentioned on the page.

First, I checked the man page, then the official website, until I found [this](https://w1.fi/cgit/hostap/plain/wpa_supplicant/wpa_supplicant.conf), specifically the network block section.

You can fill in the network block like this:

```
network={
    SSID="<The network SSID>"
    key_mgmt=WPA-EAP
    eap=PEAP
    phase2="auth=MSCHAPV2"
    identity="<username used for login>"
    password="<password>"
}
```

Note one thing: If your network provider, like the wise Tsinghua University Information Technology Center, uses WPA-EAP PEAP but doesn’t use CA certificates, **do not fill in** the `ca_cert` field at all, rather than leaving it as an empty string.

After filling this out, start `wpa_supplicant` as a daemon:

```sh
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
```

## DHCPCD to Obtain an IP

```sh
sudo dhcpcd wlan0
```

If there are no issues, it should show that an IP has been obtained. Using `ifconfig`, you should see an IP address under `wlan0`.

# Automation

The dormitory loses power every night, so it’s best to automatically connect to the network on startup. Consulting the aforementioned Arch Wiki page, I found that systemd already includes the corresponding scripts, so you just need to enable them:

```sh
sudo systemctl enable wpa_supplicant@wlan0.service
sudo systemctl enable dhcpcd@wlan0
```

Alternatively, you can avoid starting the `wpa_supplicant` service and instead add a [hook](https://wiki.archlinux.org/title/Dhcpcd#10-wpa_supplicant) to the `dhcpcd` service.
