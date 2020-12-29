---
title: "Centos配置GoogleAuthenticator动态密钥进行ssh二次验证"
date: 2020-12-29T16:44:44+08:00
tags: [Linux]
---
### 安装依赖
```bash
yum list | grep google-authenticator
yum install google-authenticator
yum install qrencode
```
### 配置Google Authenticator
安装完直接跑下面的命令进行配置，注意只在当前用户生效
```
> google-authenticator
```
之后会需要确认几点信息
```
Do you want authentication tokens to be time-based (y/n) y
```
是否配置基于时间的动态密钥，选择y，之后会出现超级大一个二维码，下面还会有一些小字,这里的key就是用于配置手机端app的，我们先保存下来，不用慌，因为这个key随时都可以查得到.
```
Do you want me to update your "/root/.google_authenticator" file? (y/n) y
```
是否将配置信息更新到自己家目录，选择y进行更新，这个文件里面就保存着上面的key信息，以防后续还有新的手机设备需要用到key
```
Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y
```
是否禁止同一密钥在30秒内被多次使用，如果想要更安全就选择y，如果想要更方便就选择n
```
By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server,
we allow an extra token before and after the current time. This allows for a
time skew of up to 30 seconds between authentication server and client. If you
experience problems with poor time synchronization, you can increase the window
from its default size of 3 permitted codes (one previous code, the current
code, the next code) to 17 permitted codes (the 8 previous codes, the current
code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
between client and server.
Do you want to do so? (y/n) n
```
是否允许前8次和后8次的动态密钥也有效，如果客户端和手机端都是基于网络的时间同步，选择n提高安全性

```
If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting? (y/n) y
```
是否限制30秒内最多3次尝试，为了防止恶意试错，选择y
这样服务端的Google Authenticator就配置完毕。下面做一些系统设置，使上面的配置用作ssh。

### 配置pam

```bash
vim /etc/pam.d/sshd
auth required pam_google_authenticator.so nullok
vim /etc/ssh/sshd_config
UsePAM yes
PasswordAuthentication no
ChallengeResponseAuthentication yes
sshd -t
systemctl restart sshd
```
需要注意的是必须设置PasswordAuthentication no（禁用密码登陆，但是并非必须使用公钥），否则二次验证无法使用，会报如下错误：
```
sshd[2690384]: fatal: PAM: pam_setcred(): Permission denied
```
### 手机设置

推荐使用 [andotp](https://github.com/andOTP/andOTP) 这个APP，扫码添加即可。

如果换了手机也很容易，登录到服务器，找到~/.google_authenticator文件，里面会有之前保存的key，重新在新手机进行添加即可。

### Xshell登录验证
下面还是正常ssh登陆服务器，不过输入完用户名以后只能选择交互键盘，依次输入密码和OTP验证码即可。关于登录的一些报错都在/var/log/secure这个日志文件中，不管是什么场景登陆失败都可以先查看下失败日志，对症下药。

注意保持时间同步。
直接使用ntpdate即可，国内可以使用国家授时中心的地址
```
ntpdate -u 210.72.145.44
```
基本上服务器和手机都是用的网络时间就不太会有时间同步的问题。

refer:[https://blog.csdn.net/Victor2code/...](https://blog.csdn.net/Victor2code/article/details/107215820)
