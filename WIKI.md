## Artifact

This tutorial was created and tested using Ubuntu 18.04 on Bandwagon VPS Hosting. It will likely work fine with other Linux distributions and any other Cloud Services Providers, but some modifications may be needed.

## Table of Contents

- [Quick Deploy](#quick-deploy)
- [Documentation](#documentation)

  - [Get Ready](#get-ready)
  - [Workflow](#workflow)
  - [Server Setup](#server-setup)
    - [Installation](#installation)
    - [Generate Keys](#generate-keys)
    - [Server Side Configuration](#server-side-configuration)
    - [Test Connection](#test-connection)
  - [Client Setup](#client-setup)
    - [Client Side Configuration](#client-side-configuration)
  - [Network Address Translation](#network-address-translation)
    - [Wireguard NAT Workflow](#wireguard-nat-workflow)
    - [Multiple LAN NAT Support](#multiple-lan-nat-support)

- [Reference](#reference)
- [License](#license)

## Quick Deploy

Deploy Wireguard on your VPS

```bash
$ sudo apt-get update && apt-get upgrade
$ sudo apt-get install wireguard -y
$ wg genkey | tee privatekey | wg pubkey > publickey
$ cat privatekey
$ cat publickey
```

Create the /etc/wireguard/wg0.conf

```bash
$ vim /etc/wireguard/wg0.conf
```

Template

```config
[Interface]
PrivateKey = <Your Private Key Goes Here>
Address = 10.77.0.1/24 <Virtual LAN Address>
ListenPort = 51820 <Custom Port>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
DNS = 8.8.8.8
MTU = 1420

[Peer]
# client
PublicKey = <Client Private Key Goes Here>
AllowedIPs = 10.77.0.2/24 <You may configure as many LANs as you want>
# Allowed IPs = 10.77.0.2/24, 10.10.10.0/24 <10.10.10.0/24 is the LAN in your home network>
```

Notes:

The Client Private Key is different from the one with the Server Private Key, you may run the code below to generate a new pair of public and private key:

```bash
$ wg genkey | tee privatekey_client | wg pubkey > publickey_client
$ cat privatekey_client
$ cat publickey_client
```

---

Create the /etc/wireguard/client.conf

```bash
$ vim /etc/wireguard/client.conf
```

Template

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
AllowedIPs = 10.77.0.0/24
PersistentKeepalive = 25
```

Notes:

```bash
PostUp=iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE;
PostDown=iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE;
```

The rule above is to allow Wireguard to take any traffic from your home network to the wg0 virtual network interface so that any peer within that virtual network (wg0) created by wireguard can visit each other. Here, you may need to modify (ens18) to your physical network interface such as (eth0)

---

Install Client Software

As for now, Wireguard is supported over a varity of platforms. The software is now supported Windows, macOS, Android, iOS, and many Linux Distributed Systems. You may find the client software that matches your operating system on the [Installation Page](https://www.wireguard.com/install/).

Turn on Wireguard and enable auto start at boot on your Wireguard Server

```bash
$ wg-quick up wg0
$ systemctl enable wg-quick@wg0
```

Test Connection

Copy the client.conf to your Wireguard Client Software, if everything has been setup properly, you should be connected, and see the network traffic.

Type the following command on your Wireguard Server to ensure the connection is active and stable:

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

## Documentation

### Get Ready

What to prepare

- a VPS from a Cloud Service provider
- a device that you want to be accessed to such as a NAS from outside of your home network

What to achieve

- Connect your device such as a mobile phone, a tablet, or a latop to your home network when you are outside of your home network via Wireguard Proxy

### Workflow

![](https://github.com/yqlbu/wireguard-proxy/blob/main/images/Screen%20Shot%202020-12-06%20at%205.25.45%20PM.png?raw=true)

Notes:

- 1.1.1.1 is the Public IP of the VPS (Provided by Cloud Service Provider)
- 10.77.0.0/24 is the virtual local network created by Wireguard Server
- 10.10.10.0/24 is the local home network

### Server Setup

Instantiate a VM instance from your Cloud Service Provider. This tutorial was created and tested using Ubuntu 18.04 on [Bandwagon](https://bwh88.net/) VPS Hosting. It will likely work fine with other Linux distributions.

Specs:

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
                `""""
```

#### Installation

Log into server and make sure system is up to date

```bash
$ apt-get update && apt-get upgrade -y
```

Next we need to enable IP Forwarding. IP forwarding is the ability for an operating system to accept incoming network packets on one interface, recognize that it is not meant for the system itself, but that it should be passed on to another network. Edit the file /etc/sysctl.conf and change and uncomment to the line that says **net.ipv4.ip_forward=1**

Now reboot or run sysctl -p to activate the changes.

Install wireguard

```bash
$ sudo apt-get install wireguard -y
```

#### Generate Keys

Go to to the Wireguard config cd /etc/wiregaurd and then run the following command to generate the public and private keys for the server.

Generate a key pair for the Server

```bash
$ cd /etc/wireguard
$ wg genkey | tee privatekey | wg pubkey > publickey
$ cat privatekey
$ cat publickey
```

Generate another key pair for the Client

```bash
$ wg genkey | tee privatekey_client | wg pubkey > publickey_client
$ cat privatekey_client
$ cat publickey_client
```

#### Server Side Configuration

Create the /etc/wireguard/wg0.conf

```bash
$ vim /etc/wireguard/wg0.conf
```

This will get the basic server side setup and we will be coming back to this file to add the peers later.

Template

```config
[Interface]
PrivateKey = <Your Private Key Goes Here>
Address = 10.77.0.1/24 <Virtual LAN Address>
ListenPort = 51820 <Custom Port>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
DNS = 8.8.8.8
MTU = 1420
```

Notes:

```bash
PostUp=iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE;
PostDown=iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE;
```

The rule above is to allow Wireguard to take any traffic from your home network to the wg0 virtual network interface so that any peer within that virtual network (wg0) created by wireguard can visit each other. Here, you may need to modify (ens18) to your physical network interface such as (eth0)

### Client Setup

Continue the following steps on your VPS Server:

#### Client Side Configuration

Create the /etc/wireguard/client.conf

```bash
$ vim /etc/wireguard/client.conf
```

Template

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
AllowedIPs = 10.77.0.0/24
PersistentKeepalive = 25
```

Notes:

```bash
PostUp=iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE;
PostDown=iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE;
```

The rule above is to allow Wireguard to take any traffic from your home network to the wg0 virtual network interface so that any peer within that virtual network (wg0) created by wireguard can visit each other. Here, you may need to modify (ens18) to your physical network interface such as (eth0)

#### Modify Server Side Configuration

Modify the /etc/wireguard/wg0.conf

Template

```config
[Interface]
PrivateKey = <Your Private Key Goes Here>
Address = 10.77.0.1/24 <Virtual LAN Address>
ListenPort = 51820 <Custom Port>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
DNS = 8.8.8.8
MTU = 1420

[Peer]
# client
PublicKey = <Client Private Key Goes Here>
AllowedIPs = 10.77.0.2/24 <You may configure as many LANs as you want>
# Allowed IPs = 10.77.0.2/24, 10.10.10.0/24 <10.10.10.0/24 is the LAN in your home network>
```

#### Client Side Software Installation

As for now, Wireguard is supported over a varity of platforms. The software is now supported Windows, macOS, Android, iOS, and many Linux Distributed Systems. You may find the client software that matches your operating system on the [Installation Page](https://www.wireguard.com/install/).

### Test Connection

To test that the server works run `$ wg-quick up wg0` to bring up the interface. Running `$ wg-quick down` will bring the interface down.

Copy the client.conf from the VPS Server to your Wireguard Client Software, if everything has been setup properly, you should be connected, and see the network traffic.

Type the following command on your Wireguard Server to ensure the connection is active and stable:

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

Now test the connection from the VPS to the home network

```bash
$ ping 10.10.10.85
```

### Network Address Translation

#### Wireguard NAT Workflow

From the above configuration, noted that `AllowIPs` is recognized as the routing rules within the Wireguard Virtual Network. It basically refers to the following:

- When the destination of data package is destinated to the `AllowIPs` networks, the package will be sent via the Wireguard VPN tunnel
- When the destination of data package is not destinated to the `AllowIPs` networks, the package will be dropped instead

```shell
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

#### Multiple LAN NAT Support

## Reference

## License

[MIT License](https://github.com/yqlbu/wireguard-proxy/blob/main/LICENSE)
