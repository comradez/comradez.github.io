---
title: Canokey 初体验 && 命令行的私密信息传递
date: 2022-11-05 22:07:40
---

2202 年双十一，我终于买了 Canokey，<del>然后到手玩了一晚上</del>。

# Canokey 配置

关于 Canokey 如何配置，已经有比较多的博客讲过这一点了，我在这里仅仅做简略的记录。如果寻求配置教程，可以考虑下文「参考链接」一节的各个教程。

## GPG 配置

我之前的 GPG key 只有一个主密钥，配置也比较烂，索性直接 revoke 掉换新的。

首先在 [Tails 官方网站](https://tails.boum.org/)下载 Tails 的镜像，并且丢进 Ventoy 然后 boot. 

> 使用 Tails 是因为它是一个专门为隐私保护特化的发行版，默认禁用了 Tor 以外的网络连接，预装 `gpg`, `paperkey`, `ccid` 等用于密钥生成、导出、智能卡写卡等的实用工具，而且每次抹除所有数据。

我用不到网络功能，所以在 Additional Settings 里面把网络连接全部关闭；同时因为我要在一个 luks U 盘分区上保存我的主密钥，我还启用了 root password.

### 以下流程在 Tails 中完成

首先生成 GPG key

```bash
gpg --expert --full-gen-key
```

类型选 (11) ECC (Set your own capabilities)

启用功能设置为认证 (Certify)

曲线选择 Curve 25519

有效期限设置为 0(key does not expire)

随后

```bash
gpg --quick-add-key <fingerprint> cv25519 encr 2y
gpg --quick-add-key <fingerprint> ed25519 sign 2y
gpg --quick-add-key <fingerprint> ed25519 auth 2y
```

添加三把子密钥，然后将公钥、公钥 revoke 证书、主私钥和三把子私钥分别导出：

```bash
gpg -ao pub-key.pub --export <fingerprint>  # 导出公钥
gpg -ao main-key.asc --export-private-key <fingerprint>!  # 导出主私钥，注意加 ! 以保证只导出主密钥而不导出子密钥
gpg -ao encr-key.asc --export-private-key <fingerprint>!  # 导出加密子私钥
gpg -ao sign-key.asc --export-private-key <fingerprint>!  # 导出签名子私钥
gpg -ao auth-key.asc --export-private-key <fingerprint>!  # 导出验证子私钥
# revoke 证书在生成主密钥后会自动生成，在输出的路径下寻找即可
```

然后 mount 我的 luks U 盘分区：

```bash
sudo cryptsetup luksOpen /dev/sdb myusb
mkdir myusb
sudo mount /dev/mapper/myusb myusb
```

过程中会需要输入 root 密码和分区的加密口令，完成后将主私钥和 revoke 证书一起存放进去，然后 umount 并拔出：

```bash
sudo umount myusb
sudo cryptsetup close myusb
```

然后 mount 我的另一块 U 盘，保存子私钥、revoke 证书和公钥。

随后将密钥导入 Canokey：

```bash
gpg --edit-key <fingerprint>
```

记得每个 key 选择正确的 slot，而且因为要用两个 key，所以第一个 key 操作完成后不要保存，直接 Ctrl-C 退出 gpg，插入第二把 key 导入完再保存，默认的 PIN 和 AdminPIN 分别为 `123456` 和 `12345678`，建议修改。这个时候也可以设置持卡人姓名等信息。

完成之后 reboot.

### 以下流程在正常工作环境中完成

首先从 U 盘中拷贝出公钥，上传到 [keyserver](https://keys.openpgp.org) 并且导入 gpg：

```bash
gpg --import pub-key.pub
```

然后插入 key，将其中的私钥 fetch 到本地

```bash
gpg --edit-card
(一大堆输出)
gpg/card> fetch
```

理论上此时就可以愉快玩耍了，我们可以简单地测试一下：

```bash
echo "茨月❤RaRa" | gpg -r "Chris Zhang" -e | gpg -d 
```

输入 PIN 之后触摸 Canokey，看到输出与 echo 内容一致即可。

## U2F/FIDO 配置

这个配置主要用于部分网站 & OpenSSH key 的生成。

首先安装 `libfido2`，提供 `fido2-token` 工具。

```bash
fido2-token -L
```

查看自己 Canokey 的设备，然后

```bash
fido2-token -S <device>
```

设置 PIN. <del>其实你现在不设置的话在网上添加的时候也会让你设置的</del>

OpenSSH 8.2 及之后支持了 `ecdsa-sk` 和 `ed25519-sk` 这两种支持 FIDO2 的 SSH 密钥生成，插入 Canokey 后输入

```bash
ssh-keygen -t ed25519-sk -c "Your Email"
```

即可，相比一般的 ssh-keygen 流程仅仅多了一步 PIN 输入和 key 触摸。

# 那命令行的私密信息传递是怎么回事？

之前为了自己在设备间传送配置文件方便，我写了一个在线剪贴板，包括[一个 Actix-Web 后端](https://github.com/comradez/paste-server)、[一个 Yew 网页前端](https://github.com/comradez/paste-frontend)和[一个 clap 命令行前端](https://github.com/comradez/paste-client)。

目前网页前端部署在了 [https://paste.zcy.moe](https://paste.zcy.moe), 后端 API 在 [https://api.zcy.moe](https://api.zcy.moe), 可以试着玩一玩 :)

这样一来，私密信息的传输可以简化为

```bash
echo <Your Message> | gpg -ar <Your Recepient> -e | pb send
```

接收则是

```bash
pb get <Token> | gpg -d
```

<del>实际上这套工具长期没人用，但是只有自己用也是挺方便的</del>