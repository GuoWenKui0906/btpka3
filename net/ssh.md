

# 生成公钥、私钥

```bash
# 交互模式：没有密码，默认文件位置。
ssh-keygen -t rsa -C "xxx@yyy.com"

# 非交互模式。
ssh-keygen -t rsa -C "xxx@yyy.com" -N 'xxxPass' -f ~/.ssh/id_rsa

# 验证 公钥和私钥是否匹配
diff <( ssh-keygen -y -e -f .ssh/id_rsa ) <( ssh-keygen -y -e -f .ssh/id_rsa.pub )

# 列出 key 的 fingerprint
ssh-keygen -lf ~/.ssh/id_rsa
ssh-keygen -lf ~/.ssh/id_rsa.pub -E md5

# test
ssh -vT git@github.com
```

之后可以把 `～/.ssh/id_rsa.pub` 中的内容追加到 远程ssh服务器用户的 `~/.ssh/authorized_keys` 中。（注意：线上环境不要配置为使用ssh登录）


# sshd 安全性

`vi /etc/ssh/sshd_config`

```ini
X11Forwarding no  #
PermitEmptyPasswords no     # 禁止空密码
MaxStartups  10             # 最多保持多少个未认证的连接，防止SSH拒绝服务
PermitRootLogin no          # 禁止root登录，否则很容易被用来暴力猜解

```

# 代理服务器

## 假设环境

gateway.kingsilk.net 有公网IP，并打算在该主机上搭建 代理 服务器。
internal.kingsilk.net 无法访问公网，打算通过代理服务器配置访问公网上其他 ssh 服务（比如 git clone）

1. 在 internal 主机上分别创建自己的ssh key （这里均是root用户的）

    ```bash
    # root@internal
    ssh-keygen -t rsa -C "root@internal.kingsilk.net"
    ```

1. 在 gateway 主机上创建代理用的用户 proxy, 并授权 root@internal 的 ssh key

    ```bash
    yum install nc

    # proxy@gateway
    vi ~/.ssh/authorized_keys  # 将 `/root/.ssh/id_rsa.pub`@internal 的 内容 追加进去
    ```
1. 在 internal 主机上配置 git 的代理 `vi /root/.ssh/config`

    ```text
    Host gitlab.com
        User                git 
        ProxyCommand        ssh proxy@gateway.kingsilk.net /usr/bin/nc %h %p
        IdentityFile        ~/.ssh/id_rsa
    ```
1. 验证：
    ```bash
    git clone git@gitlab.com:kingsilk/xxx.git /data0/xxx
    ```

# ssh 登录慢

```bash
# ------- for client
vi /etc/ssh/ssh_config
Host *
    GSSAPIAuthentication no
    AddressFamily inet

# ------- for server
vi /etc/ssh/sshd_config
# 修改为：
#GSSAPIAuthentication yes
GSSAPIAuthentication no
#UseDNS yes
UseDNS no
```



# 使用多个密钥

```
# 该命令会生成 ~/.ssh/id_rsa.github 和 ~/.ssh/id_rsa.github.pub
ssh-keygen -t rsa -f ~/.ssh/id_rsa.github  -C "511980432@qq.com"

# 将以下公钥的内容作为你的ssh key 配置到 github 上
cat ~/.ssh/id_rsa.github.pub


# 设置与github通讯时，使用刚刚生成的ssh key
man ssh_config
vi ~/.ssh/config    # 内容见后

# 在github上创建仓库
TuPengXiong.github.io
```


`~/.ssh/config` 内容如下：

```
Host kingsilk
Hostname git.kingsilk.xyz
IdentityFile ~/.ssh/id_rsa
User tpx

Host github
Hostname github.com
IdentityFile ~/.ssh/id_rsa.github
User tpx
```


