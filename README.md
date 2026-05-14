# How to create encrypted ipip-tunnel over http with fou (foo over udp) kernel module and chisel utility in Debian

An example of setting up an HTTP-encapsulated IPIP tunnel on Linux between a client without a fixed IP and a server using chisel (fast TCP/UDP tunnel, transported over HTTP, secured via SSH)

The settings were done and tested on Debian 12 with the classic network configuration defined in /etc/network/interfaces. After applying these settings, the server and client will be accessible to each other by IP addresses 10.99.99.1 and 10.99.99.2

## How it works: 

Using the standard Linux ip utility, an IPIP tunnel is created on the server and client, with peer-to-peer traffic encapsulated in UDP using the fou kernel module. The chisel utility then creates a persistent http connection between the client and server, enabling two UDP port forwardings from the client to the server and back, allowing encapsulated UDP traffic of the IPIP tunnel to pass through  without the need for a fixed IP address on client.

## Steps to do:

First, you need to download and install the deb package of the [chisel](https://github.com/jpillora/chisel) utility from its [releases page](https://github.com/jpillora/chisel/releases) both on server and client

Next, you need to create the systemd unit files for chisel and enable it and create the tunnel interface configuration both on the server and client. Examples are below. 

### Server Configuration

#### Server unit file /etc/systemd/system/chisel.service
```
[Unit]
Description=Chisel Tunnel Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/chisel server --port 58180 --auth "user1:I_AM_PASSWORD_REPLACE_ME" --reverse
Restart=on-failure
User=nobody
Group=nogroup

[Install]
WantedBy=multi-user.target
```

#### Tunnel interface configuration directives ipip/fou (tunXXX) from the file /etc/network/interfaces
```
auto tun100
iface tun100 inet manual
       pre-up  modprobe fou
       pre-up  ip fou add local 127.0.0.1 port 5555 ipproto 4
       pre-up  ip link add name $IFACE type ipip remote 127.0.0.1 local 127.0.0.1 ttl 225 encap fou encap-sport 5555 encap-dport 5556
       up      ip link set $IFACE up mtu 1380
       post-up ip addr add 10.99.99.1 peer 10.99.99.2 dev $IFACE
       down    ip link delete $IFACE
       down    ip fou del local 127.0.0.1 port 5555
```

### Client Configuration 

#### Client unit file /etc/systemd/system/chisel-client.service 
```
[Unit]
Description=Chisel Client Service
After=network.target

[Service]
Type=simple
# User to run the service (non-root recommended)
User=nobody
Group=nogroup
# The command to execute (adjust paths and arguments)
ExecStart=/usr/bin/chisel client --auth "user1:I_AM_PASSWORD_REPLACE_ME" SERVER_IP:58180 R:127.0.0.1:5556:127.0.0.1:5556/udp 127.0.0.1:5555:127.0.0.1:5555/udp
Restart=always
RestartSec=5
 
[Install]
WantedBy=multi-user.target
```

#### Tunnel interface configuration directives ipip/fou (tunXXX) from the file /etc/network/interfaces 

```
auto tun100
iface tun100 inet manual
       pre-up  modprobe fou
       pre-up  ip fou add port 5556 ipproto 4 local 127.0.0.1
       pre-up  ip link add name $IFACE type ipip remote 127.0.0.1 local 127.0.0.1 ttl 225 encap fou encap-sport 5556 encap-dport 5555
       up      ip link set $IFACE up mtu 1380
       post-up ip addr add 10.99.99.2 peer 10.99.99.1 dev $IFACE
       post-up ip route add default dev $IFACE
       down    ip link delete $IFACE
       down    ip fou del local 127.0.0.1 port 5556
```        


