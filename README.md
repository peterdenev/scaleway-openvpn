# OpenVPN image on Scaleway

[![Build Status](https://travis-ci.org/scaleway-community/scaleway-openvpn.svg?branch=master)](https://travis-ci.org/scaleway-community/scaleway-openvpn)
[![Scaleway ImageHub](https://img.shields.io/badge/ImageHub-view-ff69b4.svg)](https://hub.scaleway.com/openvpn.html)
[![Run on Scaleway](https://img.shields.io/badge/Scaleway-run-69b4ff.svg)](https://cloud.scaleway.com/#/servers/new?image=b6f4edc8-21e6-4aa2-8f52-1030cf6d4dd8)

Launch your OpenVPN app on Scaleway servers in minutes.

<img src="https://store.silvenga.com/external/openvpntech_logo1.png" width="400px" />

# Quickstart

## Installation

The install process is fully automatic.
Once your server is booted up, run

```sh
scw-ovpn status
```

To check if your server is ready.

## Creating a new user

Run

```sh
scw-ovpn create your_user
```

To create a new user certificate.
You can now download it using `show` or `serve`.

## Downloading your user configuration

There are multiple way to download your configuration file, the simplest being to run

```sh
scw-ovpn serve your_user
```

This method starts an http server serving your client config: **This method does not use encryption to transfer your configuration.**

You can also download your configuration using the command line using either:

```sh
scw exec your_server scw-ovpn show your_user > your_user.ovpn
```

or

```sh
ssh root@your_server_ip scw-ovpn show your_user > your_user.ovpn
```

## Removing an user

In order to prevent a client from connecting again, its certificate has to be revoked.

It can be done using

```sh
$ scw-ovpn revoke your_user
```

Do not try to remove the client certificate from the easy-rsa keys directory, as it does not prevent the client from connecting again.

# Internals
## Services

By default, the server starts two openvpn instances running on tcp port 443 and udp port 1194.

You can list currently running instances using

```sh
$ # <protocol> <port> <subnet suffix> <service status>
$ scw-ovpn list-instances
udp    1194   0   active
tcp    443    1   active
```

Each instance is backed by a systemd service, for instance `openvpn@udp_1194_0` and `openvpn@tcp_443_1`.

You can play with instances using

```sh
$ scw-ovpn add-instance udp 4242 3
$ scw-ovpn del-instance udp 4242 3
```

`add-instance` checks if another service uses the same tcp and port or subnet id.

The `scw-ovpn-gen-server` hook generates the server configuration on instance start and reload.


## Networking

Instances have unbridged independant interfaces, running on separate subnets.

The subnet for each instance is made using a prefix and the instance subnet ID, for both ipv4 and ipv6.

You can configure this prefix in `/etc/openvpn/scw-vars.sh`.

The prefixes currently are `100.64.0.0/16` for ipv4, and `fd42:5ca1:e3a7::0/48` for ipv6 (see [rfc6598](https://tools.ietf.org/html/rfc6598) for ipv4 and
[rfc4193](https://tools.ietf.org/html/rfc4193) for ipv6).

The next 8 bit block for ipv4 and 16 bit block for ipv6 is the correct representation of the subnet ID, which makes up a `/24` subnet for ipv4 as well as a `/64` subnet for ipv6.

### NAT
Nat is configured using a service running at boot, which runs `scw-setup-nat` before the openvpn server starts.

This is a `SNAT` based setup, so the IP addresses of the machine are looked up at boot. The script assumes the name of the main interface is `eth0`.

IPv6 is also NATed.

### DNS
The image also runs an unbound powered DNS relay to the resolvers of the host (by default scaleway DNS servers).

This relays only accepts connections from the vpn server.

The unbound configuration is generated on each boot by the `setup-unbound` service, which runs `scw-setup-unbound`.

If you change the subnet prefixes in `/etc/openvpn/scw-vars.sh`, you should restart `setup-unbound` first, then `unbound`, or restart your server.

### IPv6
As previously stated, IPv6 is currently NATed.

In order to avoid IPv6 leaks out of the VPN, we always offers the client an IP, even if the server does not have any valid route to the internet. It also routes `2000::/3` (all currently assignable IPs) to the VPN.

This setup should make the client fallback to IPv4 if the scaleway server does not feature IPv6 connectivity.

## Crypto
The current setup uses:

 * the `AES-256-CBC` cipher
 * enforces a minimum TLS version of 1.2
 * the `SHA256` authentication message digest
 * the default TLS ciphers, for better compatibility
 * a static PSK for TLS auth

Certificates are generated using easy-rsa, and properly checked for revocation.

Some of these parameters can be changed in the `/etc/openvpn/scw-vars.sh` config file.

## How to hack

**This image is meant to be used on a Scaleway server.**

We use the Docker's building system and convert it at the end to a disk image that will boot on real servers without Docker. Note that the image is still runnable as a Docker container for debug or for inheritance.

[More info](https://github.com/scaleway/image-builder)

---

A project by [![Scaleway](https://avatars1.githubusercontent.com/u/5185491?v=3&s=42)](https://www.scaleway.com/)
