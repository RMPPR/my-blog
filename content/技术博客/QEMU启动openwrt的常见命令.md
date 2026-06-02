启动命令
```
qemu-system-x86_64  -m 512 -smp 2  -drive file=openwrt-24.10.6-x86-64-generic-ext4-combined.img,format=raw  -netdev user,id=hn0,hostfwd=tcp::12222-:22,hostfwd=udp::25353-:5353  -device e1000,netdev=hn0  -nographic
```

使用QEMU启动后配置网络
```
uci set network.lan.proto='dhcp'
uci commit network
/etc/init.d/network restart
```
安装各类包：
```
Opkg update
Opkg install xxx
```
