---
title: "Design Thoughts for Another Tunneling Tool - Netdog"
layout: single
tags:
    - tunneling
    - design
---

As mentioned in the Hello World [post](https://dev-intr.github.io/hello-world/),
there are a multitude of tools that provide basic tunneling of network traffic.
It is _typically_ used by security practioners such as Red Teams on an
engagement, but can also have purposes outside of the security industry. In this
post, I hope to give a quick overview of some of the "classic" tools used for
security and then start jotting down some ideas of things I would like to
incorporate into my tool I plan to design and develop in the coming months.

## Brief Overview of Tunneling Tools
### Port Forwarding with SSH
The simplest tool out there to tunnel network traffic has to be classic SSH.
With SSH, you can create a basic forward tunnel that will allow you to access
any host that the SSH server is able to reach on a specified port. For example, say I am a client
machine located on the 192.168.0.0/24 subnet and I have access to an SSH server
on the 10.0.0.0/24 subnet. The SSH server is "dual-NIC"'ed, so it has an address on
the 10.0.0.0/24 subnet and the 172.16.0.0/24 subnet. My host (on the
192.168.0.0/24) is not able to directly reach the 172.16.0.0/24 subnet (there is no
route), but there is a web server on that subnet that I would like to reach.
Since the SSH server is able to access the 172.16.0.0/24 subnet, I can
use SSH tunneling to reach that web server. 
Here's a (bad) ASCII diagram of the situation:

```txt
 ----------                                  ---------                                 ---------
|          | 192.168.0.2/24     10.0.0.5/24 |         | 172.16.0.5/24  172.16.0.9/24  |         |
| My Host  | --------------- <> ----------- | SSH Svr | ----------- <> -------------- | Web Svr |
|          | eth0                     eth0  |         | eth1                    eth0  |         |
 ----------                                  ---------                                 ---------
```

If I want to access the web server at 172.16.0.9 from My Host at 192.168.0.2, 
I can SSH from My Host to the SSH Server and setup a local port forwarding rule.
What this rule will do is open a port on my machine (can be defaulted to
127.0.0.1) such that any traffic sent to
that port will be sent through the SSH connection
(tunneled). Then, the SSH server will open a connection to the specified
destination and copy the traffic between the two. What this does is effectively
creates a magic portal (like that game) with the entrance of the portal on
a specified port on my host and the end of the portal on the specified destination host and port. The key
here though is that the SSH Server makes this all possible. The SSH daemon on
the server will ensure that any traffic sent on the established channel within
the SSH session will actually be sent to the specified destination.

For this example, we would run something like:

```sh
ssh -L 8080:172.16.0.9:80 user@10.0.0.5
```

Then, if I run:

```sh
curl http://localhost:8080
```

I will actually be curl'ing the web server at 172.16.0.9:80 due to the SSH local
port forward. (Magic through the portal).

It is worth noting that you can also define port forwarding in the "reverse"
direction. All the reverse direction does is...well... reverse who opens ports.
When you specify a reverse port forward, you are instructing the SSH Server to
open up a port and to send any traffic sent to that port to the specified
destination.

For example, if I use this SSH command:

```sh 
ssh -R 8080:localhost:8081 user@10.0.0.5
```

Then, any traffic sent to the SSH server to port 8080 will be
forwarded back to me (localhost) on port 8081. The interface the server will
listen on depends on the configuration of the server (if GatewayPorts is uncommented in `/etc/ssh/sshd_config`, the server will listen on all interfaces; otherwise only 127.0.0.1). 
That means if I run a simple
python3 HTTP server (`python3 -m http.server 8081`) on my host, if anyone curl's the SSH
server on port 8080 (`curl http://10.0.0.5:8080`), they will actually be
curl'ing my python HTTP server. (More Magic). This is incredibly useful for Red
Teams to be able to stage things such as payloads and other tools through an SSH
server they have established access to previously. (For example, when I
completed RastaLabs on HackTheBox, I used SSH reverse port forwarding to stage
payloads on one of the Linux machines I had access to because the target
machines could not directly reach out to my attacker machine). 

Meterpreter also has a built-in port forwarding mechanism that does essentially
the same thing, just tunneling the traffic over the established meterpreter
session.

What this generic port forwarding gives us are a few advantages:

1. Ability to access hosts we would not otherwise be able to access through an established
   connection
2. (Potentially) encrypted traffic between your host and the host serving as the
   'tunneler'

However, there are a number of disadvantages:

1. Inability to tunnel UDP (DNS) or ICMP (Host Discovery)
2. Limited to a 1-to-1 port mapping (you have to open up a new tunnel for every
   host, port pair you want to access in the remote network). This limits your ability to
   conduct massive NMAP scans or other scans to find new services
3. Some common Red Team tools will false positives because they are not
   aware of the tunneling that is going on.

### SOCKS Proxy through SSH
To solve disadvantage 2 for SSH port forwarding, SSH also provides the ability
to create a SOCKS proxy. This is done using the `-D` option and specifying the
port to open the proxy up. Essentially, `ssh` will open up a SOCKS proxy server
(version 4 and 5 is supported) on your local machine. Then, you can use a proxy
client (such as a firefox web-browser with proxy settings configured or
proxychains) to proxy your traffic through the proxy server and have "dynamic"
port forwarding. 

This sort of proxy effectively operates on the transport layer where it
makes a request to the proxy server to connect to a specified host and port.
That request is part of the initial handshake in the SOCKS protocol. The proxy
server will hold open that original connection and just transfer data to the
specified host and port. 

In the scenario above, we could just run this command to start the proxy:

```sh 
ssh -D 9050 user@10.0.0.5
```

Then, we could `curl` the webserver at 172.16.0.9 with `proxychains`:

```sh 
proxychains curl http://172.16.0.9
```

Notice how we don't specify localhost because we configured `proxychains` (in
`/etc/proxychains/proxychains.conf`) to use the SOCKS server on localhost:9050.
Of note, this is the default configuration with proxychains, so it **should**
work out of the box; check the file to see if it defines the proxy server as
`localhost:9050` if you have issues. 

