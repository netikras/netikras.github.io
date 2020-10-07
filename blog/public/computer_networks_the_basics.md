

# Computer networks - the basics: OSI and devices

When you spin-up a REST web server you expect clients to connect to it and send you an HTTP request. Your webserver will accept that request, analyze its headers, payload, apply business rules, maybe reach out to the database, MQ, will build a response and send it out to the requestor. How simple is that, right? But is it though....?

Programming libraries do a very good job of protecting us from the complexity of networking. They act as an abstraction (or rather their APIs do) to the actual networking. And they do their job pretty well! Until something goes south. And when the network debugging begins programming frameworks/APIs are of little help. Well maybe with an exception of the OS API (which is a programming API as well). In such cases, it's quite handy to know how computer networks work and what devices are involved.

## OSI
The ones of you who have studied any part of the CS are probably aware of the OSI concept. It's a model classifying network components into 7 groups stacked on each other - layers:

1. **Physical layer** - this is the actual ones-and-zeroes in a form of the digital signal and the medium they travel in. It's the  hardware: NICs, cables, hubs, etc. The "physical" in the name kind of hints it, doesn't it! This is the only layer your applications cannot "see" the data in.
2. **Data link layer** is built on top of the physical layer. In this layer bits and bytes are coupled together into packets. These packets usually have a header containing  some metainformation, like source and destination devices' addresses. Devices in this layer are able to forward data to each another in the same network based on low-level address bits found in the data packets.
3. **Network layer** is yet another layer that has source/destination address, except here addresses are higher-level and can point to different networks. This is where the IP protocol resides in. Data-link layer helps to transmit data between devices that are either directly connected to each other or interconnected using a switch or any other OSI-L2 device. Network-layer, on the other hand, builds a higher-level addressing system (think **IP**) that can have source/destination addresses from different networks. That's a luxury Data-Link cannot offer.
4. **Transport layer** - now that's finally something you can use in your application. The physical layer contains the actual bits and bytes, data-link and network layer are driving addressing and packaging. When you think of Transport layer, think TCP or UDP. These are the most common protocols that reside in the Transport layer.
5. **Session layer**, just like the name states, contains sessions. It's a logical concept, meaning a context of communication. In this context parties can establish trust, agree on some preferences regarding the communication and the session itself. Sessions are usually established between two parties sharing the same time-bound unique value. As long as both (all the relevant) parties hold that value AND the value is not expired (time-bound, remember?), it's said that both the parties have an established session. In order to establish a session parties may require to authenticate each other first, and, if authorized, generate and exchange the session identifier that represents the established trust. This concept lives in the Session layer of the OSI.
6. **Presentation layer** is a proxy between the session and application layers. In this proxy data is prepared for the application to understand. In other words, transport layer passes session-bound data to the presentation layer, where this data is processed (decrypted, reencoded to a format application can understand, unzipped, etc.), and then passed on to the application.
7. **Application layer** - at last the layer at which application-specific data structures are used. For web developers this is the HTTP: headers+newline+payload(+newline). This is the layer at which an application reads data it can understand and writes data to another application in the application-specific format (protocol).

This model has been adopted since long ago and nowadays we have situations in which it's difficult to assign a protocol to an OSI layer. For instance, which layer does an HTTP packet in a UDP-based VPN session belong to? Which layer do HTTP packets belong to when transferred from inside a virtual machine through a virtualized NIC?

However, while some aspects might be debatable, the OSI model is correct and all the network communication is taking place according to OSI. Why is it useful to know? Well, for starters it helps to understand some of the protocols - knowing which OSI level it belongs to explains the protocol's purpose alone. It helps to design the custom application protocol to communicate on the right levels. Also, network devices are, in often cases, quite smart and they can help you boost application performance if you allow them to at particular OSI levels, e.g. SSL termination (OSI6), time-based unused session termination (to save resources) (OSI5), application data caching (OSI7), etc.

## Network devices

There are multiple kinds of devices that can be plugged into the network. A computer (i.e. a terminal device) is one of them - it's a device you and I interact with directly on a daily basis. But there are quite a few more of them. When you open up some social network website and browse in there all the network packets created by your browser travel through all of the kinds of network devices there are.
All of them, except for cables, can be both: physical and virtual. Physical devices are.. physical :) They are either wires or SoC systems that have one or more network ports. Virtual devices, on the other hand, are either OS kernel space or user-space applications that perform what a physical device would do. They are less efficient performance-wise, but significantly more flexible and can be applied to virtual environments as well as the physical ones.

