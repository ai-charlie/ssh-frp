# 内网穿透

[https://blog.csdn.net/qq_44577070/article/details/121893406](https://blog.csdn.net/qq_44577070/article/details/121893406)

搭建一个完整的frp服务链需要：

1. VPS一台（也可以是具有公网IP的实体机）
2. 访问目标设备（就是你最终要访问的设备）
3. 简单的Linux基础（如果基于Linux配置的话）

我这里使用了腾讯云服务器作为服务端（Ubuntu Server 20.04 LTS）、本地Linux虚拟机作为要访问的客户端（Ubuntu 20.04 LTS）和本地Xshell（用于远程连接客户端）进行测试。

最后是想把实验室的电脑进行[内网穿透](https://so.csdn.net/so/search?q=%E5%86%85%E7%BD%91%E7%A9%BF%E9%80%8F&spm=1001.2101.3001.7020)实现 ssh 远程访问。

流程很简单，服务端和客户端都下载 frp 配置文件，分别修改配置（地址、[端口映射](https://so.csdn.net/so/search?q=%E7%AB%AF%E5%8F%A3%E6%98%A0%E5%B0%84&spm=1001.2101.3001.7020)），然后启动运行即可。

## 服务端配置

下载 frp 的安装包（[GitHub连接在此](https://github.com/fatedier/frp/releases)），**注意服务端和客户端要同一版本**。我这里是64位Linux，最新版是0.38.0，根据需要选择即可，速度太慢自己想办法，可以考虑本地下了通过ftp传到服务器。

```bash
wget https://github.com/fatedier/frp/releases/download/v0.38.0/frp_0.38.0_linux_amd64.tar.gz
```

解压，文件夹改名为frp，方便使用。腾讯云可能把这玩意当作木马，去控制台信任即可。

```bash
tar -zxvf frp_0.38.0_linux_amd64.tar.gz
cp -r frp_0.38.0_linux_amd64 frp
cd frp
```

看一下文件夹下的内容：

```
ubuntu@VM-0-10-ubuntu:~/frp$ ls -a
.  ..  frpc  frpc_full.ini  frpc.ini  frps  frps_full.ini  frps.ini  LICENSE  systemd

```

我们只需要关注如下几个文件

- frps
- frps.ini
- frpc
- frpc.ini

前两个文件（s结尾代表server）分别是服务端程序和服务端配置文件，后两个文件（c结尾代表client）分别是客户端程序和客户端配置文件。因为我们正在配置服务端，可以删除客户端的两个文件（删不删无所谓），然后修改frps.ini文件。

这里 vim 命令就不详解了，编辑和保存不会可以搜一下。

```
rm frpc
rm frpc.ini

vim frps.ini
```

`frps.ini`文件内容如下：

```bash
[common]
bind_port = 7000
```

`bind_port`表示用于客户端和服务端连接的端口，默认绑定 7000 端口，云服务器注意打开相应端口。

其实还有其他参数可选配置实现其他功能，这里只实现[ssh](https://so.csdn.net/so/search?q=ssh&spm=1001.2101.3001.7020)远程访问，故不作讨论。有需要可以看 `frps_full.ini` 文件，里面有详细说明。

编辑完成后即可保存，运行服务端应用：

```
./frps -c frps.ini
```

如下则成功运行：

```
2021/12/12 00:32:59 [I] [root.go:200] frps uses config file: frps.ini
2021/12/12 00:32:59 [I] [service.go:192] frps tcp listen on 0.0.0.0:7000
2021/12/12 00:32:59 [I] [root.go:209] frps started successfully
```

此时的服务端仅运行在前台，如果 `Ctrl+C` 停止或者关闭 SSH 窗口后，frps 均会停止运行，因而我们使用 [nohup 命令](https://www.runoob.com/linux/linux-comm-nohup.html)将其运行在后台。nohup命令可以让你的shell命令忽略SIGHUP信号，即可以使之脱离终端运行；“&”可以让你的命令在后台运行。

```
nohup ./frps -c frps.ini &

```

此时可先使用Ctrl+C关闭nohup，frps依然会在后台运行，使用jobs命令查看后台运行的程序`jobs`在结果中我们可以看到frps正在后台正常运行：

```
[1]+  Running                 nohup ./frps -c frps.ini &

```

## 客户端配置

以同样的方式下载 frp

```bash
wget https://github.com/fatedier/frp/releases/download/v0.38.0/frp_0.38.0_linux_amd64.tar.gz
tar -zxvf frp_0.38.0_linux_amd64.tar.gz
cp -r frp_0.38.0_linux_amd64 frp
cd frp
vim frpc.ini

```

`frpc.ini`文件内容如下：

```bash
[common]
server_addr = xxx.xxx.xx.xx
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000

```

`common`字段下的三项即为服务端的设置。

`server_addr`为服务端IP地址，自行更改。

`server_port`为服务器端口，填入你设置的端口号即可，如果未改变就是7000。

`[xxx]`表示一个规则名称，自己定义，便于查询即可。这里用于 ssh 所以命名为 [ssh] 了。

`type`连接类型，默认为 tcp，如有需要请自行查询 frp 手册。

`local_ip` 本地 IP

`local_port` 用于 ssh 的端口号，默认 22

`remote_port` 映射的服务端端口，访问该端口时默认转发到客户端的 22 端口，不同的客户端设置不同的端口号。

如果我还想通过 vnc 访问远程桌面，就可以在`frpc.ini`文件中加入 vnc 端口映射（默认为5900端口）

```bash
[vnc]
type = tcp
local_ip = 127.0.0.1
local_port = 5900
remote_port = 5900
```

配置好后可以使用同样的方法后台运行客户端程序：

```
nohup ./frpc -c frpc.ini &
```

# 开机自启动

https://github.com/fatedier/frp/issues/176

**客户端**

```bash
sudo vim /etc/systemd/system/frpc.service
```

在文件中输入

```bash
[Unit]
Description=frpc daemon
After=syslog.target  network.target
Wants=network.target

[Service]
Type=simple
ExecStart=/home/zlq/frp/frpc -c /home/zlq/frp/frpc.ini
Restart= always
RestartSec=1min
ExecStop=/usr/bin/killall frpc

[Install]
WantedBy=multi-user.target
```

```bash
cd /home/zlq/frp && nohup ./frpc -c frpc.ini &
```

**服务端**

```bash
sudo vim /etc/systemd/system/frps.service
```

在文件中输入

```bash
[Unit]
Description=frpc daemon
After=syslog.target  network.target
Wants=network.target

[Service]
Type=simple
ExecStart=/root/frp/frps -c /root/frp/frps.ini
Restart= always
RestartSec=1min
ExecStop=/usr/bin/killall frps

[Install]
WantedBy=multi-user.target
```

### systemctl

```bash
# 修改 frpc.service, frps.service 文件后
sudo systemctl daemon-restart
# 客户端
sudo systemctl enable frpc.service
sudo systemctl restart frpc.service
sudo systemctl stop frpc.service
sudo systemctl status frpc.service
# 服务端
sudo systemctl enable frps.service
sudo systemctl restart frps.service
sudo systemctl status frps.service
sudo systemctl status frps.service
```

## 测试

启动完成后就可以通过 ssh 连接到[内网](https://so.csdn.net/so/search?q=%E5%86%85%E7%BD%91&spm=1001.2101.3001.7020)服务器了，同时也可以用 sftp 传输文件。

强烈建议你在使用frp直接测试内网穿透前，先在局域网内测试好相关功能的正常使用。