Since we used a proxy server and proxychains, we could specify the IP directly.
We also didn't need to worry about the 1-to-1 mapping of host:ports. We could
have easily ran:

```sh 
proxychains curl http://172.16.0.10
```

And been successful without having to add another "tunnel" to our `ssh` command. 

Similar to port forwarding, meterpreter also has a capability to 'tunnel' 
traffic through a SOCKS proxy. 

This setup provides us with some additional advantages:

1. We no longer have to have a separate tunnel for every host:port we want to 
   tunnel traffic to.
2. We can refer to hosts by their IP's directly and no longer have to keep track
   of which port on our localhost maps to a particular host:port.
3. We can now proxy NMAP scans (without ICMP) for entire subnets 

But, there are still some disadvantages:

1. The performance is usually pretty bad through `proxychains`.
2. `proxychains` uses LD_PRELOAD hooking to hook any TCP connections. This isn't
   necessarily bad, but a bit of a "hacky" method to get it working. This
becomes problematic if you are using a tool that doesn't dynamically link `libc` (i.e.
golang based tools won't work because golang implements their own syscall
wrappers).
3. `proxychains` can't tunnel UDP or ICMP so we're still lacking the ability to
   do host discovery and tunneling DNS traffic.
4. Introduces easy mistakes if you forget to run `proxychains` on your commands
   and end up routing traffic a place you don't want to route traffic.

Of note, this is not matching the exact definition of tunneling (encapsulate one
protocol in another) but rather setting up a proxy that tunnels the data for us.  

### Modern Tools to Solve Some Shortfalls
Some modern tools exist to try to address some of the shortfalls discussed with
SSH. Some of the top tools are: 

1. [Chisel](https://github.com/jpillora/chisel): Chisel agents will use an HTTP connection to call back to a server
   that allows tunneling from the server through the clients. Similar to port
   forwarding in SSH, you must send data to a specific port on the server.
2. [Ligolo-Ng](https://github.com/nicocha30/ligolo-ng): Agents callback to a server that sets up a tun interface and manages
   your routing table to forward traffic through the tun interface.
3. [Sshuttle](https://github.com/sshuttle/sshuttle): This is a hybrid VPN and SSH tunnel (at least that's how the docs
   describe it). You login to another machine over SSH and it will uses `iptables` 
   rules to redirct specified traffic to a port listening on your machine that sends
   it over the SSH connection and then reaches out to your intended destination.

This is not an exhaustive list, but I would say all of these solutions have nice
features that I like and others that I don't like. For example, I always found
chisel's syntax a bit clunky. I love sshuttle's use of `iptables` to
transparently tunnel traffic but you're limited to having an SSH server to
tunnel the traffic. This is phenomenal to quickly use a cloud VM to act as a
lightweight VPN, but doesn't help a Red Team on an engagement in a Windows
network with no SSH traffic or servers.

## Another Tunneling Tool 
With that being said, I have started to throw some ideas together on what this
Rust tunneling tool will look like. I want to use the benefits of `iptables`
like sshuttle with the 'callback' method of Chisel along with the
straightforward TUI presented by Ligolo-Ng. That being said, here are some of
the preliminary requirements:

1. Easy-to-use TUI to manage agents that can tunnel traffic 
2. Use of `iptables` to control the tunneling on the 'server'
3. Design of a protocol between the 'server' and the 'agents' to avoid the
   TCP-over-TCP problem and just copy data
4. Leverage the speed, security, ease of development, and simple cross-compiling
   of Rust and Tokio

One specific thing to keep things limited will be the requirement that the
'server' side (the Red Team operator or person trying to tunnel traffic) needs
to be on a Linux based machine that has `iptables` installed and loaded. 

## First Steps
To keep things manageable, I plan to set some certain milestones to hit before
moving on. The first milestone will be to get the `iptables` rules setup and
have a server receive traffic destined to be tunneled. After that, I will add a
client that will connect to a static listener and tunnel the traffic to that
client. Once this is up, I will write up another post detailing some of the code
and showing what I learned with the first milestone.

Lastly, I had to come up with a name and I chose `netdog`. It's not the same
thing as `netcat`, but provides you the ability to connect to arbitrary hosts.
