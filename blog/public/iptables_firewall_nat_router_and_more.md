# Iptables - a beast worth training: a firewall, a router, a NAT router, a port-forwarder, a load-balancer, anti-DoS, a network logger,... for free!

In a previous post, I have introduced iptables and explained a little about how it works. Now that you understand what a chain and a target are we can move on to implementing some handy network configurations.

As the title says iptables is so much more than just a firewall. It's a swiss army knife of networking. Using default features alone one can easily build a Linux-based smart home router using a computer, or a smart router for virtual machines on an ESX; or a smart router for containers in a Kubernetes cluster. 


## A firewall
First and foremost iptables is a network filter's UI. Filtering, by definition, allows some units to pass through while others get blocked. This is what a basic firewall does. And this is the default mode of iptables (default table is *filter*) -  a firewall.

To use iptables as a firewall is plain simple. You have to specify a matcher for a packet, which could be as specific as you want, and an action that's to be taken once a matching packet is found -- the *target* part of the rule. A matcher must contain at least the direction the packet is moving (i.e. Netfilter chain): INPUT (INto the computer), OUTPUT (OUT of the computer) or FORWARD (through the computer). The matcher part can also specify other properties of the packet: protocol, MAC address, source/destination IP address, source/destination port. However, this elaboration is optional - an iptables rule is perfectly correct without it. Bellow are some firewall rules' examples.

### Examples

```
1. iptables -A INPUT -p tcp --dport 22 -j ACCEPT
2. iptables -A INPUT -j DROP
3. iptables -A INPUT -p tcp 8080 -j ACCEPT
4. iptables -I INPUT -p tcp 5432 -j ACCEPT
5. iptables -I OUTPUT -i eth1 -j ACCEPT

6. iptables -A OUTPUT -d 192.168.0.1 -p udp --dport 53 -j ACCEPT
7. iptables -A OUTPUT -j REJECT
8. iptables -I OUTPUT -d 192.168.0.0/24 -p tcp --dport 443 -j ACCEPT
```

### Explanation
1. Accept all TCP connections to 22/tcp port on this device. This means "Allow me to connect to this computer using ssh"
2. Drop all the traffic that did not match any rules before this one.
3. Accept all TCP connections to port 8080/tcp on this device. HOWEVER, this rule will not work as it is appended after the rule #2, which says to drop the packet.
4. Accept all TCP connections to port 5432/tcp on this device. This rule will work because it is inserted in the ruleset -- not appended. Note the difference between **-A** and **-I** (capital 'i'). -I inserts a rule to the top of the list. -A appends a rule to the bottom of the ruleset.
5. Allow all the incoming traffic to this computer from any network device with a condition that the traffic is received through an interface *eth1*.
6. Accept all the outgoing UDP connections to 192.168.0.1:53. This means "Allow me to use my router's built-in DNS server"
7. Reject all the outgoing traffic originated in this computer that did not match any rules before this one.
8. Allow all the traffic from this computer to any other computer having IP 192.168.0.1-124. Only allow traffic to 443/tcp addressed on those computers. This means "Do not block my computer to access HTTPS webservers in my LAN ". Please note that this rule is inserted rather than appended and it will work.


## A router
INPUT chain grabs all the traffic that originates OUTSIDE the computer and is addressed TO the computer.
OUTPUT chain grabs all the traffic that originates IN that computer and is addressed to some OTHER computer.
FORWARD is a different kind of chain. It gets all the traffic that originated OUTSIDE the computer and is addressed to some OTHER computer in the network but the one we are setting iptables on. Doesn't it sound like something a router would do?

If the source is not our server's IP address and the destination is not our server's IP address, then how did that packet get to our server in the first place?  I can think of 2 scenarios that wuold allow this:

