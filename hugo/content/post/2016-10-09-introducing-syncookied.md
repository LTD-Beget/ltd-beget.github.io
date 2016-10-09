{
    "title": "SYN flood mitigation with syncookied",
    "date": "2016-10-09",
    "author": "Alexander Polakov"
}

Here at Beget, we operate hundreds of servers hosting thousands of websites for our customers. Like any large hosting provider we have to deal with DDOS attacks on our network. Due to the nature of shared hosting, an attack directed against one site can affect other sites on the same server.

There're many different types of attacks. One of the most dangerous is the SYN flood attack.

### TCP connection establishment

TCP connections is established by so-called "three-way handshake". When TCP server starts, it passively waits for a connection by executing `listen()` and `accept()` system calls. The client on the other side sends a `SYN` packet with an arbitrary sequence number, starting connection parameter negotiation. The server saves client's parameters and sequence number and responds with a `SYN+ACK` packet with its own preferred parameters, sequence number and client's sequence number incremented by one. The client sends `ACK` packet back and connection is now established.

![3 way handshake](/img/3-way-handshake.png)

During SYN flood attack the server is flooded with millions of SYN packets per second. In older releases kernel created a new socket for each received SYN packet. It was a huge waste of resources. More recent kernels employ "mini-sockets" technique to reduce the impact of SYN attacks, this work was done by an awesome Google engineer Eric Dumazet and is available in mainline kernels since 4.5.

Using mini-sockets or not, it's clearly undesirable to store any state before the connection is fully established, and this is how we come to the idea of SYN cookies.


### SYN cookies

This technique was invented by Daniel J.Berstein in 1996. The crux of the idea is to encode incoming tcp connection parameters into the sequence number sent back to the client (in a `SYN+ACK` packet). A conforming client will then increment this number and send it back to us, effectively freeing the server from the burden of saving state. Upon receiving a final `ACK` from the client server validates that the cookie was indeed generated on the server.

The cookie is created by hashing source and destination IP addresses and ports and adding initial client's sequence number to the result. A unique secret key is used to ensure that cookie is not forged.

As an additional features, Linux uses lowest bits of the TCP timestamp option to encode ECN, S-ACK and WSCALE options.

SYN cookie significantly improves system's ability to process SYN packets. In our tests a system without any specific tuning was able to withstand 300K-350K packets per second (data is valid for a 1G network card with 4 interrupt lines). 

[ Talk about disadvantages ]

### SYNPROXY

SYN proxy iptables target was added in Linux 3.12. It is similar to syn cookies technique, but has some additional advantages and disadvantages. `synproxy` target processes packets before they hit Linux TCP stack and can be used for proxying valid packets to another server. Upon receiving SYN packet, SYNPROXY sends a cookie in response, and if it finds the response valid, it generates new SYN packet for the proxied host and rewrites sequence numbers in both directions.

![synproxy](/img/synproxy.png)

In fact, this is how most of commercial SYN flood attack protection solutions work.

This method has some undeniable advantages:

  - there's no need to use the heavy SHA1 algorithm
  - invalid packets don't reach protected server
  - it's possible to protect a server running any OS

There also are some disadvantages:

  - disadvantages of SYN cookie
  - firewall has to rewrite sequence numbers which leads to termination of all established connections when activating protection
  - equipment has to be installed in the split, meaning that all traffic to the server and from it has to pass through the proxy

We have been using iptables SYNPROXY for some time and found that this solution was suboptimal for our requirements. The power of DDOS attacks available for a dollar is increasing every day, and we're about to hit the limit of what's possible to do with SYNPROXY.

Then we turned to commercial vendors and quickly found out that their solutions are too costly ($100 000 for a piece of hardware).

That's how we came to the idea of developing our own solution which will fit our requirements.

### Our requirements

- We connect our servers with 1G links, which provide suffient bandwidth in normal operation. In case of an attack they quickly get flooded, which means we can't just use syncookies on the end server
- We can't use iptables SYNPROXY for permonance reasons
- We can't use commercial solutions because they cost too much
- We want to be able to turn protection on and off without disturbing established connections
- We want to use commodity hardware

### Our solution

Syncookied system is a further development of SYN cookies, in a distributed fashion. It consists of 3 parts:

 - **syncookied firewall** is a binary running on the firewall machine. Its task is to filter packets according to the rules in the configuration file. Syncookied firewall communicates with the syncookied server over local network and retrieves the secret keys.
 - **syncookied server** is a binary running on the protected machine. Its task is to transfer the secret key, used for SYN cookie generation, to the firewall.
 - **tcpsecrets kernel module** is a kernel module which exposes tcp secret key and the timestamp as a file in /proc filesystem to be read by syncookied server. It also installs an Ftrace handler to fool the kernel into thinking that SYN cookie was sent.

![syncookied](/img/syncookied.png)

When not under attack, the traffic between the router and the server flows directly.
In case of an attack the ARP entry for the protected IP is overriden on the switch with the MAC address of the firewall, directing the traffic flow to it.

Syncookied firewall holds an in-memory table matching protected IP addresses with the secret keys and mac addresses, which is updated at 10 second interval. Upon receiving a SYN packet it consults the table, finds the appopriate key and timestamp, and generates a cookie, using the same alghorithm Linux kernel uses. ACK packets are then validated against the same key, and if found legit, are forwarded to protected server's MAC. Protected server accepts the packet, because it was generated with its key.

In our tests we found that this system can handle up to 12Mpps SYN flood attack with 12 cores of Xeon E5-2680v3. How is this possible?

### Netmap and Rust

We evaluated multiple kernel bypass technologies for syncookied and decided to use netmap for its stability and simple API. Netmap provides userland access to network card packet buffers (called rings), excluding kernel from the data path.

Rust was choosen for its safety features and zero-cost abstractions. We haven't seen a segmentation fault or a memory leak yet. Netmap bindings are provided by netmap-sys crate, while packet parsing is handled by awesome libpnet library.

We'd like to thank open source community for their work and give back by putting our solution on Github. It's still at 0.2.x version, but we believe in "release early, release often" philosophy and hope that it will find its users and we will find new contributors.
