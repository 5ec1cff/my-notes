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