- network interface in promiscuous mode. This is the case for network hubs (don't use them). Network cards in promiscuous mode (more on that in another post) accept ALL the ethernet frames travelling in LAN (a way to sniff LAN's traffic that is). When promiscuous mode is disabled (default state of a NIC) network cards only accept ethernet frames that have *destination_mac* matching that card's MAC address. So at ETHERNET protocol level, a frame is addressed to our server, but an IP protocol packet in that frame has a different *destination_address* set. 
- the ethernet frame destination_mac matches the router's destination_mac. This is the case for gateways.

Once the packet reaches the router, it's then a job for ARP and routing tables to find the next *hop* for that packet and for a conntrack module to know how to return the packet to the original requester. And so the received IP packets that have *src_address* and *dst_address* irrelevant to our server get into the FORWARD chain.

To turn a server into a router first we have to enable routing in kernel level. That's easy and it does not even require a restart: edit a file `/etc/sysctl.conf` and add the following line:
```
net.ipv4.ip_forward = 1
```
And then run a command `sysctl -p` to reload kernel settings. Now routing *per se* should be enabled. It's disabled by default for security reasons as it is simple to misconfigure (or use default settings for) a firewall to turn a computer into a router. If all computers in the network were routers spreading an anonymous attack would be very easy.

Now as we have routing enabled we can go ahead and set it up.
```
1. iptables -I FORWARD -s 192.168.0.0/24 -d 10.10.7.0/24 -j ACCEPT
2. iptables -I FORWARD -s 10.10.7.0/24 -d 192.168.0.0/24 -m conntract --ctstate ESTABLISHED -j ACCEPT
```
That's it. These two rules join two networks: 192.168.0.0/24 and 10.10.7.0/24. Any queries passed on to 10.10.7.0/24 through 192.168.0.1 as the gateway will be routed to the correct destination and responses will be returned. However, this rule restricts 10.10.7.0/24 network from accessing 192.168.0.0/24 -- that's the `ctstate`'s job. All connections in Netfilter are being tracked (conntrack). When Netfilter is instructed to only match `--ctstate ESTABLISHED` connections it will ignore NEW connection requests. This sort of rule will only act on connections that have already been established. To rephrase -- Netfilter only allows NEW connection from the 192.x.x.x network and only already ESTABLISHED connections can transfer data from 10.10.x.x to 192.168.x.x. Should you want to make both networks see each other remove the `-m conntrack --ctstate ESTABLISHED` part of the rule.

**HOWEVER** keep in mind that this kind of routing requires to update routing tables on both the networks. This means that a device on 10.10.7.0/24 must have an explicit route added to routing table specifying to use our device for queries to the 192.168.0.0/24 network. And vice versa: 192.168.0.0/24 must have a route in routing table specifying that 10.10.7.0/24 network is accessible through 192.168.0.1. Otherwise, a server on a receiving network will not know how to return a response: received IP packet's SRC_ADDR header contains an IP address on another network and the server has no idea how to reach it

In case you cannot or do not want to edit routing tables yourself on one of the networks you can try out a form of NAT'ing - SNAT'ing.


## A NAT router

Now that we have a basic router, let's try something more advanced - a technology that has been created as a hack: NAT'ing. A picture below illustrates what NAT does quite well:

