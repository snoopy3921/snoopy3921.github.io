---
layout: default
title: "00 Our first driver"
parent: Linux device driver on Luckfox
nav_order: 0
nav_exclude: false
search_exclude: false
has_children: false
has_toc: false
---
## 00 Our first driver
Connect board to wifi: [Link](https://wiki.luckfox.com/Luckfox-Pico/Luckfox-Pico-Ultra-W-WIFI#wifi)
```bash
# Obtain IP of our board
[root@luckfox root]# ifconfig
eth0      Link encap:Ethernet  HWaddr CE:F8:FD:DC:C8:57
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
          Interrupt:52

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:220 errors:0 dropped:0 overruns:0 frame:0
          TX packets:220 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:16172 (15.7 KiB)  TX bytes:16172 (15.7 KiB)

usb0      Link encap:Ethernet  HWaddr CE:68:80:57:DC:80
          inet addr:172.32.0.93  Bcast:172.32.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:982 errors:0 dropped:359 overruns:0 frame:0
          TX packets:87 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:173097 (169.0 KiB)  TX bytes:17498 (17.0 KiB)

wlan0     Link encap:Ethernet  HWaddr 38:54:39:03:12:53
          inet addr:192.168.1.33  Bcast:192.168.1.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:39489 errors:0 dropped:459 overruns:0 frame:0
          TX packets:61824 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:6407913 (6.1 MiB)  TX bytes:13306196 (12.6 MiB)
```
So now we know the IP of out board is wlan0: ``192.168.1.33``

```bash
# To build ko driver 
make build-00 
# To clean ko driver 
make clean-00 
# To upload ko driver to our board 
make upload-00 
```

Now at the console of our board we'll see that file ofd.ko is already there.
```
[root@luckfox root]# pwd
/root
[root@luckfox root]# ls
ofd.ko
[root@luckfox root]# insmod ofd.ko
[root@luckfox root]# lsmod | head -5
Module                  Size  Used by    Tainted: G
ofd                      820  0
aic8800_btlpm           1719  0
aic8800_fdrv          272882  0
aic8800_bsp            46005  2 aic8800_btlpm,aic8800_fdrv
[root@luckfox root]# dmesg | tail -5
[ 4249.861441] Alvida: ofd unregistered
[ 4253.320727] Namaskar: ofd registered
[ 4324.912839] Alvida: ofd unregistered
[ 4357.466605] Namaskar: ofd registered
[ 4403.473862] Alvida: ofd unregistered
[root@luckfox root]#
```
#### Summing up
1. ``insmod`` is used to load the driver into the kernel
2. ``rmmod`` is used to unload the driver 
3. ``lsmod`` prints a list of all the loaded kernel modules.