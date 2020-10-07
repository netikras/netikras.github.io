
# Iptables - a beast worth training: netfilter, tables, and chains

Network communication is unsafe by design. It's not like computer networking has been designed to be insecure on purpose. It's just that security has never been and probably never will be the primary goal of networking. The primary goal of networking is data transfer. Some transfers must be protected from eavesdropping, others do not need it. Some network traffic is desired, other could be malicious. 

Network security is hard to imagine without firewalls. While they could be annoying sometimes for giving you a hard time while trying to use some application you've just installed on your PC, they protect you from internet wilderness infecting and rotting your precious devices. Firewalls are a tool very much worth befriending. Here comes the `iptables` - ***The** Linux Firewall*.

## Packet phases - chains in `netfilter`
The engine, the essence of iptables lies within the kernel. It's a set of kernel modules built using the `netfilter` framework.  Iptables is just a command-line tool used to add or remove `netfilter` rules.

Netfilter introduces a concept of network packet stages: each network packet coming to the device or being sent out from the device has to pass through several phases. Each phase can apply filters. Some phases can perform some specific packet modifications. All the phases can have multiple rules matching the packet - if that's the case, the first rule in the list is applied and further rules in that phase are ignored (there is an exception of RETURN target).

Below is a chart of all the phases (chains) that network packages have to pass through in Netfilter.
![Packet phases in Netfilter](https://upload.wikimedia.org/wikipedia/commons/thumb/3/37/Netfilter-packet-flow.svg/1920px-Netfilter-packet-flow.svg.png)

There are 3 essential paths a packet could take:

- INPUT - a packet received by a computer, meant for some application running in it
- OUTPUT - a packet originated in the computer, meant for some device in the network
- FORWARD - a packet received by a computer meant for another device in the network

All three paths and related chains are depicted in the image above.

All netfilter chains (phases) are listed in the table below. All the chains are ordered in a sequence they are processed in. Tables (filter, nat, etc.) will be explained in the next chapter.

|  filter |     nat     |    mangle   | security |     raw     |
|---------|-------------|-------------|----------|-------------|
|         | PREROUTING  |  PREROUTING |          |  PREROUTING |
|  INPUT  |   INPUT     |    INPUT    |  INPUT   |             |
| FORWARD |             |   FORWARD   | FORWARD  |             |
|  OUTPUT |   OUTPUT    |    OUTPUT   |  OUTPUT  |   OUTPUT    |
|         | POSTROUTING | POSTROUTING |          |             |

## Tables in `iptables`
In the table above I have listed all the netfilter chains: **PREROUTING**, **INPUT**, **FORWARD**, **OUTPUT**, and **POSTROUTING**. These chains are projected across multiple tables, which are the column names in the table. Netfilter has 5 tables hardcoded in kernel module code: *filter*, *nat*, *mangle*, *security* and *raw*. The first two are used the most - one would hardly ever find himself in a situation where the last 3 tables would be needed.
Packets visit all the chains across all the tables as shown in the picture below:

![Chains' processing sequence](https://www.frozentux.net/iptables-tutorial/chunkyhtml/images/tables_traverse.jpg)

- **raw** table is hit at phases before a packet has a chance to be modified by any other phases/chains: PREROUTING, where rules will be applied on a packet immediately when it comes in from the network, and OUTPUT, where rules will be applied on a packet immediately when the application in the device creates it in the kernel-space. It's called *raw* because no other chains have modified it - it's raw, unchanged.
- **mangle** table is accessed each time before moving packet to chains in the *nat* and the *filter* tables. This means *mangle* table will be queried before nat@PREROUTING and filter@INPUT on the incoming packet and before filter@OUTPUT and nat@POSTROUTING on the outgoing network packet.
- **security** table is accessed each time immediately after *filter* table.  filter@INPUT -> security@INPUT -> *application* -> filter@OUTPUT -> security@OUTPUT. If the packet is only to be routed, the path would be: filter@FORWARD -> security@FORWARD. This table is meant for advanced security control, e.g. SELinux MAC.
- **filter** table is queried immediately before passing a package to the application (filter@INPUT) and at the end of OUTPUT chain (application -> raw -> mangle -> nat -> *filter*). The filter table is also accessed when routing (FORWARDing) a package. It's the default iptables' table and it's quite enough to only use this table when only using iptables as a firewall. 
- **nat** table is consulted for new connections at points where address translation might be required: immediately before sending a packet out to the network (nat@POSTROUTING), right before passing a packet over to INPUT chain (nat@PREROUTING), immediately after filter@INPUT phase, before passing the packet over to the application. PRE/POSTrouting *nat* rules are applied to *all* the traffic, regardless of whether it's only meant to be forwarded through the computer or it's meant for an application running in that computer. IN/OUTput nat rules are applied only to traffic that's considering an application running inside the computer. It's possible to use all the 4 nat chains for an application running in a computer: IN/OUTput path and PRE/POSTrouting. *nat* table is only queried for new connections, so new rules will not have any effect on already established connections.


## Policies
Policies are default behaviors for each netfilter chain in each table separately. By default, iptables set all policies to permissive: *ACCEPT*. Policies can be easily changed by running a command like this one: `iptables -t filter -P INPUT DROP`.  Replace `filter` with the desired table name, `INPUT` with a chain name and `ACCEPT` with a default value. These default values are applied to a network packet if no other rules in that table@chain match it.

## Targets
Each netfilter table contains chains. Each chain in a table can have a collection of rules. All rules have 2 basic parts: a matcher and a target. Matcher defines a list of criteria that should match a network packet: source IP address, destination port, protocol, MAC address, connection state, etc. Target is an action that should be taken against the matchet packet. Consider the following rule for example:

 `-A INPUT -m tcp -p tcp -s 10.77.0.0/24 -m conntrack -ctstate ESTABLISHED,RELATED -j ACCEPT`
 
Everything until the `-j` part is a matcher, that says: all TCP traffic from network 10.77.0.0/24 through connections that are already established (e.g. initiated by the device this rule is configured on). The `-j ACCEPT` part is the action that's to be taken. `-j` means `--jump` and **ACCEPT** is a *target* accepting that packet. 

There are plenty of predefined targets, some of them are only available is specific chain:table combinations. Please consult `man 8 iptables` or `man 8 iptables-extensions` for more information about targets.  **ACCEPT**, **REJECT**, and **DROP** are the most used targets. If you are configuring a NAT or a router you might also be interested in **SNAT**, **DNAT**, and **MASQUERADE** targets. I'll expand more on that in another post about iptables.

Nftables' rules are created for each chain:table combination separately. The order of rules is also important. When kernel traverses all the rules in the table@chain it attempts to match them in the same order for each network packet. The first rule that matches the packet is applied and no further rules in that table@chain list are checked. If the applied target is terminating, e.g. **REJECT** or **DROP**, the packet's journey ends at that point, It will be either dropped with no further actions taken or rejected with some error sent back to the sender. If, however, the applied target is non-terminating, that packet will move on to another table@phase in the netfilter traversal sequence. It will keep moving on until either a terminating target is applied, packet successfully leaves the computer or it's successfully passed on to the application running in that computer.

## Custom-defined chains
While there are plenty of chains predefined already, they might be not enough when building a sophisticated filtering/forwarding mechanism. It's all fine when you only have 5 or 20 rules for each table. But as rulesets grow they become more tedious to maintain. It's a straight path towards human errors right there! 

Now imagine a set of rules in a filter@INPUT table having 20-50 rules with different matchers but identical targets. Now, what if the policy changed, and instead of DROPing those requests we decided to REJECT them? This means updating all the rules one-by-one.

Both the problems above can be easily solved with custom-defined chains. Custom-defined chains can be used as both: chains and targets. This means that multiple `-I INPUT -m tcp -p tcp -s ....... -j ACCEPT` rules could be classified as "customer 'Happy Tails'" by applying them the same target: `-I INPUT -m tcp -p tcp -s ....... -j CUST_HAPPY_TAILS`. And that chain could have a rule: `-A CUST_HAPPY_TAILS -j ACCEPT`. Once the contract with 'Happy Tails' expires or they don't pay you on time, refusing to provide a service would be a single rule amendment: `iptables -I CUST_HAPPY_TAILS -j REJECT`.

Custom-defined rules can only be created separately for each table. To create a new rule issue a command `iptables -t filter -N CUST_HAPPY_TAILS`

## A future of iptables
Iptables is quite an old project. I still remember the hype it caused when it was released. And the hype wasn't for nothing. It turned out to be a very successful project that significantly changed computer networking and security. There have been quite a few Linux firewall implementations:

- ipfirewall (BSD), Linux 1.1
- ipchains/netfilter, Linux 2.2
- iptables/netfilter, Linux 2.4
- nft/nftables, Linux 3.13
- bpfilter/ebpf, Linux 3.18

As you see in this list, iptables is quite an old beast and it's already been replaced by nftables, which itself is on its way to becoming legacy. Bpfilter is a very promising project introducing a brand new way to run and control firewalls. However, like with other tools in the Linux world, iptables is not going to die any time soon. It's still being widely used in many Linux distributions, even in new versions. This is likely change in the next 3-5 years though. If bpfilter turns out worth the hype it causes now I believe distributions might even skip nftables and jump immediately to bpfilter. If not -- nftables will probably the next default firewall.

## Sum-up
`Iptables` is a user interface for a kernel space firewall, called `netfilter`. It works by dividing the packet's lifecycle into multiple phases and applying multiple sets of rules during each phase. When a rule matching the packet is found, a predefined action is applied, and, depending on the action, the packet's lifecycle either continues or terminates at that point. It's a very simple model for a firewall, however, having multiple phases (layers) and being able to create new ones gives us a lot of flexibility. Some phases have their specific set of available modifications that allows us to use iptables not only as a firewall but also as a router, logger, and many other things. 

It might take a while to get your head around how iptables works, but it's definitely worth the trouble. In this chapter I have introduced you to iptables end explained a bit on how it works and how it's used. In the next chapter, I'll expand a bit more on what can you do with it. Stay tuned!

# Sources
- [https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)
- [https://www.wikiwand.com/en/Netfilter](https://www.wikiwand.com/en/Netfilter)
- [https://www.wikiwand.com/en/Iptables](https://www.wikiwand.com/en/Iptables)
- [https://linux.die.net/man/8/iptables](https://linux.die.net/man/8/iptables)
- [https://wiki.archlinux.org/index.php/Iptables#Tables](https://wiki.archlinux.org/index.php/Iptables#Tables)
- [https://www.frozentux.net/iptables-tutorial/chunkyhtml/](https://www.frozentux.net/iptables-tutorial/chunkyhtml/)
- 

> Written with [StackEdit](https://stackedit.io/).
