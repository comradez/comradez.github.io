+++
title = "2022 年了，Wayland 准备好了吗？"
date = "2022-03-26 00:20:14"
authors = ["茨月"]
+++

太长不看版：

对于 KDE Plasma & Hi-DPI 用户，没有。

<!-- more -->

# Why Wayland?

前天装修了一下工位，嫖了一台闲置的 1920×1080, 21.5' 显示器，我现在笔记本屏幕是 2880×1800, 14' 的屏幕，而且希望能够组双屏幕协同<del>抄代码真的方便啊XD</del>。

如果分别取到最佳的缩放率，显示器是 100% 缩放，而笔记本屏幕则是 200% 缩放。问题在于，基于 X11 的桌面环境难以实现逐显示器配置缩放率（听 c10s 说这是因为 X.org 实现里面绘制部分就是单个巨大 buffer，不知真假 >_<），只能配置全局缩放率，要么选 100%，笔记本屏幕字变得小到不能看；要么选 200%，显示器被挤爆。

而 Wayland 在设计时就考虑到了多显示器适配，允许为每个显示器配置其自己的分辨率。<del>再加上它是更现代、面向未来的显示协议，切换到 Wayland 也是历史的进程嘛。</del>

# How to Wayland?

对于 Manjaro + KDE 用户，安装 Wayland 桌面环境比较简单：

```
sudo pacman -Sy --needed plasma-wayland-session xorg-xwayland libxcb egl-wayland
```

退出当前 session 之后就可以找到 Wayland session 然后登录了。

# Why *NOT* Wayland？

在 Wayland session 中，我主要遇到了两个问题，而且任何一个都会让系统变得*不可忍受*：

- `xorg-xwayland` 允许运行 X 窗口，但***尚不支持***高分辨率，因此这些窗口在 200% 缩放率下会变得非常糊。
    - Firefox 寄了
    - VS Code 寄了
    - 这俩都寄了之后我就不想再往下试了

- 触控板只能滑动，不能点击。

因为打算集中精力先解决第一个问题，第二个问题我靠插鼠标暂时绕过了。随后发现 aur 和 archlinuxcn 源里都提供了一个叫做 `xorg-xwayland-hidpi-git` 的包，看名字，我们要找的似乎就是它。

然后看一眼 [AUR 留言区](https://aur.archlinux.org/packages/xorg-xwayland-hidpi-git#comment-809202)——

> @modnoob This adds an Xorg extension that needs to be supported by the compositor. The only
> implementation is in wlroots-hidpi-git, so only wlroot-based compositors can use it. AFAIK the only
> compositor using this patches on wlroots is sway-hidpi-git. No chance that Plasma (it's KDE, right?) can
> use these patches.

啊，寄了。

所以触控板问题我都没打算解决，直接连滚带爬地回到了 X.org 快乐老家。

# So...... When Wayland, exactly?

实际上，KDE 早在今年一月发布的 [roadmap](https://pointieststick.com/2022/01/03/kde-roadmap-for-2022/) 中就有提过多显示器支持和 Wayland 完全替代 X11 的*可能性*……好吧，也就是个可能性。

实际上在[这里](https://community.kde.org/Plasma/Wayland_Showstoppers)这个毛病已经被挂了好久了，但是被标记为了“因为等待上游所以被阻塞了”，所以在 xorg-xwayland-hidpi-git 被合并入 xorg-xwayland 主线之前，我们大概是看不到它开工了。基于此，我对今年用上它表示很大的怀疑……

# 那工位问题怎么办？

目前的凑合解决方案是，调全局 100% 缩放，显示器上放 CLion、浏览器（正常字体大小），笔记本屏幕上放 VS Code 和终端（调大字体）。
<del>如果老板看到这个给我安排个 Hi-DPI 显示器呗。</del>

# 意外的解决方案

后来发现 X 可以提供显示器级别的「缩放」的，使用 `xrandr` 命令。

首先将全局缩放率调整到 200% 以保证笔记本显示正常且细腻，随后 `xrandr --output <interface> --scale <scale>` 调整特定输出的缩放。例如我会将它调整到 `1.75x1.75`，这样 X 会认为这台显示器的分辨率是 1.75 倍的 1920×1080——即 3360×1890，然后将输出结果降采样到 1920×1080 显示。

这样的坏处是降采样得到的结果比较糊，但确实是我目前已知的在 X 下让两台显示器都以正常 DPI 工作的唯一办法。<del>还得是 4K</del>