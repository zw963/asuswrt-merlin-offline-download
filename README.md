# 使用华硕 merlin 假设离线下载

Billy.Zheng 2016/07/26

本文是基于华硕(ASUS) RT-AC66U MIPS 架构的路由器, 同样适用于 ARM 架构的 RT-AC68U 及更高级版本。
本教程使用的 asuswrt-merlin 版本为 Firmware:380.59，请不要低于这个版本, 

本文章的部署策略稍作修改，应该也适用于 OpenWRT 及其他系统, 因为思路是一样的。

警告:

1. 本文完全基于命令行操作，无任何 GUI 支持, 你需要一定的 CLI 操作能力，以及配置 SSH 登陆的能力 (见下述)
2. 请刷官方版的 asuswrt-merlin, 版本不早于 Firmware:380.59，否则 dnsmasq 可能无法支持 ipset, 你需要自己度娘搞定。
   并且开启远程 SSH 登陆。
3. 自动安装脚本需要 ssh, scp 命令支持，这在 Linux 下完全不是问题，如果是 Windows 下，请参考一键部署脚本自行解决。

## 部署

### 初始化 jffs.

插入一个 U 盘到路由器，自己想办法确保这个 U 盘是 ext3 分区格式，在此不详述，自行度娘解决。

登陆路由器 => 系统管理 => 系统设置 => Format JFFS partition at next boot 点 `是`。
登陆路由器 => 系统管理 => 系统设置 => Enable JFFS custom scripts and configs 点 `是`。
登陆路由器 => 系统管理 => 系统设置 =>  Enable SSH 选择 `LAN+WAN`。
登陆路由器 => 系统管理 => 系统设置 =>  SSH Authentication key, 加入你的 public key, 请自行度娘解决。

应用本页面设置，等待提示完成后, 然后务必重新启动路由器。
等待重新启动再次进入路由器本页面，确保 `Format JFFS partition at next boot` 已经恢复成 `否`.

假设 192.168.1.1 是你的路由 IP, 尝试使用客户端的 ssh 工具登陆路由器。(Windows 下如何做，请自行度娘解决)

```sh
$ ssh admin@192.168.1.1
```

如果登陆成功，出现命令提示符, 键入 `entware-setup.sh` 来初始化包管理系统 opkg.

```sh
admin@RT-AC66U-20F0:/tmp/home/root# entware-setup.sh
```

如果你的 U 盘分区格式没问题，这个脚本会出现类似如下提示： 选择 1 即可。

```sh
admin@RT-AC66U-20F0:/tmp/mnt/sda/asusware/etc# entware-setup.sh
 Info:  This script will guide you through the Entware installation.
 Info:  Script modifies "entware" folder only on the chosen drive,
 Info:  no other data will be changed. Existing installation will be
 Info:  replaced with this one. Also some start scripts will be installed,
 Info:  the old ones will be saved on Entware partition with name
 Info:  like /tmp/mnt/sda1/jffs_scripts_backup.tgz

 Info:  Looking for available partitions...
[1] --> /tmp/mnt/sda
 =>  Please enter partition number or 0 to exit
[0-1]: 
```

请务必确保 opkg 工具可用的情况下再进入下一步，如果出现问题，重启后再次执行 entware-setup.sh.

### clone 项目到本地。

首先克隆项目到本地.

```sh
$ git clone https://github.com/zw963/asuswrt-merlin-offline-download.git ~/
```

### 修改配置文件

然后，进入项目内的 route/opt/etc/nginx, 修改 nginx.conf。(如果你知道为什么要这么做)
这一个步骤非必须，如果保持默认，部署完成后，在浏览器访问 192.168.1.1:8080 将会看到离线下载管理页面。
(你需要替换 192.168.1.1 为你的路由器 IP 地址)

注意：确保 local_address 设定为你的路由器 ip 地址。
创建 ss 配置文件完毕后，进入下一步。

### 为 aria2c 指定一个新的 token.

查看部署脚本 `aria2+YAAW+nginx` 的内容，找到 ``replace_string Passw0rd '你的密码' /opt/etc/aria2.conf``
将 `你的密码` 替换为一个新的密码。稍后，访问 192.168.1.1:8080 时，你需要以下面的形式设定告诉 YAAW 去找到
aria2c daemon. ``http://token:你的密码@192.168.1.1:6800/jsonrpc``


### 运行一键部署脚本进行部署. 

请尽量采用 public key 免密码方式通过访问你的路由器, 否则这个脚本会停下来多次让你输入 ssh 密码，不胜其烦。
即：网页中，SSH Authentication key 部分加入你的 public key, 具体使用请自行度娘解决。

当然，如果你不放心，完全可以选择照着[部署脚本](https://github.com/zw963/asuswrt-merlin-offline-download/blob/master/aria2%2BYAAW%2Bnginx)逐条自行复制粘帖即可。
相信我，部署脚本真的很简单，而且添加了大量的注释，配合 route 目录下的文件，看看就应该懂。

假设我的路由器 ip 地址是 192.168.1.1, 并且开放了 22 的 ssh 端口, 则运行以下命令即可。

```ssh 
./aria2+YAAW+nginx admin@192.168.1.1
```

脚本如果如果未出错，执行完后，尝试访问 192.168.1.1:8080, 应该出现了默认的 YAAW 界面了。
此时，你应该可以看到 `Error: Unauthorized` 提示。进入下一步。


### 设定 YAAW

找到 Settings 按钮， 修改默认的 JSON-RPC Path 为 ``http://token:你的密码@192.168.1.1:6800/jsonrpc``,
然后享受离线下载的乐趣吧。

注意：留意 JFFS 分区上的容量，最好找个大容量的 U 盘进行挂载。2G 的 U 盘实在是不够用。

## 基本思路

1. aria2c 做离线下载器.
2. 使用 nginx 做 Web 服务器，提供 192.168.1.1:8080 访问。
3. 使用 YAAW 做 aria2c 前端，这个东西尤其适合放到路由器上玩。

## 感谢

[YAAW](https://github.com/shadowsocks/shadowsocks-libev)

[aria2](https://aria2.github.io)

[nginx](http://nginx.org)

[asuswrt-merlin](https://github.com/RMerl/asuswrt-merlin)

[Entware-ng](https://github.com/Entware-ng/Entware-ng)

## 其他
[使用华硕 merlin 架设透明代理](https://github.com/zw963/asuswrt-merlin-transparent-proxy)
