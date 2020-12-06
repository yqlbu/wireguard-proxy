## Artifact

This tutorial was created and tested using Ubuntu 18.04 on Bandwagon VPS Hosting. It will likely work fine with other distributions and any other Cloud Services Providers, but some modifications may be needed.

## Table of Contents

- **[Table of Contents](#table-of-contents)**
- **[Quick Deploy](#quick-deploy)**
- **[Documentation](#documentation)**
  - [Workflow](#workflow)
  - [Get Ready](#get-ready)
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
- **[Reference](#reference)**
- **[License](#license)**

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
AllowedIPs = 10.77.0.2/24 <You may config as many LAN as you want>
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

Installation Client Software

As for now, Wireguard is supported over a varity of platforms. The software is now supported Windows, macOS, Android, iOS, and many Linux Distributed Systems. You may find the client software that matches your operating system on the [Installation Page](https://www.wireguard.com/install/).

Turn on Wireguard and enable auto start at boot

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

---

## Documentation

### Workflow

### Get Ready

### Server Setup

#### Installation

#### Generate Keys

#### Server Side Configuration

#### Test Connection

### Client Setup

#### Client Side Configuration

### Network Address Translation

#### Wireguard NAT Workflow

#### Multiple LAN NAT Support

## Reference

## License

[MIT License](https://github.com/yqlbu/wireguard-proxy/blob/main/LICENSE)
