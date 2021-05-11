# 前言

一款帮你轻松连接回家的代理神器

本教程搭建和测试基于Ubuntu 18.04系统，在云上搭建，VPS 提供商为搬瓦工。WireGuard 也可以在其他的云服务商所提供的其他 Linux 发行版系统上运行，但是需要做一些细节上的调整。

 本教程将引导您知道:当您外出时，如何通过 wireguard-proxy 将手机，平板电脑或笔记本电脑等设备连接到家庭网络。

 具体的网络逻辑框架如下：

```bash
10.77.0.1 (Wireguard Server on VPS) <--------+ 10.77.0.X (Wireguard Client)
              +
              |
              v
10.77.0.2 (NAS or any other devices within the home network)
              +
              |
              v
10.10.10.0/24 (Home Network)
```

关于 WireGuard 具体的网络逻辑框架介绍：[点击此处](#工作流程)

## 目录

- [快速部署](#快速部署)
  - [将 WireGuard 部署到您的VPS上](#将-WireGuard-部署到您的VPS上)
  - [设置自动启动](#设置自动启动)
  - [激活ipv4转发规则](#激活ipv4转发规则)
  - [客户端软件安装](#客户端软件安装)
  - [配置移动客户端](#配置移动客户端)
  - [测试连接](#测试连接)
- [文档](#文档)
  - [前期准备](#前期准备)
  - [网络逻辑架构](#工作流程)
  - [服务器设置](#服务器设置)
    - [安装](#安装)
    - [创建密钥](#创建密钥)
    - [服务端设置](#服务器端配置)
    - [(可选) 防火墙高级配置]((可选)-防火墙高级配置)
  - [客户端设置](#客户端设置)
    - [客户端配置](#客户端配置)
    - [确认服务端配置](#修改服务器端配置)
    - [开启转发规则](#开启转发规则)
    - [客户端软件安装](#客户端软件安装)
    - [连接通路测试](#测试连接)
  - [网络地址转换](#网络地址转换)
    - [WireGuard NAT 网络逻辑框架](#Wireguard-NAT工作流程)
    - [多链路对等连接（多设备连接）](#多链路对等连接-多设备连接)
    - [多局域网NAT支持](#多局域网NAT支持)
- [参考](#参考)
- [授权](#授权)

## 快速部署

### 将 WireGuard 部署到您的VPS上

```bash
$ sudo apt-get update && apt-get upgrade -y
$ sudo apt-get install wireguard openresolv -y
$ wg genkey | tee privatekey | wg pubkey > publickey
$ cat privatekey
$ cat publickey
```

创建该路径文件 `/etc/wireguard/wg0.conf`

```bash
$ vim /etc/wireguard/wg0.conf
```

该文件模板如下：

```config
[Interface]
PrivateKey = <Your Private Key Goes Here>
Address = 10.77.0.1/24 <Virtual LAN Address>
ListenPort = 51820 <Custom Port>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
DNS = 8.8.8.8
MTU = 1420

[Peer]
# client
PublicKey = <Client Public Key Goes Here>
Allowed IPs = 10.77.0.2/32, 10.10.10.0/24 <10.10.10.0/24 is the LAN in your home network>
```

注释：

客户端私钥不同于服务器私钥，您可以运行以下命令来生成一对新的公钥和私钥：

```bash
$ wg genkey | tee privatekey_client | wg pubkey > publickey_client
$ cat privatekey_client
$ cat publickey_client
```

创建该路径文件` /etc/wireguard/client.conf`

```bash
$ vim /etc/wireguard/client.conf
```

该文件模板如下：

```config
[Interface]
PrivateKey = <Client Private Key Goes Here>
PostUp=iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE;
PostDown=iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE;
Address = 10.77.0.2/24
DNS = 8.8.8.8
MTU = 1420

[Peer]
PublicKey = <Server Public Key Goes Here>
Endpoint = <Your VPS IP or domain goes here>:<Custom Port on wg0.conf>
AllowedIPs = 10.77.0.2/24
PersistentKeepalive = 25
```

请注意以上命令中所包含的其中两行命令：

```config
PostUp=iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE;
PostDown=iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE;
```

注释：

上面的规则是允许 WireGuard 从您的家庭网络到wg0虚拟网络接口接收任何流量，以便Wireguard在该虚拟网络（wg0）中创建的任何对等链路方都可以相互访问。在这里，您可能需要修改（ens18）您的物理网络接口，例如（eth0）（注:不同的操作系统网络接口有不同的命名规则，根据您操作系统实际的命名情况而进行调整）

**(可选) 防火墙高级配置:**

用`firewalld`配置防火墙。[Firewalld](https://Firewalld . org/)是一个完整的防火墙解决方案，它管理系统的iptables规则，并为在这些规则上操作提供D-Bus接口。从CentOS 7开始，FirewallD取代iptables成为默认的防火墙管理工具。

安装 `Firewalld`

```
$ sudo apt install firewalld -y
```

模板

```
[Interface]
PrivateKey = QO4hRctHejfP9MD+j6IXzlskPwW3+NwoMQhxyHVGYFM=
Address = 172.8.0.1
ListenPort = 51820

PostUp = firewall-cmd --zone=public --add-port 51820/udp && firewall-cmd --zone=internal --add-interface=wg0 && firewall-cmd --zone=internal --add-service=dns && firewall-cmd --zone=internal --add-service=http && firewall-cmd --zone=public --add-masquerade && firewall-cmd --zone=internal --add-masquerade

PostDown = firewall-cmd --zone=public --remove-port 51820/udp && firewall-cmd --zone=internal --remove-interface=wg0 && firewall-cmd --zone=internal --remove-service=dns && firewall-cmd --zone=internal --remove-service=http && firewall-cmd --zone=public --remove-masquerade && firewall-cmd --zone=internal --remove-masquerade

DNS = 8.8.8.8
MTU = 1420

[Peer]
# client
PublicKey = <Client Public Key Goes Here>
Allowed IPs = 10.77.0.2/32, 10.10.10.0/24 <10.10.10.0/24 is the LAN in your home network>
```

您可以在 [这里](https://linuxize.com/post/how-to-setup-a-firewall-with-firewalld-on-centos-7/) 找到关于 `PortUp` 和 `PortDown` 的防火墙-cmd配置的说明。

---

### 设置自动启动

打开Wireguard服务并在你承载wireguard服务的服务器引导时自动启动

```bash
$ wg-quick up wg0
$ systemctl enable wg-quick@wg0
```

### 激活ipv4转发规则

确保需要共享 nat 的 wireguard 节点 ip 转发已开启，编辑`/etc/sysctl.conf`，将下面的这行的注释去掉

```bash
# net.ipv4.ip_forward=1
```

您也可以每次开机手动激活ipv4转发规则

```bash
$ sysctl -w net.ipv4.ip_forward=1
```

重启并来激活上述规则

```
$ reboot
$ sysctl -p
```

---

### 客户端软件安装

至今为止，多种不同的平台已经支持Wireguard 。例如 Windows，macOS，Android，iOS 和许多 Linux 发行版系统。您可以在 [安装页面](https://www.wireguard.com/install/)上找到与您的操作系统匹配的客户端软件。

### 配置移动客户端

移动客户端私钥不同于服务器私钥，您可以运行以下命令来生成一对新的公钥和私钥：

```bash
$ wg genkey | tee privatekey_client | wg pubkey > publickey_client
$ cat privatekey_client
$ cat publickey_client
```

创建该路径文件` /etc/wireguard/client_mobile.conf`

```bash
$ vim /etc/wireguard/client_mobile.conf
```

该文件模板如下：

```config
[Interface]
PrivateKey = <Client Private Key Goes Here>
Address = 10.77.0.3/24
DNS = 8.8.8.8
MTU = 1420

[Peer]
PublicKey = <Server Public Key Goes Here>
Endpoint = <Your VPS IP or domain goes here>:<Custom Port on wg0.conf>
AllowedIPs = 0.0.0.0/0, ::0/0
PersistentKeepalive = 25
```

在服务器端更新新的Peer配置：

```
[Interface]
PrivateKey = <Your Private Key Goes Here>
Address = 10.77.0.1/24 <Virtual LAN Address>
ListenPort = 51820 <Custom Port>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
DNS = 8.8.8.8
MTU = 1420

[Peer]
# client
PublicKey = <Client Private Key Goes Here>
Allowed IPs = 10.77.0.2/32, 10.10.10.0/24 <10.10.10.0/24 is the LAN in your home network>

[Peer]
# client_mobile
PublicKey = <Client Private Key Goes Here>
AllowedIPs = 10.77.0.3/32
```

最后，将配置好的`client_mobile.conf`文件导出至对应的移动客户端即可。若有多个移动客户端需求，则可以生成多个密钥对，并在服务端更新即可。

### 测试连接

将 `client.conf` 从VPS服务器中复制到 WireGuard 客户端软件中，如果一切均已正确设置，则你的客户端应该已连接，并可以看到客户端开始有网络流量。

在 WireGuard Server上输入以下命令，以确保链路已经被激活且稳定运行：

```bash
$ wg

# output
interface: wg0
  public key: <Server Public Key>
  private key: (hidden)
  listening port: 13789

peer: <Peer Public Key>
  endpoint: <Server IP>:<Custom Port>
  allowed ips: 10.77.0.2/32, 10.10.10.0/24
  latest handshake: 54 seconds ago
  transfer: 9.52 MiB received, 4.01 MiB sent
```

## 文档

### 前期准备

在开始使用前，你需要准备的东西如下：

- 来自云服务提供商的 VPS
- 您想从家庭网络外部访问的设备（例如NAS）

这个技术能让您做什么

- 当您不在家庭网络中时，通过Wireguard Proxy将设备（例如手机，平板电脑或笔记本电脑）连接到家庭网络

### 工作流程

![img](https://github.com/yqlbu/wireguard-proxy/raw/main/images/Screen%20Shot%202020-12-06%20at%206.48.17%20PM.png?raw=true)

注释：

- `1.1.1.1` 是 VPS 的公共 IP（由云服务供应商提供）
- `10.77.0.0/24` 是 WireGuard Server 创建的虚拟本地网络
- `10.10.10.0/24` 是本地家庭网络

### 服务器设置

从您的云服务供应商商提供的VM实例进行安装。本教程是使用 [Bandwagon](https://bandwagonhost.com/aff.php?aff=63096) VPS Hosting 上的Ubuntu 18.04创建和测试的。WireGuard 也可以在其他的云服务商所提供的其他 Linux 发行版系统上运行。

系统版本预览：

```bash
root@quick-beep-1:~# screenfetch
         _,met$$$$$gg.           root@quick-beep-1.localdomain
      ,g$$$$$$$$$$$$$$$P.        OS: Debian 10 buster
    ,g$$P""       """Y$$.".      Kernel: x86_64 Linux 5.8.0-0.bpo.2-cloud-amd64
   ,$$P'              `$$$.      Uptime: 1d 2h 43m
  ',$$P       ,ggs.     `$$b:    Packages: 394
  `d$$'     ,$P"'   .    $$$     Shell: bash 5.0.3
   $$P      d$'     ,    $$P     CPU: QEMU Virtual version (cpu64-rhel6) @ 2x 2.7GHz
   $$:      $$.   -    ,d$$'     GPU: EFI
   $$\;      Y$b._   _,d$P'      RAM: 211MiB / 999MiB
   Y$$.    `.`"Y$$$$P"'
   `$$b      "-.__
    `Y$$
     `Y$$.
       `$$b.
         `Y$$b.
            `"Y$b._
```

#### 安装

用终端SSH到你的VPS上，确保系统各个组件已经升级到最新。

```bash
$ apt-get update && apt-get upgrade -y
```

接下来，我们需要开启ip转发，IP转发是让操作系统在一个网络接口上有接受传入的网络数据包的能力，并且可以识别这个网络数据包目的地址并非系统本身，且将其传递给另一网络。编辑文件 `/etc/sysctl.conf` ， 并将 `net.ipv4.ip_forward = 1` 这行取消注释。

接下来你需要输入命令 `reboot` 或者 `sysctl -p` 来让你的配置生效。注：前者会重新启动系统，生产环境中需要谨慎使用。

安装 WireGuard

```bash
$ sudo apt-get install wireguard openresolv -y
```

#### 设置密钥对

转到Wireguard的配置,然后运行命令来生成服务器的公钥与私钥。

对服务器生成密钥对，命令如下：

```bash
$ cd /etc/wireguard
$ wg genkey | tee privatekey | wg pubkey > publickey
$ cat privatekey
$ cat publickey
```

为客户端生成另一个密钥对，命令如下：

```bash
$ wg genkey | tee privatekey_client | wg pubkey > publickey_client
$ cat privatekey_client
$ cat publickey_client
```

#### 服务器端配置

创建文件  文件路径及名称为:/etc/wireguard/wg0.conf

```bash
$ vim /etc/wireguard/wg0.conf
```

这将获得基本的服务器端设置，稍后我们将返回此文件编辑内容来使对等链接可以成功。

模板如下：

```config
[Interface]
PrivateKey = <Your Private Key Goes Here>
Address = 10.77.0.1/24 <Virtual LAN Address>
ListenPort = 51820 <Custom Port>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
DNS = 8.8.8.8
MTU = 1420
```

请注意上面配置中的如下的两行：

```config
PostUp=iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE;
PostDown=iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE;
```

上面的规则是允许Wireguard从您的家庭网络到wg0虚拟网络接口接收任何流量，以便Wireguard在该虚拟网络（wg0）中创建的任何对等链路方都可以相互访问。在这里，您可能需要修改（ens18）您的物理网络接口，例如（eth0）（注:不同的操作系统网络接口有不同的命名规则，根据您操作系统实际的命名情况而进行调整）

#### (可选) 防火墙高级配置

用`firewalld`配置防火墙。[Firewalld](https://Firewalld . org/)是一个完整的防火墙解决方案，它管理系统的iptables规则，并为在这些规则上操作提供D-Bus接口。从CentOS 7开始，FirewallD取代iptables成为默认的防火墙管理工具。

安装 `Firewalld`

```
$ sudo apt install firewalld -y
```

模板

```
[Interface]
PrivateKey = QO4hRctHejfP9MD+j6IXzlskPwW3+NwoMQhxyHVGYFM=
Address = 172.8.0.1
ListenPort = 51820

PostUp = firewall-cmd --zone=public --add-port 51820/udp && firewall-cmd --zone=internal --add-interface=wg0 && firewall-cmd --zone=internal --add-service=dns && firewall-cmd --zone=internal --add-service=http && firewall-cmd --zone=public --add-masquerade && firewall-cmd --zone=internal --add-masquerade

PostDown = firewall-cmd --zone=public --remove-port 51820/udp && firewall-cmd --zone=internal --remove-interface=wg0 && firewall-cmd --zone=internal --remove-service=dns && firewall-cmd --zone=internal --remove-service=http && firewall-cmd --zone=public --remove-masquerade && firewall-cmd --zone=internal --remove-masquerade

DNS = 8.8.8.8
MTU = 1420

[Peer]
# client
PublicKey = <Client Public Key Goes Here>
Allowed IPs = 10.77.0.2/32, 10.10.10.0/24 <10.10.10.0/24 is the LAN in your home network>
```

您可以在 [这里](https://linuxize.com/post/how-to-setup-a-firewall-with-firewalld-on-centos-7/) 找到关于 `PortUp` 和 `PortDown` 的防火墙-cmd配置的说明。

### 客户端设置

在VPS服务器上继续执行以下步骤：

#### 客户端配置

创建文件，文件路径及文件名:/etc/wireguard/client.conf 

命令如下：

```bash
$ vim /etc/wireguard/client.conf
```

文件模板:

```config
[Interface]
PrivateKey = <Client Private Key Goes Here>
PostUp=iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE;
PostDown=iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE;
Address = 10.77.0.2/24
DNS = 8.8.8.8
MTU = 1420

[Peer]
PublicKey = <Server Public Key Goes Here>
Endpoint = <Your VPS IP or domain goes here>:<Custom Port on wg0.conf>
AllowedIPs = 10.77.0.2/24
PersistentKeepalive = 25
```

请注意上面配置中的如下的两行：

```config
PostUp=iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE;
PostDown=iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE;
```

上面的规则是允许Wireguard从您的家庭网络到wg0虚拟网络接口接收任何流量，以便Wireguard在该虚拟网络（wg0）中创建的任何对等链路方都可以相互访问。在这里，您可能需要修改（ens18）您的物理网络接口，例如（eth0）（注:不同的操作系统网络接口有不同的命名规则，根据您操作系统实际的命名情况而进行调整）

#### 修改服务器端配置

模板

```config
[Interface]
PrivateKey = <Your Private Key Goes Here>
Address = 10.77.0.1/24 <Virtual LAN Address>
ListenPort = 51820 <Custom Port>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
DNS = 8.8.8.8
MTU = 1420

[Peer]
# client
PublicKey = <Client Public Key Goes Here>
Allowed IPs = 10.77.0.2/32, 10.10.10.0/24 <10.10.10.0/24 is the LAN in your home network>
```

#### 开启转发规则

确保需要共享 nat 的 wireguard 节点 ip 转发已开启，编辑`/etc/sysctl.conf`，将下面的这行的注释去掉

```bash
# net.ipv4.ip_forward=1
```

您也可以每次开机手动激活ipv4转发规则

```bash
$ sysctl -w net.ipv4.ip_forward=1
```

重启并来激活上述规则

```
$ reboot
$ sysctl -p
```

#### 客户端软件安装

至今为止，多种不同的平台已经支持Wireguard 。例如 Windows，macOS，Android，iOS 和许多 Linux 发行版系统。您可以在 [安装页面](https://www.wireguard.com/install/)上找到与您的操作系统匹配的客户端软件。

#### 测试连接

如果想测试您的服务器wireguard服务能否正常运行。您可以这么做：

此命令会打开wg0口：

```bash
$ wg-quick up wg0
```

此命令会关闭wireguard服务：

```bash
$ wg-quick down
```

将client.conf从VPS服务器中复制到Wireguard客户端软件中，如果一切均已正确设置，则你的客户端应该已连接，并可以看到客户端开始有网络流量。

在Wireguard Server上输入以下命令，以确保链路已经被激活且稳定运行：

```bash
$ wg

# output
interface: wg0
  public key: <Server Public Key>
  private key: (hidden)
  listening port: 13789

peer: <Peer Public Key>
  endpoint: <Server IP>:<Custom Port>
  allowed ips: 10.77.0.2/32, 10.10.10.0/24
  latest handshake: 54 seconds ago
  transfer: 9.52 MiB received, 4.01 MiB sent
```

现在测试从VPS到家庭网络的连接状况：

```bash
$ ping 10.10.10.85
```

![img](https://github.com/yqlbu/wireguard-proxy/raw/main/images/IMAGE%202020-12-06%2018:22:48.jpg?raw=true)

如果所有的配置正确无误，则应该看到VPS服务器已通过Wireguard VPN隧道连接到家庭网络。

### 网络地址转换

#### WireGuard NAT工作流程

通过以上配置，请注意，这`AllowIPs`被视为Wireguard虚拟网络中的路由规则。它基本上指的是以下内容：

- 当数据包的目的地到达`AllowIPs`所属网络时，该包将通过Wireguard VPN隧道发送
- 当数据包的目的地未到达`AllowIPs`所属网络时，该包将被丢弃

```bash
10.77.0.1 (Wireguard Server) <--------+ 10.77.0.X (Wireguard Client)
              +
              |
              v
10.77.0.2 (NAS or any other devices within the home network)
              +
              |
              v
10.10.10.0/24 (Home Network)
```

#### 多链路对等连接 (多设备连接)

您可以根据实际需要添加任意数量的对等链接方，只需确保将所有对等方添加到VPS服务器中：

移动客户端私钥不同于服务器私钥，您可以运行以下命令来生成一对新的公钥和私钥：

```bash
$ wg genkey | tee privatekey_client | wg pubkey > publickey_client
$ cat privatekey_client
$ cat publickey_client
```

创建该路径文件` /etc/wireguard/client_mobile.conf`

```bash
$ vim /etc/wireguard/client_mobile.conf
```

该文件模板如下：

```config
[Interface]
PrivateKey = <Client Private Key Goes Here>
Address = 10.77.0.3/24
DNS = 8.8.8.8
MTU = 1420

[Peer]
PublicKey = <Server Public Key Goes Here>
Endpoint = <Your VPS IP or domain goes here>:<Custom Port on wg0.conf>
AllowedIPs = 0.0.0.0/0, ::0/0
PersistentKeepalive = 25
```

更新VPS服务器端的配置，具体模版如下：

```config
[Interface]
PrivateKey =
Address = 10.77.0.1/24
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort =
DNS = 8.8.8.8
MTU = 1420

[Peer]
# Device One
PublicKey = <Client Private Key Goes Here>
AllowedIPs = 10.77.0.2/32

[Peer]
# Device Two
PublicKey = <Client Private Key Goes Here>
AllowedIPs = 10.77.0.3/32
```

注释：

- 每个对等方都具有一个唯一的密钥对，其中包括一个公共密钥和一个私有密钥
- 要生成新的密钥对，只需使用以下命令 `$ wg genkey | tee privatekey | wg pubkey > publickey`

#### 多局域网NAT支持

如果要在本地网络中配置多个LAN，则可以将 [WireGuard NAT工作流](#工作流程) 作为参考，并修改Peer配置，`/etc/wireguard/wg0.conf`如下所示：

配置模板

```config
[Interface]
PrivateKey =
Address = 10.77.0.1/24
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 13789
DNS = 8.8.8.8
MTU = 1420

[Peer]
# Client One
PublicKey =
AllowedIPs = 10.77.0.2/32

[Peer]
# Client Two
PublicKey =
AllowedIPs = 10.77.0.3/32, 10.10.10.0/24, 10.20.0.0/24 <Where you add more LANs with comma to split them>
```

注释：

​    本人的搬瓦工AAF为：https://bandwagonhost.com/aff.php?aff=63096

​    如果看完本文章觉得对您有所帮助，且需要购买VPS的小伙伴，请您通过上面的链接购买，这样您在购买的同时我也会获得一部分返利，玫瑰送人，手留余香，感激不尽。

## 参考

- [WireGuard 官方网站](https://www.wireguard.com/)
- [YouTube上Lawrencesystems的视频教程](https://forums.lawrencesystems.com/t/getting-started-building-your-own-wireguard-vpn-server/7425)
- [通过 Wireguard 构建 NAT to NAT VPN](https://anyisalin.github.io/2018/11/21/fast-flexible-nat-to-nat-vpn-wireguard/)

## 许可

[MIT License](https://github.com/yqlbu/wireguard-proxy/blob/main/LICENSE)

