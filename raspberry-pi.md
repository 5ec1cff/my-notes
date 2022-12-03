# 树莓派

从老朋友那以只有邮费的价格淘到了一台 2.5 手的树莓派，至于为何是 2.5 手，因为第二任主人自称从第一任主人那里买来后几乎没有动过。

现在轮到我做它的第 2.5 任主人了，不知道它将来会经历怎么样的命运……

## 第一课：开机和查看配置

Type-C 电源

ssh 默认用户名和密码：`pi@raspberry`

……

## 第二课：关机

[power supply - How do I turn off my Raspberry Pi? - Raspberry Pi Stack Exchange](https://raspberrypi.stackexchange.com/questions/381/how-do-i-turn-off-my-raspberry-pi)

> 1. Execute the command:  
> `sudo shutdown -h now`  
> 2. Wait until the LEDs stop blinking on the Raspberry Pi.  
> 3. Wait an additional five seconds for good measure (optional).  
> 4. Switch off the powerstrip that the Raspberry Pi power supply is plugged into.  

简单来说，直接拔电会对 SD 卡有影响。正确的关机方法：执行 `sudo shutdown -h now` ，然后等待闪烁的灯熄灭（红灯和黄灯都会熄灭），此时就可以拔电了。

## 2022.12.2

### 刷入新系统

考虑到上一任留下的系统不知道会有什么暗坑，以及它还是 32 位的，也不是 latest 的，决定更新系统。

查了一些资料，CPU `BCM2711` 是 armv8 ，支持 64 位。

[Raspberry Pi 4 Model B specifications – Raspberry Pi](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/)

考察了一些资料，可供我选择的系统有官方的 Raspberry PI OS 、Ubuntu MATE 、Ubuntu Server 等。

[用于各种用途的最佳树莓派操作系统 | Linux 中国 - 知乎](https://zhuanlan.zhihu.com/p/141068779)

考虑到这玩意目前只是用来跑一些脚本，甚至不需要图形界面，于是选择 Raspberry PI OS Lite 。

去官方下载 64 位版本，基于 Debian 11 和 Linux 5.15 内核，发布日期 September 22nd 2022。

`2022-09-22-raspios-bullseye-arm64-lite.img`

[Operating system images – Raspberry Pi](https://www.raspberrypi.com/software/operating-systems/#raspberry-pi-os-32-bit)

[Raspberry Pi OS – Raspberry Pi](https://www.raspberrypi.com/software/)

下载好后用官方的工具刷入 SD 卡即可。这个工具甚至还能自动配置 ssh 公钥登录（看上去是从 `~/.ssh/id_rsa.pub` 提取的）和与 PC 相同的 WIFI 配置（它用什么黑科技看到的密码……），我最后选择用 ssh 公钥登录、网线连接。

登录用户名默认是 `pi` ，可以用 `sudo su` 访问 root （无需密码，可能因为我没设置？）。

### 配置用户

在用了一段时间，安装了一些东西之后，我才注意到每次 ssh 都有下面的提示：

```
Please note that SSH may not work until a valid user has been set up.

See http://rptl.io/newuser for details.
```

[An update to Raspberry Pi OS Bullseye - Raspberry Pi](https://www.raspberrypi.com/news/raspberry-pi-bullseye-update-april-2022/)

根据文章所说，现在的 rasp os 引入了新的安全措施，要求首次登录必须设置新用户名和密码，对于 headless ，应该用刷写工具创建新用户。然而我只配置了 ssh 登录，并没有配置用户名和密码。

文章还指出对于已经安装的用户，可以用 `rename-user` 命令来重命名。但是这个命令需要重启，并且启动后似乎不能远程操作。

看了一下那个脚本 `/usr/bin/rename-user` ，会创建一个向导 (wizard) 用户，下次启动会自动启动用户配置向导。

另外里面也有写入 sshd banner 的逻辑（不是很懂为什么写在这里）：

```sh
cat <<- EOF > /etc/ssh/sshd_config.d/rename_user.conf
        Banner /usr/share/userconf-pi/sshd_banner
EOF
```

看起来现在这个状况要配置新用户挺麻烦的，不知道有什么未知后果，反正这个 `pi` 用户也不是不能用。

至于那个 banner ，干脆修改 `/etc/ssh/sshd_config.d/rename_user.conf` 把它注释掉，然后 `systemctl --quiet reload ssh`