![NAT in a meme](https://techstat.net/wp-content/uploads/2017/06/img_5936c079e7b58.png)

Devices behind nat are sending packets to the outside through a NAT router. NAT router rewrites packet's source IP address with its own and passes it on to the next hop. When a remote device returns a response to NAT router, the router looks at its connection tracker and finds what should be the real recipient of that package, overwrites IP DST_ADDR header and passes that packet on to it's LAN. NAT routers are the usual home routers. Have you ever noticed how whatsmyip.org shows your external IP address instead of your computer's 192.168.0.3? That's what NAT does. It makes all your requests look like they came from the router itself while keeping a record of how to deliver a returning response to the correct computer.

Let's return to our 192.168.0.0/24 and 10.10.7.0/24 networks. Let's assume that our home network is 192.168.0.0/24 and there's a device at 192.168.0.4 which runs a couple of docker containers. And docker on that computer has its network set to 10.10.7.0/24. Our goal is to reach a container 10.10.7.15 from a device with IP 192.168.0.6.

```
            [gateway]
           192.168.0.1
     ___________|__________________________....
    |                     |            |
192.168.0.6          192.168.0.4
                      10.10.7.1
                    ______|______________....
                    |            |      |
                10.10.7.15
```

First, we need to allow FORWARD rules in 192.168.0.4 as we did in our simple router's example

```
1. iptables -I FORWARD -s 192.168.0.6/32 -d 10.10.7.15/32 -j ACCEPT
2. iptables -I FORWARD -s 10.10.7.15/32 -d 192.168.0.6/32 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

Now network traffic from 192.168.0.6 should reach the container just fine! But there's a problem - the container does not know how to return that traffic. It does not know where to find 192.168.0.6! It's running in a completely different network. We go around that by SNAT'ing requests from outer network to the container. SNAT is a form of NAT - it only rewrites *SRC_ADDR* part of the IP packet. 192.168.0.6 will send a packet meant for 10.10.7.1 to 192.168.0.4's MAC. The packet will be picked up by FORWARD chains and SNAT'ed - packet's *SRC_ADDR* will be changed from 192.168.0.6 to 10.10.7.1. When container receives a packet it will see 10.10.7.1 as it's sender, so naturally, it'll send a response to that IP address. There our Netfilter will intercept the request and restore 192.168.0.6 as packet's receiver.

I know this theory probably sounds difficult to read. Try reading it again and following the thought on the ascii network chart a little above.

Enough talk - let's do some code.
```
3. iptables -t nat -I POSTROUTING -s 192.168.0.6/32 -d 10.10.7.15/32 -j SNAT --to 10.10.7.1
```

And that should be it. The SNAT target is only available at POSTROUTING chain as it's the network packet's phase before leaving the device. Connection tracker will keep a track of connection's state and will intercept all the related network packets to do the required address rewritings.



## A port-forwarder
Port forwarding is a concept of accepting a package on port X and delivering it to port Y. Suppose you have an application in your server that runs on port 8080/tcp. But you want it to be accessible via 80/tcp without changing application's configuration. That's the job for port forwarding. Iptables can be set up to accept connections on port 80 and forward them to 8080. Not only that, but it's perfectly possible to forward from address 192.168.0.6:80 to another address 127.0.0.1:8080. And this is the iptables rule for that:
```
iptables -t nat -I PREROUTING -d 192.168.0.6 -p tcp --dport 80 -j DNAT --to 127.0.0.1:8080
```

Iptables rules are quite easy to read and understand when reading them carefully. This rule is added to PREROUTING chain -- we need this modification to happen before the packet reaches our application. Netfilter matches packets sent to computer's LAN IP address at port 80 and DNATs them. DNAT is another form of NATing - it rewrites the packet's destination. By the time a packet arrived at this server's doorstep, it had its *DST_ADDR* set to 192.168.0.6 and *DST_PORT* set to 80. Now we need to change those coordinates to what we need: 127.0.0.1 and 8080 respectively. And once changed we proceed to other chains/tables unless the packet is either discarded (dropped/rejected) or delivered to the application.

There is another use case for port forwards. Suppose your application runs on 192.168.0.6, it can only query localhost address and it needs to talk to 192.168.0.4:80. That is also doable with iptables. Except for this time, a network packet is not coming to the device from LAN - instead, it's created in the device itself and is being sent out. For this reason we 
need to use the OUTPUT chain.

```
iptables -t nat -I OUTPUT -o lo -p tcp -s 127.0.0.1 --dport 80 -j DNAT --to 192.168.0.4:80
```

This rule captures all packets sent to **lo**opback device at address 127.0.0.1 (remember that loopback's network is 127.0.0.0/8) and port 80, rewrites its destination IP and port and passes it down the line. 

## A load-balancer
iptables is perfectly capable to act as a network load-balancer. Probably the only drawback of iptables-based LB is that it's static - if you added more nodes to your application cluster or removed some of them you'll have to manually add or remove them from iptables in all the concerned devices. Also, iptables will not detect destination servers that are not working. So if you want an advanced LB then you should look into some other solutions. But if a simple and dumb LB is enough for your needs then look no further -- iptables is free, powerful and preinstalled on most of the Linux devices already!

Iptables can balance traffic in two modes (like most LBs): randomly and in round-robin fashion.

### Random LB
Random LB chooses the packet receiver randomly. Each downstream server has almost equal possibility to receive the packet. Iptables implements random LB by setting a probability the rule will "take" the package. For an equally distributed random LB the probability must be divided equally across all the downstream servers. However, this probability setting allows us to distribute load unequally: some (more powerful) servers would get more traffic as other (weaker) ones would receive less of it.

>    Here's a fun-fact. Kubernetes uses iptables for traffic balancing. Netfilter operates entirely in kernel space which makes iptables the most performant LB there is. Packets never have to leave and reenter kernel space at all.

Consider our previous LAN/docker example. The commands below distribute load among three docker containers:

```
iptables -A PREROUTING -t nat -p tcp -d 192.168.0.4 --dport 80 \
         -m statistic --mode random --probability 0.33 -j DNAT --to-destination 10.10.7.14:8080
iptables -A PREROUTING -t nat -p tcp -d 192.168.0.4 --dport 80 \
         -m statistic --mode random --probability 0.5  -j DNAT --to-destination 10.10.7.15:8080
iptables -A PREROUTING -t nat -p tcp -d 192.168.0.4 --dport 80 \
                                                       -j DNAT --to-destination 10.10.7.16:8080
```

Please note the **--probability** values. They are not equal, are they? Do you see why?

The first rule consumes ~33% of the incoming packets. The rest ~67% are passed on to further rules.

The second rule consumes 50% of the packets that made it to that rule - that would be ~67% of total packages. This means 67/2 == ~33% again. So ~33% of total traffic will be consumed by the second rule and the remaining 33% will be passed on.

The third rule receives 33% of total traffic and it consumes it all -- the statistic matcher is not even defined as it would say 100% anyway.

Setting probabilities with equal distribution is not as simple as writing 33% or 25% or similar values on each rule. One needs to do some calculations for each entry.

### Round-robin LB
A round-robin LB always passes a packet to the *next* IP; if all the IPs have been served already - the packet is then passed on to the first address again.

```
iptables -A PREROUTING -t nat -p tcp -d 192.168.0.4 --dport 80 \
         -m statistic --mode nth --every 3 --packet 0 -j DNAT --to-destination 10.10.7.14:8080
iptables -A PREROUTING -t nat -p tcp -d 192.168.0.4 --dport 80 \
         -m statistic --mode nth --every 2 --packet 0 -j DNAT --to-destination 10.10.7.15:8080
iptables -A PREROUTING -t nat -p tcp -d 192.168.0.4 --dport 80 \
                                                      -j DNAT --to-destination 10.10.7.16:8080
```

Please note the **--every** parameter's value. It's calculated using the same principles a random LB's **--probability** is calculated. The first rule consumes every 3rd matching packet that is delivered to it. The second rule consumes every 2nd packet that reaches it. And the last rule consumes all the (remaining) packets that reach it.

So there it is. A perfectly functional, although difficult to maintain, kernel-space load balancer using nothing but iptables.

## Anti-DoS
There are at least 2 ways to use iptables to mitigate DoS attacks.

### Limiting connections

```
1. iptables -A INPUT -p tcp --syn --dport 80 \
            -m connlimit --connlimit-above 15 --connlimit-mask 32 -j REJECT --reject-with tcp-reset
2. iptables -A INPUT -m state --state RELATED,ESTABLISHED \
            -m limit --limit 50/second --limit-burst 150 -j ACCEPT
```

The first approach limits amount of parallel connections each IP address can have to this device. The 16th and all other simultaneous connections will be rejected. This particular rule allows each IPv4 address to make 15 simultaneous connections to this device. If we changed **--connlimit-mask** value to, say, 24, then the whole network (whatever network is querying our server) can have no more than 15 open connections at the same time. This value defines a netmask - a CIDR number that indirectly defines how many network devices are sharing the same connections' counter.

### Limiting request rate

The second approach limits the request rate from each IP address separately. The 2nd rule above limits requests rate to 50 requests per second. This rule is implemented in a rather interesting way. This filter has a "bucket" with 150 tokens. Each request coming into the server takes one token. Each second Kernel adds 50 new tokens. When the bucket becomes empty, i.e. when all the tokens have been used before the bucket got a refill, a packet is rejected.

These two simple rules allow us to mitigate simple DoS attacks quite well. Of course, there are more sophisticated kinds of attacks which could bypass those two rules, but this is a very solid start at the very least.

## A network logger
No, iptables cannot log the whole packet. Not that it would be wise anyway... But iptables can log packets' metadata - headers. To log a packet add an iptables rule with a target: `-j LOG`. This target is non-terminating and it does not consume the message. Once logged the packet will continue traversing further Netfilter rules in the same table@chain. By default, iptables uses a Syslog daemon as its logging engine, which means log entries are made to /var/log/messages, /var/log/kern.log or similar Syslog files. Changing the log file is not exactly possible. One could change iptables' log level with a `--log-level 4` parameter and configure Syslog to write all kernel warnings (log level 4) to another file, e.g. *iptables.log*, However, this approach will likely capture some other messages, not just the iptables' log entries.

An example network logging rule:

```
iptables -I INPUT  -m conntrack --ctstate NEW -j LOG --log-prefix="New incomming: "
```

This rule will write Syslog entries about each new connection. The matcher can be changed to be more specific and only log a part of the inbound traffic. Each logged message will be prefixed with a string: *"New incoming: "*. Prefixing log entries is especially useful when an iptables setup has multiple logging rules defined.

>    Logging rules are also helpful for debugging purposes. If you are debugging your iptables rules and wondering at which point your rules stopped matching your packet - just add some logging rules to see how far does the packet go. 

>    Feel free to also use logging rules to craft matches for new rules. This works quite well in PROD environments too as logging rules do not terminate Netfilter traversal - traffic will not be impacted.


# A sum-up
Iptables is one powerful and valuable tool a backend engineer must have under his belt. It might take time to communicate with NET-Ops to get all the rules set up in network devices: firewall, anti-DoS, routers, LBs, etc. Using iptables is a lot quicker and more flexible approach. You can use iptables to have all those large, expensive not-accessible-by-you devices in a single Linux computer at your complete control (assuming you have *root* access). 

This tool is rather useful for designing an environment, making hotfixes/workarounds, setting up some temporary solutions either in your local environment or somewhere else, simulating network events, etc. It's also good to know how iptables works when you're dealing with Kubernetes since Kube-proxy is using iptables mode as default, which means all the load balancing, routing and other network-related rules are mostly implemented with iptables.


# Sources
- `man 8 iptables-extensions`
- [https://www.linuxtopia.org/Linux_Firewall_iptables/x2682.html](https://www.linuxtopia.org/Linux_Firewall_iptables/x2682.html)
- [http://www.netfilter.org/documentation/HOWTO/packet-filtering-HOWTO-7.html](http://www.netfilter.org/documentation/HOWTO/packet-filtering-HOWTO-7.html)
- [https://serverfault.com/questions/485400/what-exactly-do-limit-1-s-and-limit-burst-mean-in-iptables-rules](https://serverfault.com/questions/485400/what-exactly-do-limit-1-s-and-limit-burst-mean-in-iptables-rules)
- 

> Written with [StackEdit](https://stackedit.io/).