### Cables
Yepp, cables are also classified as network devices! And they fall into the OSI1 layer. There are quite some kinds of cables, let's not get into this for now - we are not going to be building our own datacenter, are we now :) The point is that cables are there to physically connect 2 devices with each other and transfer data between them. Some devices can even be powered via network cables using the PoE (Power over Ethernet) technology.

### NIC
Network Interface Controllers or, as usually called, Network Cards / Network Adapters. These are the devices in your computer (and other network devices) that you plug network wires into. Usually, they are either wired into your computer's motherboard or installed as PCI cards. While a NIC usually associated with an ethernet controller, WiFi adapters also fall into this category. Except they use invisible wires -- Wi-Fi radio waves.

Now that your computer has a network thread connected to it, where is the other end of that wire?

### Hub
A hub is the "dumbest" network device of all (let's ignore cables for a moment). They operate in OSI1, so they are not aware of any network protocols. All they can do is to boost a network signal. If a hub has multiple ethernet ports, it will send the same packet to all the ports. It's like an ethernet cable splitter that is better to be avoided unless you need to boost your signal (but then again, there are extenders/repeaters for that) as it adds more noise to the LAN and introduces security issues.

### Switch
If you are in a company with a large-ish network your ethernet cable is most likely plugged into a switch. Switches usually are quite dumb devices (except for the expensive ones - enterprise-grade smart switches). They operate in OSI2 layer. That means they are only aware of ethernet frames. Ethernet frames, as I'll explain in other networking posts, contain 4 pieces of data: 
- frame type - that's just 2 bytes hinting layer2 devices what kind of data does that frame contain
- source address - a MAC address of a sending device's NIC
- destination address - a MAC address of the device that should receive that packet
- the payload - the data

What a switch does, it holds a forwarding table and routes Ethernet frames according to it. It's like 'the middle-man, the guy who knows how to reach that other guy'. Switches have several physical ethernet ports in them - that's where network cables are plugged into. Switches monitor ethernet packets being sent through those ports and determine what MAC addresses are at the end of each network cable. That way a switch knows what MAC can be reached if a packet is sent out through either of its ethernet ports. Believe it or not, switches, being so dumb, add a layer of security to the network as they prevent promiscuous NICs from receiving traffic that's not meant for them.

### Router
That's like a higher level of a switch. Except routers rarely know how to operate at layer 2 -- they sit comfortably at the OSI3. A router is something you most likely connect your computer to at home. It knows what IP addresses are and operates similarly to a switch in that sense. While a switch uses MAC addresses to forward packets, a router uses IP addresses for that purpose. Switches are more limited in a sense of what addresses can it serve - usually, switches are only aware of how to find a device that is connected directly to that particular switch (or a cluster of switches). A router, on the other hand, operates on IP addresses, which a router might or might not have assigned to an immediate client.
All routers have an ARP table - that's a mapping between IP and MAC addresses of the router's immediate clients. If a router receives a packet that is destined to 1.1.0.3, it checks it's ARP table to see if it knows what MAC address does the owner of that IP address has. If the router finds the 1.1.0.3 - MAC mapping in it's ARP table it forwards that packet to the found MAC address, i.e. forms a new ethernet packet, writes original sender's MAC as the sender and the found MAC as a destination, appends the payload and drops that frame to it's LAN. The owner of that MAC address should pick the package up and receive the data.

There is that other thing routers can do -- it's called **routing**. If there are multiple routers connected to each other, managing their own networks and all, a package, originated in one network, can be routed to another. This happens as follows. A computer sends a package from 1.1.0.3 to 1.1.2.4. These IPs are in different networks (assume netmask /24). The package is sent to the router, it looks up its ARP table for the 1.1.2.4 address -- no matches. The router looks up its routing tables - maybe there's another router that knows how to reach the 1.1.2.0/24 network. It turns out there is a routing entry in that router that explicitly defines where to send packages destined to that network. Hooray! Our router delivers the request to that router's MAC and hopes it'll know what to do next. And that's actually all to it. Should the 1.1.2.4 want to send a response, its gateway should know how to route packets to the 1.1.0.1/24 network - that's the task for network admins to preconfigure.

Usually, routers are even smarter than that. They can often operate in OSI4 layer - they know of TCP/UDP ports, they can forward or block them (an integrated firewall), most routers can act as NAT devices and gateways, they host a DNS server, have a firewall, and so on. So all in all routers are pretty intelligent devices.

### Gateway
Gateways are devices connecting several networks to each other. Does that sound like something a router would do? Sure it does. Routers usually do act as a gateway. However, you'll most likely only see a router that operates as an OSI3 gateway. Hardware gateways, on the other hand, are a bit more advanced in that manner and can operate in either of the OSI layers. 

### NAT
NATs are OSI3 devices, just like routers. Many (probably MOST of) routers can act as NAT devices as well. NAT (Network Address Translation) is a device hiding a LAN IPs behind NAT device's external IP. I.e. if a request originates in LAN and is passed on through a NAT device, the nat device will remember which device sent that request (its IP address), rewrite the SRC_IP_ADDRESS with its own IP address (could be an external company's IP address) and pass on the request. Upon a response packet is received, a NAT will find the receiver's IP address in its connections' tracking table, rewrite the DST_IP_ADDRESS and pass the request back to LAN.