# windows
1. 安装 [git for windows](https://git-scm.com/download/win)
1. 安装 [tortoisegit](https://tortoisegit.org/)
1. 安装 [putty](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html) 完整版，建议放到环境变量 PATH 下。
1. 通过 'git Bash Here' 打开名控制他，运行 ssh-genken 生成相应的公钥，私钥（默认路径同linux，但个人目录为 `/c/Users/YouName`)
1. 运行 puttygen, Convertions -> import key : 选择刚刚生成的私钥 -> File -> Save private key : 建议保存到 `~/.ssh/id_rsa.ppk`
1. 找一个目录，鼠标右键，git clone, 选中 'Load putty key'并选择刚刚生成的 ppk文件，即可。

# 使用公钥远程登录

# 使用 ~/.ssh/config 简化登录

示例内容如下：

```
Host newcrm.nalashop.com
    User git
    Port 2222
    IdentityFile ~/.ssh/id_rsa
```

最后记得修改权限

```
chmod 600 ~/.ssh/*
```


# SSH HTTP proxy

```
ssh -D ${localSocketProxyPort} user@remoteSShServer
# 然后浏览器配置一下使用 localhost，上面的端口即可
```

# SSH 隧道

[参考](https://help.ubuntu.com/community/SSH_VPN/)


## 场景举例

|        |  `A@dev`       |  `B@prod`     | `C@prod`     |
|--------|----------------|---------------|--------------|
|Inet IP |-               |122.225.11.207 |              |
|Lan IP  |192.168.101.222 |192.168.71.207 |192.168.71.80 |
|service |Redis:6379      |               |MySql:3306    |

* dev 为公司的开发环境，通过ADSL上网，没有固定公网IP
* prod 为公司的线上环境，有固定的公网IP。
* `A@dev` 只可以通过公网IP 122.225.22.222 SSH到 `B@prod`
* `B@prod` 和 `C@prod` 位于同一个内网，可以相互ssh到，但都无法访问到 `A@dev`
* `C@prod` 安装有MySql服务，端口是3306
* `A@dev` 安装有Redis服务，端口是6379


## SSH 动态端口转发
可通过本地特定端口，访问远程所有服务————即代理服务器。

```bash
# 在SSH client端执行 （如果想前台执行，则不要加 -f）
ssh sshUser@sshHost -C -f -N -g -D [localBindIp:]localBindPort
```

需求示例：线上环境同一种web服务有集群，我需要调试特定某个节点上的服务。
关于 xxx_proxy 环境变量设置请参考[这里](https://wiki.archlinux.org/index.php/Proxy_settings)

1. 启动代理转发

    ```bash
    # 在 A@dev 上执行以下命令：
    ssh root@122.225.11.207 -C -f -N -g -D 9999

    # 如果D@dev无法上网，但可以连接到A@dev，则可以在D@dev上通过该代理上网
    # man wget
    export http_proxy=socks5://prod11.kingsilk.net:9999
    export https_proxy=$http_proxy
    export ftp_proxy=$http_proxy
    export no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com"
 
    # man curl
    export http_proxy=socks5://prod11.kingsilk.net:9999
    export HTTPS_PROXY=${http_proxy}
    export FTP_PROXY=${http_proxy}
    export ALL_PROXY=${http_proxy}
    export NO_PROXY=$no_proxy
    curl -x socks5://localhost:9999  http://www.baidu.com
    ```
1. 在 chrome 浏览器中 安装 SwitchySharp 插件，新建 Proxy Profiles：
    1. 设置SOCKS v4连接的IP地址为 localhost, 端口为 9999;
    1. 设置本机IP和本地局域网IP地址不进行代理： `localhost; 127.0.0.1; 192.168.101.* `

1. 临时修改本机 /etc/hosts, 设置域名为线上环境某个IP地址

1. 就可以在chrome 浏览器中使用SwitchySharp的新建的profile通过域名访问了。


PS：不同应用的socks代理设置的方式不同，需要自行阅读相关文档解决。


## SSH 本地端口转发

访问本地特定端口，就是访问远程特定服务。

```bash
# 在SSH client端执行
ssh sshUser@sshHost -C -f -N -g -L [localBindIP:]localBindPort:remoteServiceIP:remoteServicePort
```

需求示例：需要从`A@dev`上直接访问 `C@prod` 的MySql服务


1. 开启端口转发

    ```bash
    # 在 A@dev 上执行
    ssh ssh@122.225.11.207 -C -N -g -L 13306:192.168.71.80:3306 -p 2222
    ```
1. 本地访问远程MySql服务

    ```bash
    # 在A@dev 上执行
    mysql -h 192.168.101.222 -P 13306 -u yourDbUser -p yourDb
    ```

## SSH 远程端口转发
使单向网络中末端可以通过特定端口逆向访问特定服务。
比如：有静态外网IP的网络中的主机 可以访问 无静态外网IP（ASDL）的网络中主机上的服务。


```bash
# 在SSH client端执行
ssh sshUser>@sshHost -C -f -N -g -R [bindIpOnSshClient:]sshBindPortOnSshClient:bindHostOnSshServer:listenPortOnSshServer &
```

需求示例：需要从C@prod上直接访问 A@dev 的Redis服务

1. 开启远程端口反向转发

    ```bash
    # 在 A@dev 上执行
    ssh root@122.225.11.207 -C -N -g -R 192.168.71.207:16379:localhost:6379  -o ExitOnForwardFailure=yes
 
    # 默认是绑定在远程ssh 服务器的 loopback(127.0.0.1) IP地址上。可以通过 
    #      "-R  :16379:localhost:6379"
    #      "-R *:16379:localhost:6379"
    # 的方式来绑定所有IP。但需要 ssh 服务器设置 GatewayPorts 为 yes 配置，详情请参考 man sshd_config
    # vi /etc/ssh/sshd_config
    ```

1. 访问Redis服务

    ```bash
    telnet 192.168.71.207 16379
    key *                            # redis 命令 : 列出所有key
    ```
1. 完整示例脚本

    ```bash
    set password xxx
    spawn ssh ddns@pub-prod11.kingsilk.net -C -N -g -R \
        localhost:14300:localhost:80 \
        -o ExitOnForwardFailure=yes \
        -o ServerAliveInterval=60
    expect "password" {send "$password\r"}
    interact 
    ```

# TODO SSH VPN

参考[1](http://bodhizazen.net/Tutorials/VPN-Over-SSH/)、[2](https://help.ubuntu.com/community/SSH_VPN)
[sshuttle](https://github.com/apenwarr/sshuttle)


# 超时设置

## 方式一：修改 server 端的配置 `vi /etc/ssh/sshd_config`

```ini
ClientAliveInterval 60 ＃server每隔60秒发送一次请求给client，然后client响应，从而保持连接
ClientAliveCountMax 3 ＃server发出请求后，客户端没有响应得次数达到3，就自动断开连接，正常情况下，client不会不响应
```

之后重启sshd服务 `systemctl restart sshd`


## 方式二：修改 client 端的配置 `vi /etc/ssh/ssh_config`

```ini
ServerAliveInterval 60 ＃client每隔60秒发送一次请求给server，然后server响应，从而保持连接
ServerAliveCountMax 3  ＃client发出请求后，服务器端没有响应得次数达到3，就自动断开连接，正常情况下，server不会不响应
```
## 方式三： ssh 连接时，指定参数： `ssh -o ServerAliveInterval=60`



