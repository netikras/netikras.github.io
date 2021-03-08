# Computer networks. Routing


**TL;DR;** Router is a guy who knows a guy who knows a guy who ..... who knows where your destination is. Although "should know" is more realistic than "knows".

## What, why, where
This post is hosted in GitHub servers somewhere on the planet Earth (or so I'm told). Your computer is at your house. How do bits and bytes of this post get delivered to your computer? One way is to find out where these servers are, drive/swim/fly/walk over there, get into the datacenter, find the right server, plug your USB flash drive in, copy the computer-networks-routing.md file in your drive, go back home, plug your USB drive in your computer and open that file up with your favourite markdown editor/viewer. All that assuming you get clearance to enter the DC (or to know where it is) and plug your USB drive into that server, copy files from it. 

OR.. 

or you could enter this post's HTTP URL in your browser's address bar and hit enter. The post will magically appear on your screen. But how does the browser knows where the DataCentre is located? How does it travel there? Does a browser have a map of all the DCs?

In this post I will be talking about the "map" part of the packet's journey. I'll try to observe how the packed is chosen the right route on the map and sent over to the destination. A computer networks' GPS system, if you will.

## The route
If we're planning a trip, naturally we need 3 things to get straight:
- destination of our journey - WHERE we want to end up
- route from our current location to the destination - which way do we go
- means of transportation - on foot, car, plane, bike, etc.

The same is also the case in computer networks. If we want to send a packet somewhere, we need to know WHERE we are sending it, we need the ROUTE and we need the CARRIER (transportation).

The carrier is usually the least important. The packet doesn't care whether it's traveling through the switch, router, LB, proxy or other hop.
The destination is also not that difficult of a problem. Usually we already know the address, e.g. google.com:443, 192.168.1.1:80, and so on. So the destination is any device that identifies itself with the mentioned coordinates.
This raises a problem: if I setup my device to identify as google.com:443, how will you find the real google server, rather than my laptop? We need to know the right route to the real google server.

Route in computer networks is a little different and a route from Paris to Hawaii. If you're planning a trip to Hawaii, you plan your route ahead and then follow it until you get to your destination. In CN you do not know the route in advance. Even if you did, computer networks are so wildly dynamic, that the route can change many times in one second. So instead of planing ahead, a network packet travels from one hop to another. A hop knows which direction is the destination and sends the packet to another hop that way, that hop does the same thing and this hopping is repeated until either the destination is reached, or the packet gets too old and dies on the way. 

This fits well our Paris->Hawaii analogy too. Imagine you don't have a map, you don't have a GPS, nor a compass. How do you get from Paris to Hawaii? You go to the nearest person and ask him that question. The person will point yo to the nearest information centre, so you the go there. The Info Centre will direct you to the airport and point a finger to the airport's direction. You walt that direction until you meet someone and ask that person for the way from there. And you keep hopping through the random people, Info centres, airports, harbours, other hops until you reach Hawaii. Or die along the way... whichever happens first.

This is how packets travel in Computer Networks. Packets do not know the route in advance. But they know where to start, they know when they reach their destination, and they know who to ask where to find another hop, where they could ask for another hop, where they could ask for another hop, where..... You get the picture.

Oh, and the death of the packet. Each packet has a concept of TTL (TimeToLive). It's a number, usually either 64, 128 or 256. This is how long the packet can live. This number is decremented in each hop. So if the initial TTL is 64, the packet can travel through 64 hops at most. By looking at the packet's TTL at any time along the way you can guess how many hops has the packet passed through already. I'm saying *guess*, because if we're observing the packet in-transit, we do not know what was the initial TTL. And if the packet dies, the hop it died in calls to the packet's home and informs us of its death. In CN terms - the hop sends the ICMP packet to the packet's source address informing of an expired TTL. This might come handy when investigating network issues. (I probably should say "might send the ICMP packet" rather than "sends the ICMP packet", as ICMP is often blocked in the networks).

The route could be pictured as

```
__________     _______     _______               _______     _______________
| Laptop | --> | Hop | --> | Hop | -->  ...  --> | Hop | --> | Destination |
----------     -------     -------               -------     ---------------
```

## The router - "the hop"
The *hop* is a router. It picks a next hop on the route and sends the packet there. A router is a smart network device and is capable to operate in Ethernet (OSI-2) IP protocol (OSI-3). Usually, routers are smart enough to also handle higher level protocols, like TCP/UDP (OSI-4), some might be capable to deal with protocols above OSI-4, but a home user will hardly ever have any business with such routers.

A router accepts ethernet frames which have their destination set to either ff:ff:ff:ff:ff:ff (broadcast, meaning the frame has been sent to all the devices in the network), or that router's MAC address. Your probably don't want to broadcast your HTTP packets with your facebook login information, so you must know the router's MAC address. 

Once the frame reaches the router, an IP packet is extracted from the frame's Data section. The router looks at the destination IP address and tries to figure out the next hop. To do that, the router has to:
- 1. verify whether that IP address in a part of that router's network, i.e. if both the router and the ip.dst_addr are in the same network. 
- 2. If the ip.dst_addr is in the same network:
	- 2.1. look that IP up in the ARP table to find out which device has that IP assigned
	- 2.2. if the ARP record is found:
		- 2.2.1. rewrite the Ethernet Frame's eth.src_addr with the router's MAC address
		- 2.2.2. rewrite the Ethernet Frame's eth.dst_addr with the MAC found in the ARP table
		- 2.2.3. send the modified frame to the destination device through the right physical network port
		- 2.2.4. **DONE! The packet has been delivered**
	- 2.3. if the ARP record is not found:
		- 2.3.1. if ARP discovery is enabled:
			- 2.3.1.1. broadcast (frame.dst_addr=ff:ff:ff:ff:ff:ff) an ARP request, asking for the owner of that IP address to raise a hand by replying with its MAC address (basically to say "yes, my IP address is that and my address is *this*")
			- 2.3.1.2. if any IP owners raise hands:
				- 2.3.1.2.1. update the ARP table
				- 2.3.1.2.2. **go to 2.2.**
			- 2.3.1.3. if no device identifies with this address
				- 2.3.1.3.1. extract the ip.src_addr from the IP packet
				- 2.3.1.3.2. send an ICMP response back to the sender saying that the destination is unreachable
				- 2.3.1.3.3. **FAIL and drop the packet**
		- 2.3.2. if ARP discovery is disabled (static ARP table only):
			- 2.3.2.1. **go to 2.3.1.3.**
- 3. if the ip.dst_addr is not in the same network:
	- 3.1. look at the router's routing table and try to determine which route matches that IP address
	- 3.2. if the route is found:
		- 3.2.1. rewrite frame.src_addr with router's MAC address
		- 3.2.2. forward the frame to the device configured in that routing rule
		- 3.2.3. **DONE! The next hop goes to 1.**
	- 3.3. if the routing rule is not found:
		- 3.3.1. if the router has a gateway configured (default route):
			- 3.3.1.1. take the gateway's MAC address from the ARP table
			- 3.3.1.2. rewrite frame.src_addr with router's MAC address
			- 3.3.1.3. rewrite the frame.dst_addr with that MAC address
			- 3.3.1.4. send the frame to that router's gateway
			- 3.3.1.5. **DONE! The gateway goes to 1.**
		- 3.3.2. if the router has no gateway to promote the packet to
			- 3.3.2.1. **go to 2.3.1.3.**

This above is the core rule for any router out there. The 2.3.1 might be disabled by network specialists if the router is to only rely on a static ARP table.

## Gateway
Gateway is nothing but another router. Pretty much any router can be a gateway. Routers usually have their own gateways configured, but some might not have any. A network can function perfectly without a gateway - a network router, would be enough (however, in that case a simple switch might also suffice).

Consider this network

```
                      ... [internet?]
                          ↑ ?
                     | Router C |
                         ↑  ↑  192.168.0.1
       __________________|  |_________________
       | 192.168.0.2                         | 192.168.0.3
  | Router A |                          | Router B |
       ↑ 192.168.1.1                      ↑  ↑  ↑ 192.168.2.1
       |                  ________________|  |  |_________________
       | 192.168.1.2      | 192.168.2.2      | 192.168.2.3       | 192.168.2.4
| Application |         | PC |           | Laptop |          | Smart TV |
```
Looks like a person is hosting an application in his home network. There are 3 networks depicted: netC, net A and netB. NetC network has 3 members: all the 3 routers. NetA only has 2 members: RouterA and Application server. And netB has 4 members: RouterB, PC, Laptop and a SmartTV. 

In netA the gateway is RouterA.
In netB the gateway is RouterB.
In netC the gateway is RouterC.

Gateway is a router positioned in a special place in the topology. Simply put, a gateway is the middle-man between networks. It oversees its own network and can delegate packets to its gateway, if the adresee is not in the same network. In home network a gateway is something that gates your access to the internet, and also hides your network from the internet. In industrial networks the same sentence applies, just replace "internet" with "higher level network".

Let's analyze this as a set of examples:
- PC sends a message to the RouterB. They are both in the same local network, they are even connected directly to each other. PC does not need a gateway to reach RouterB - it already knows how to reach it.
- PC sends a message to the Laptop. While they are both on the same LAN, they are not wired together, so PC needs its message to be routed to the Laptop. RouterB acts as a router.
- PC sends a message to the Application. 192.168.2.2 and 192.168.1.2 are on different networks, so PC needs a gate to the other network. Gateway 192.168.2.1 (RouterB) will relay that message to its parent network - to the RouterC. RouterC doesn't know anything about the 192.168.1.0 network, so it will also send that message to its gateway. If the gateway above RouterC is your ISP, chances are they are rejecting all the requests with ip.src_add in private IP address ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16), so eventually you will get a Network Unreachable ICMP response. If, however, RouterC gateway is another router in home network, it is possible it might know where 192.168.1.2 is, but in this case it will most likely be some other device than we expected, i.e. not the Application server from our example topology.

The application network is isolated from the home appliances' network, so the PC cannot send packets to the Application and vice versa. If we wanted to make Application accessible from the appliances' network, we should configure RouterC and add a custom rule to the routing table, which should say:

> if the ip.dst_addr is in a subnet 192.168.1.0/24, route that traffic to the hop with ip 192.168.0.2

or a more strict rule

> if the ip.dst_addr is in a subnet 192.168.1.2/32, route that traffic to the hop with ip 192.168.0.2

That would be rather enough for a NAT-enabled router. However, if we don't have NATting available, such a setup would only allow TCP traffic from PC to Application, and the Application would not be able to reply. Why? There's no route for Application to reach PC! Let's add another route:

> if the ip.dst_addr is in a subnet 192.168.2.2/32, route that traffic to the hop with ip 192.168.0.3

Now the IP packets that originate in the Application server and are sent to the PC IP will be relayed to the router that manages the network the PC is in.

Why do we need this? We wouldn't if we were communicating in UDP. UDP doesn't have any echo. But TCP or ICMP, on the other hand, does. "echo" in this context means that any message sent to one direction will cause another message sent back. TCP is a verbose protocol and involves lots, LOTS of message exchanges back and forth. It requires a bidirectional communication channel to open a connection alone, not to mention the actual data exchange.

In this case PC's packet, once it reached RouterC, will be routed to the RouterA. And we know RouterA knows how to find 192.168.1.2. We have now successfully joined two isolated networks.

## Routing table
A routing table is a set of rules instructing a router how to behave when an IP packet is received. Routing rules can be as simple as "route all packets that are addressed to B to C" or as complex as "route all packets from subnet A sent to host B to host C via interface eth3 with priority of 100 and cost of 17" or more complex. Normally simple rules are perfectly enough.
A routing table usually has a Default Route. It's a fallback route - it's used when no other routing rules match the packet. The default route has a Destination address set to 0.0.0.0, meaning it will catch all the traffic that's left after traversing the routing table.

Multiple rules can match any packet, it is perfectly legal. And, in fact, one will hardly ever find a device with just one rule matching a packet. That is the case, because a default route matches ALL packets, and other rules match just a subset of them. The narrower the rule, the higher is its priority in the table. For example, consider 3 rules:

```
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.2.1     0.0.0.0         UG    100    0        0 enp6s0
192.168.2.3     0.0.0.0         255.255.255.255 U       0    0        0 enp6s0
192.168.2.0     0.0.0.0         255.255.255.0   U       0    0        0 enp6s0
```

If we were sending a message to 192.168.2.3, the second rule would be picked up, because its mask is the strictest, i.e. it only matches 1 host. The third matches the whole subnet, while the first matches all the IPs, which makes them more loose than the second one, so their priority is lower. Think of this priority mechanism as fallbacks: try the next least loose rule until it matches the packet.

## NAT router
Normally routers do not alter the packets a lot - they only change up to 2 fields of each packet: frame.src_addr and frame.dst_addr during the routing process. These are both OSI-2 fields. However, in larger and/or more isolated networks this might not be enough. Imagine if you sent this packet to google.com:
```
frame
  src_addr: aa:aa:aa:aa:aa:aa <ignore>
  dst_addr: bb:bb:bb:bb:bb:bb <ignore>
  ip_packet
	src_addr: 192.168.2.3
	dst_addr: 216.58.208.206
	...
```

AND I sent the same packet, and I happen to have the same LAN IP address as you: 192.168.2.3.
Who would get the response? I or you? Or, perhaps, some server at google.com LAN, that so happens to have the same IP as we do?

This is wrong, confusing and succeptable to errors. A better approach is to assign ip.src_addr to some IP address that google.com can identify uniquely, without any chance of confusion. This is where the private and public IP address separation comes in handy.

We know our ISP assigns us an IP address. Some get a static one (the same IP forever), others - a dynamic one (IP changes over time). But at any point in time each IP address is unique. If you have a NAT router (which you probably do), all the packets that originate in the LAN and leave the router through the WAN port, i.e. all the packets sent to the internet, get ip.src_addr set to your public IP address. This way, even though your LAN IP address is 192.168.2.3, your packets sent to google.com will be sent as if from 35.36.37.38 (or whatever your public IP address is). But there's a caveat. If google.com wanted to send you a packet, they should use 35.36.37.38 as the ip.dst_addr. NAT effectively hides your LAN IP address and allows you to be uniquely identified by the target. Any packets sent to your external IP address will be sent to your router rather than your Laptop/PC/SmartTV 

This applies to new connections originating from the internet, and it does not apply to connections made from within the lan to the internet, i.e. if you connect to google.com and send a HTTP request - you will still receive a response to your request. This is all thanks to connection tracking built in to your router.

## PAT router
Just like NAT, PAT (Port Address Translation) also does some translation between public and local spaces.

Consider sockets. A socket is a triplet of 
- protocol (tcp/udp/icmp/...)
- IP address
- port

A socket represents either end of a connection: either local or remote. A socket is a cup in a cups-and-string-telephone. You speak to one cup (socket), the data is transferred through the wire/thread, and it's emitted through the cup (socket) on the other end. In a connection you have one cup and google has another. Each socket in either of the devices must have a unique triplet. That is, if you're sending a http request to google, your Laptop will create a socket [216.58.208.206:443:tcp], and google server will create a socket [<your_external_ip>:<your_port_number>:tcp]. Now NAT takes care of the <your_external_ip> part. What's with the <your_port_number>?

When you create a TCP socket for HTTPS communication, your OS (Windows/Linux/MacOS/other) will assign a port 443 as remote port number and any random number between 0 and 65535 (in reality the range is a lot narrower, something like 49152 to 65535) as local port. This says "I will be sending data to port 443 and expecting to receive data at port <random_number>".

Consider the same topology in the ascii-picture above. You have a PC, Laptop and SmartTV governed by RouterB. If you are sending a packet to google from your Laptop, RouterB might apply NAT rules and rewrite your packet's ip.src_addr to your router's external IP address. It will leave the same ip.src_port that your laptop has set. Then, it will forward your request to RouterC, which, if it's NAT-enabled, will do the same thing. And this continues on and on until your packet reaches the google server you were aiming for. The remote server creates a socket using TCP as protocol, your public IP address for IP address and your Laptop's assigned private (local) port number for port number. It's all nice when you only use laptop for communications with google. You might not even notice anything wrong if you used all the 3 devices simultaneously: laptop, PC and SmartTV. However, what would happen if you had 70'000 devices in your network, sharing the same public IP address? They would all NAT to the same public IP address. And they would all generate a private port number in a range [0;65535]. There are 70'000 devices. Naturally, some of the private port numbers will overlap. And this introduces a problem - several devices will be using the same protocol, IP address and port number to connect to google.com. Google server (or firewall or LB or any other device in the way) will get confused: "hey, I already have this triplet in an already established connection. I cannot register another, identical to that one! ECONNREFUSED"

The problem is that several devices in the same network might generate the same random private port number. The larger the private network, the higher the probability of private port number collisions. This might be a problem in your network as well, not neccessarily in a remote server: if your internal routers are NATting and governing large networks, your packet might find a collision in either of your network devices too, e.g. routers, firewalls, ..., because your network devices are also tracking connections (sockets) internally, to maintain conversations.

A solution is PAT. A PAT-capable router will generate its own private port number for each connection originating in your network and will rewrite the TCP/UDP packet's src_port with it and send it out. It will also keep the mapping between the original port number and the newly generated port number for incomming packets in the same connection and rewrite the tcp.dst_port accordingly. This way you make sure that even though your servers happen to generate the same private port number, the outter world will not see collisions, because PAT router will make sure they are unique.

## Port Forwarding
Now, what if you want to allow google.com to connect to your Laptop or SmartTV? What if you are having trouble using GMail and you call to google and ask aa technician to connect to your Laptop using RDP (Remote Desktop (Windows))? You will give them your public IP address - that's for sure. But how does your external router knows that it should redirect the google technitian's RDP request to yor laptop, and not to your SmartTV or PC or, perhaps, drop the packet? This is where OSI-4 features of a router come in handy.

Most routers can also operate in OSI-4, i.e. they can interpret TCP, UDP, other OSI-4 packets (segments). TCP and UDP are wrapped inside IP packets, and they also have a protocol-speciffic identifier. Picture it like this:
```
Identifiers:
- Ethernet: src_mac, dst_mac
- IP:       src_ip,  dst_ip
- TCP/UDP:  src_port, dst_port
``` 

This allows us a high level of multiplexing, where higher OSI layers grant us with even more contexts in the same TCP/UDP communication. This also allows us a more fine-grained routing to be set up. In this case, we could set up our router so that it would accept traffic at 35.36.37.38:3389/tcp and forward it to 192.168.2.3, which is your Laptop. This way, a google support person would be connecting to 35.36.37.38:3389/tcp and your router will know that this traffic is meant for your Laptop. This is called **Port Address Translation**,  or **Port Forwarding**. 

Port Forwarding even allows us to remap port numbers. Suppose you are running a nginx server on your laptop at 80/tcp, and you want to show your website to your friend remotely. You are in different home networks, so you set up a rule in your router:

```
From port      To port      Local address      Protocol
80             80           192.168.2.3        TCP
```
This is all standard and simple, no remapping is being done. Simply all the 80/tcp traffic to your public IP address is forwarded to your laptop's LAN IP.

Now imagine, you have another version of that website, just on your PC. You are also running on nginx 80/tcp. The router is already instructed to route all the :80/tcp traffic to your laptop, how do you also tell it to route other traffic to your PC? It's quite simple - you remap the port number. You add another rule, and now your port forwarding table looks like this:

```
From port      To port      Local address      Protocol
80             80           192.168.2.3        TCP
81             80           192.168.2.2        TCP
```

This means, that your friend can go to 35.36.37.38:80/tcp, and it will be forwarded to your laptop (192.168.2.3:80/tcp), or he could go to 35.36.37.38:81/tcp and he will be forwarded to your PC (192.168.2.2:80/tcp). You are remapping external (From) and internal (To) port sumbers.



## References
- https://lartc.org/howto/lartc.rpdb.html
- http://wiki.wlug.org.nz/SourceBasedRouting
- https://www.coursehero.com/file/p12iiks/returns-to-the-router-it-uses-the-connection-tracking-data-it-stored-during-the/
- https://www.wikiwand.com/en/Default_gateway
- https://www.ciscozine.com/nat-and-pat-a-complete-explanation/
- https://www.sciencedirect.com/topics/computer-science/registered-port#:~:text=Port%20numbers%20can%20run%20from,application%20processes%20on%20other%20hosts.

> Written with [StackEdit](https://stackedit.io/).