### VPN
Virtual Private Network is a technology allowing to create a virtual LAN. A VPN server accepts VPN client connections. Once authenticated, a new virtual NIC is created in the client's computer and a new LAN address is assigned to it. The same thing happens to another computer connecting to the same VPN server, trying to access the same virtual LAN. Both the devices can now reach each other, no matter how far from each other physically they are -- they are in a virtual LAN. VPNs can be both hardware and software-driven.

### Bridge
A bridge is an OSI2 device joining multiple networks. They are very similar to switches, although switches, especially enterprise-grade, tend to be more configurable, "smarter" and more efficient. 

### Firewall
Firewalls are nothing but traffic filters. Like LBs, firewalls usually operate in OSI3, OSI4 and OSI7 layers. The purpose of firewalls is to either allow or deny packets based on their properties. Rules, evaluating those properties and making the GO/NO GO decision, must be explicitly set up by an administrator. This means that a firewall is nothing but a large list of rules, deciding whether to pass a packet through or not.

### WAF
Web Application Firewalls are designed specifically to protect web-applications and they mostly operate on OSI7, OSI6 and OSI5. They protect web applications in many aspects: session protection, DoS protection (up to a point), protection from some generic attacks, protection from application-specific exploits (might require custom configuration) and so. It's a specific implementation of a firewall, which means that it is there to filter out potentially malicious packets and reject such requests. A WAF can also be used to restrict access to the application so that particular resources are only accessible to those who are authorized to, e.g. site's admin page shall not be visible to anyone except administrators operating from IP range x.x.x.x/xx.

### Load Balancer
Load balancers can operate in several OSI layers: OSI3, OSI4, OSI7. The lower level, the more efficient, though less configurable balancing can be. The purpose of LBs is to distribute incoming traffic among (sub)set of connected downstream devices. LBs can use several methods to distribute the traffic, but _random_ and _round-robin_ (looping through all the devices sequentially on and on) are the most popular modes. More advanced LBs can make more sophisticated decisions, evaluating connections' count to each downstream device, response times, availability, etc.

### Proxy
There are many kinds of proxies and they all can operate more or less in any OSI layer (except for, maybe, OSI1). The purpose of a proxy is to act as a distant NAT. An application sends a packet to a proxy asking it to delegate a request to some other address. The proxy issues that request and returns a response, if there is any, to the original issuer. Suppose 1.1.0.3 sends an HTTP request to google.com through a proxy, which has an external IP address 5.5.5.5. The proxy issues the request to google.com without a single hint of 1.1.0.3 -- google.com logs will only see 5.5.5.5 as the request issuer. Anything google.com responds will be returned back to 1.1.0.3.

Very often proxies are overlooked and a VPN service is picked where a simple proxy would suffice.

## Security
All the network protocols we are using today have been invented and applied a long ago - most of them reach back to the previous century. Back in those days, security was not a concern - networks were a very fresh concept only used locally or semi-locally. No one bothered to make network protocols secure from eavesdropping. There was no one to eavesdrop :) 

When the internet happened computers' networks exploded. They started to grow at rates their authors haven't imagined. As computers got more popular, parties with malicious intentions emerged. And since network protocols are not secure, to this day we suffer from cyber-attacks other than just social engineering.

