---
title: 使用 wpa_supplicant 命令行连接无线网络
date: 2022-03-15 15:52:09
---

# 背景

几天前，我找出了吃灰两年有余的 Raspberry Pi 4B（4G RAM 版本）打算搭个 MC 服务器玩，两年前的系统早就陈旧不堪了，而且我也不记得当时的用户名密码了，索性重装系统。

一开始装了 Raspberry Pi OS（也就是原来的 Raspbian），基于 Debian 的官方发行版。但我很快发现这个发行版的依赖管理简直魔幻——试图 ntpdate 对时，发现缺少 ntp；而尝试 `sudo apt install ntp` 时，则发现它试图把我的整个桌面环境都一起扬了。想安装一个 neofetch（纯 sh 脚本）竟然要移除预装的 git……经历了这两次毒打之后，年轻的茨月不堪重负，决定把系统换成 Archlinux on ARM。

Archlinux 本身不包含任何桌面环境，因此必须通过命令行连接无线网络。

# 流程

## WPA_SUPPLICANT

遇事不决，先查 Arch Wiki，可以找到[这个页面](https://wiki.archlinux.org/title/Network_configuration/Wireless)。因为校园网是 WPA-EAP(PEAP)，查阅对应的条目可以找到[wpa_supplicant的使用页面](https://wiki.archlinux.org/title/Wpa_supplicant)。

首先查看 interface 的名称：

```sh
ip a
```

树莓派 4B 的无线网卡叫 `wlan0`。

wpa_supplicant 一般会以 daemon 的形式在后台常驻，并使用 config 文件里的配置连接无线网。默认的配置文件路径为 `/etc/wpa_supplicant/wpa_supplicant.conf`。

因此可以首先编辑这个文件，把这些东西写入：

```
ctrl_interface=/run/wpa_supplicant
update_config=1
```

接下来，如果只是简单的 WPA-PSK（即无需额外服务器进行验证的 WPA 个人无线网），则可以跟随页面上的教程使用 wpa_cli 交互式填写 SSID 和 Passphrase，或者用 wpa_passphrase 生成配置。但问题在于，我们的校园网是更复杂的 WPA-EAP（也即 WPA 企业），具体的配置方法在页面上并没提及。

首先查 man，然后查官网，直到查到了[这里](https://w1.fi/cgit/hostap/plain/wpa_supplicant/wpa_supplicant.conf)，找到其中 network block 一部分。

大概在 network block 里这样填写即可：

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

注意一个问题：如果你的网络提供商像英明的清华大学信息化中心一样采用 WPA-EAP peap 但不采用 CA 证书，`ca_cert` 字段**直接不填写**，而不是填写空字符串。

填写完成之后，启动 wpa_supplicant 作为 daemon：

```sh
sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
```

## DHCPCD 获取 ip

```sh
sudo dhcpcd wlan0
```

如果没有问题，则应当显示获取了 ip。用 `ifconfig` 可以看到 wlan0 下出现了 ip。

# 自动化

每天晚上寝室都会断电，因此能在开机时自动连网最好。同样查阅上文所述的 Arch Wiki 页面，发现 systemd 已经自带了对应的脚本，只需要 enable 即可：

```sh
sudo systemctl enable wpa_supplicant@wlan0.service
sudo systemctl enable dhcpcd@wlan0
```

也可以不启动 wpa_supplicant 的服务，而是给 dhcpcd 的服务加上[钩子](https://wiki.archlinux.org/title/Dhcpcd#10-wpa_supplicant)。