### Promiscuous mode of NICs
There's this interesting thing about NICs. They can operate in two modes: normal and promiscuous. 

In promiscuous/monitor mode a NIC accepts all the traffic that reaches it. This means that if ethernet hubs or wifi access points are used in the network, all the devices connected to them will receive the same packets and all the devices will accept them. That's a serious security issue as if a malicious party joins the same network it can capture all the traffic that's been sent or received by all the devices in that network.

In normal operation mode, NICs can still receive all the packets but they will only accept those that are sent to that particular NIC's MAC address. This means that even though the network setup is not safe, NICs will not allow user-space processes to see what's not theirs. However, if a user has administrative privileges on said devices it can switch NIC to promiscuous mode and leak the data. Tools like `Wireshark`, `tcpdump`, `airmon-ng` and similar operate in promiscuous mode and they can monitor all the traffic that is accepted by the NIC. 

A word of warning: bear in mind that malicious parties can always build their own NICs and wifi antennae/adapters (or purchase them from someone else who can) that operate in promiscuous mode by default. That being said, never trust a terminal device will be always there to protect all the other terminal devices' traffic.

### How to be safe
First and foremost - set your network up correctly. Form your networks isolated from each other as much as possible, avoid shortcuts and unsafe network devices. Only allow authorized devices to join the network (static IP addresses, MAC whitelists, etc.).

Then there are firewalls. Some of them can operate as multiple network devices, which allows you to make very flexible network security rules. A rule of thumb - always try to use firewalls in restrictive mode and build whitelists. And make firewall rules as restrictive as possible.

Use as little external resources as possible. Try to avoid public DNS servers, public mail servers, etc., minimize requests to external systems as much as possible. All external systems can be breached and malicious content can be returned to your network if you make calls to them. Do not make a mistake of thinking that a malicious party is going to actively attack your network to gain unauthorized access. Threats can come from within your network - someone can click on a suspicious link and download a malware that infects your network and uses your devices as puppets controlled by a puppeteer hiding somewhere in the network. Or worse - the malware can damage your data and devices. Thus using as little external resources as possible and operating in restrictive mode is a technique a secure network must have adopted.

Partition your networks. Classify your network terminals into categories and place them in separate networks. Some terminals might require communication to external systems -- put them in DMZs. Others might store highly sensitive data (databases) - create a separate subnet for them. Some might only be accessed by multiple clients in the company's network - that's a separate subnet (or a network) right there. Think of other categories to group your devices into even smaller groups. It might be wise to partition by applications as well as other factors, which might end up in having separate networks or subnets for each cluster of application instances (I'm using the term 'application' in its broad meaning, i.e. an application can be a web application, a DNS server, a database or anything else).

Encrypt your traffic where possible. While encryption adds an overhead to the end-to-end traffic, it sure does boost your security. If secure encryption techniques are used you'll have a significantly lower chance of your company's data being leaked and your network being breached.

Newer modifications of some protocols keep getting released. If possible, upgrade your network infrastructure and programs to use safer versions of protocols and disable the ones that are deemed insecure.

## Sum-up
Computer networks are a complicated topic at first. I believe it's because it makes our lives so much easier: it's so simple to just click a button in your browser and get a response from your application! Anything that threatens this simplified model, e.g. finding out how many layers, solutions, requirements and practices are beneath that simplicity, looks scary and very complicated. But in fact, there's not that much to know about computer networks; at least as far as basics are concerned. As in your application, networks are a structure that can suffer from both overloading and security issues, networks are something that requires some configuration to make things work. Networks, like your applications, are structured into layers and components for easier maintenance and better security. So in a way networks are very much like applications you write, except they are less flexible and usually reside in the physical world.


## Sources
- [https://www.networkworld.com/article/3239677/the-osi-model-explained-how-to-understand-and-remember-the-7-layer-network-model.html](https://www.networkworld.com/article/3239677/the-osi-model-explained-how-to-understand-and-remember-the-7-layer-network-model.html)Network protocols - the basics

All network communications are based on protocols. A stream of bytes is structurized, riddled with patterns. These patterns are used to split the stream into data segments which have a very strict structure. Most of the protocols have a header

- promiscuous mode
- 

> Written with [StackEdit](https://stackedit.io/).
