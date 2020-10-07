
# SSH - a tool for IT engineers


SSH is a protocol enabling us to access remote machines. Be it servers, PCs, routers, switches, washing machines, or dashcams. The most popular SSH implementation is OpenSSH - it's completely free of charge and open-source. It's a successor of telnet and RSH suite. While the remote shell is probably the main use for ssh it's by far not the only one. Truth be told other ssh's features often cases are even more useful than remote shell! This post will explain a couple of ssh features that I've found extremely useful as a developer. Without them working on some clients' development projects would have been impossible for me.

## SSH - the connection or what does "SSH" mean

Shell is one of the OS components. It's a text-based user interface allowing users to interact with applications available on the system. SSH enables users to gain shell access to remote devices.

SSH is made up of two parts: a server and a client. The server is to be installed on a machine that's supposed to be accessed remotely. Client applications can be installed on devices that are required to access ssh servers. Ssh client can be initiated to connect to a remote device via TCP protocol using 22/tcp port by default. However, it's quite easy (and is advised for ssh servers facing public internet) to change server's port to anything else.

Once ssh client connects to ssh server they perform a handshake to establish a secure communication channel - that's the first 'S' in 'ssh': "Secure". A version of the Diffie-Hellman algorithm is used to produce a pair of asymmetric keys which is used to exchange a symmetric key. The symmetric key is then used to secure the connection. When the channel is secured authentication information is exchanged. This means that the client verifies whether the server is what the client expects it to be and a server verifies whether the client is authorized to access that system. If everything is correct the server authorizes the connection and grants access to a shell on that device. Usually, the client requests server to execute some shell command - that's what the shell is for. This remote shell is the "SH" part of "SSH".

By default ssh client requests interactive access to the remote shell. That is as if you opened up a terminal on that target machine and were typing commands interactively. However, ssh client can be given a set of commands to execute on a remote machine automatically. This is quite useful in tasks automation and scripting.


## SSH - the SOCKS5 proxy

While the shell is the most commonly used feature of ssh it's not the only one you might find useful in your day-to-day activities. SSH is also very strong in networking! One of it's networking features is SOCKS5 proxy. Simply put an ssh client can set up a remote ssh server to act as your proxy accessible through your localhost.
Suppose you have a remote TEST environment where some application is running. The application is using integration with http://172.14.0.27:8080/api and you need to test that API on your local setup. You cannot access that API from your computer directly as you do not have network routes to that IP address. If you can access your remote environment via ssh you should be able to set up a proxy. Use the `-D <port_number>` parameter when connecting to that environment with an ssh client, e.g. `ssh test.env.application.com -l my_username -D 7788`. This command will start a SOCKS5 Proxy server on your local machine at 127.0.0.1:7788. Set up your browser to use this address as your proxy and try accessing http://172.14.0.27:8080/api now. This time it should work. Now go ahead and test that malfunctioning API!

Why did this work? SSH client started a proxy server on your machine at 7788/tcp. All the traffic coming to that port is routed to the ssh server the client is connected to. SSH server has been instructed to proxy that part of the traffic and so it does -- the ssh server issues the request on your behalf. Since the computer ssh server is running on (your TEST environment's server) knows how to find 172.14.0.27, the request succeeds and a response that's received is returned to ssh client, which returns it to your browser. It's quite a roundabout, it's slower than direct access, but at least it's possible!

Needless to say, API testing is not the only use of the SSH-based SOCKS5 proxy. You can set up your local application to use your proxy to attach to test environment's database, to quickly test your hotfix in test env w/o installing it there or attaching a debugger to an application instance on your local computer while using test environment's external resources (DB, APIs). Or you could use that proxy to, you know, like watch Netflix movies only available in Canada ;)


## Port forwarding

One of my most used features of ssh is port forwarding. It's a feature that enables you to access remote ports locally and local ports remotely. Let me rephrase: using this feature you can access the TEST environment's database from your PC via 127.0.0.1:5433 and you can make an application installed in the test environment to use a database in your local computer. You can also query http://127.14.0.27:8080/api in your local machine to access http://172.14.0.27:8080/api which is normally only accessible from the test environment. SSH's port forwarding is a piece of networking magic when it comes to accessing remote resources!
Now that I got your attention let me start from the beginning. SSH port forwarding comes in two flavours:

- local port forwarding
- remote port forwarding

### Local forwarding
Local port forwarding enables you to pull remote resources to your LOCAL machine. Remote port forwarding, like you, must've correctly guessed, pushes local ports to REMOTE devices. To rephrase: local forwarding makes remote ports accessible locally and remote forwarding makes ports on a local machine (a machine that the ssh client is installed on) accessible by remote devices.

Let's try an example. Suppose you want to access a test environment's database that's only accessible from the test application server. You want to run some SQL queries on that database using your favourite DB client. But you can't since DB port is not accessible from your local machine :sadface: . If you have ssh access to the test application server that is all you need! Just issue this command on your local computer: `ssh test.env.application.com -l my_username -L 127.0.0.1:5433:db01.test:5432` (-L stands for LOCAL port forwarding). Assuming the database's hostname is db01.test and the database is listening on 5432/tcp port (PostgreSQL's default port), this command should be all you need. Now open up your favourite DB client application and create a connection to 127.0.0.1:5433, enter your test DB credentials and you should be all set. Magic, right?
How did that happen? Let's have a look at the ssh command's -L parameter. Its value says this: when connected to SSH server, ask it to make a TCP connection to db01.test:5432, meanwhile I'll create a server-socket on my local machine and bind it to 127.0.0.1:5433; whenever anyone connects to my server-socket I'll ask ssh-server (through the established SSH connection) to open a TCP connection to db01.test:5432 and mediate all the traffic between me (the traffic coming through my server-socket) and the database.

### Remote forwarding
Now suppose you want to make a test application talk to your DEV environment's database for whatever reasons. But test application is in a different network than DEV is, probably even behind some VPN, not to mention it's probably a hardened environment, which oftentimes is the case. It seems like a mission impossible, right? But hey, guess what - as long as you have ssh access very little is impossible! Try this command: `ssh test.env.application.com -l my_username -R 127.0.0.1:5432:db01.dev.env.application:5432`. Now assuming you have full control of the DEV environment, meaning you can access DEV's database directly from your computer, this command should do the trick. Now open up the test application's configuration file, change database address to 127.0.0.1:5432, restart the application and you should be all set! The application should now be talking to DEV's database.

How did that happen? Let's have a look at the ssh command's -R parameter. Its value says this: make a connection to db01.dev.env.application:5432 from this local machine and create a TCP server-socket on the SSH server bound to 127.0.0.1:5432; whenever the server-socket is connected to, forward that connection and all the related traffic to db01.dev.env.application:5432 via the SSH channel.

### Caveats
#### Performance
SSH port forwarding enables you to make connections possible where they would be impossible otherwise. However, this is only about making things possible. Connections established this way will suffer from poor performance. SSH encrypts all the traffic, regardless of whether it's shell access or port forward. So if you are establishing an SSL connection via ssh-forwarded port be ready for slow communications because of dual-encryption. And if you do something like this

- `ssh -l my_username -L 10022:app01.test:22 bastian.env.application.com`
- `ssh localhost -p 10022 -L 1443:api01.test.integration.com:443`
- browser -> https://localhost:1443

you will have triple-encryption: 2x ssh and 1x https-TLS. Such communications are quite heavy on CPU time as well, so they will be significantly slower on a raspberry PI when compared to some Intel i7 processor-powered workstation.

#### Requirements
Port forwarding happens through the SSH connection. This means you need to establish an SSH connection with a remote device if you want any of this to happen. This effectively means that

- the remote device's SSH port must be accessible from the device you are connecting from (networks, firewall rules)
- you must have correct remote machine's credentials
- SSHD (ssh server) must allow port forwarding: **`AllowTCPForwarding yes`**


## SSH OSI-2/3 network interface - ssh-based VPN
Believe it or not, ssh is quite all you need for a two-node VPN connection. A simple iptables rule can teach this VPN to NAT too. How can you make it happen? It's rather simple!
For this VPN to happen a TUN/TAP network interface is required on a device where the SSHD server is running. There are two ways to create it: automatic and manual. The manual approach requires creating a point-to-point network interface manually by whatever means are available on the server. The automatic way requires root ssh access to the system (which is against sshd configuration recommendations!). Despite recommendations, I'll continue assuming we are using the automatic approach as covering all the possible ways an interface can be created on different systems will bloat this post beyond manageable levels. I will assume sshd_config allows root logins: **`PermitRootLogin yes`**.
```
sudo ssh remote.environment.com -l root -w 5:5 'ifconfig tun5 192.168.244.1 pointopoint 192.168.244.2 netmask 255.255.255.0; sysctl net.ipv4.ip_forward=1; iptables -t nat -I POSTROUTING -j MASQUERADE; echo tun5 ready'
sudo ifconfig tun5 192.168.244.2 pointopoint 192.168.244.1 netmask 255.255.255.0
```
These two commands should be enough to create and configure tun5 interface on both sides (server and client) automatically. Since we are creating a system-scope network interface we'll have to run all the commands as root, hence the 'sudo' and '-l root'.

The first command connects to a remote server as root, creates a tun5 TUN (OSI3) device on both the server and the client (`-w 5:5`). The commands run on a server assign an IP address to the server-side tun5 interface, enable IPv4 forwarding and create a firewall rule to NAT outgoing traffic. Should you only require a VPN w/o NAT feel free to omit the sysctl and iptables commands. The second command assigns an IP address to the tun5 interface on a client device. Now the client should be able to ping 192.168.244.1. Even better - the client should be able to access all ports open on a remote device by accessing them on 192.168.244.1. Since we've enabled NAT'ing, the client should be able to access all the network the remote device can reach! To rephrase: if we connected to test.env.application.com, like in previous examples, we now should be able to access the test database directly from our local computer by it's IP:port. http://172.14.0.27:8080/api should work too!

Since pointopoint connections only allow connection between two devices (you cannot reuse the same tun5 for another ssh TUN/TAP connection), it's quite a restrictive VPN. Each ssh TUN/TAP connection will have to create and configure it's own individual tunX device on the server. However, there's still a way to have a true VPN by bridging all the tunX interfaces. See https://la11111.wordpress.com/2012/09/24/layer-2-vpns-using-ssh/ for more on this.

**NOTE**: for NAT to work properly some configuration is required at the client-side as well. When we're querying 172.14.0.27 the OS must know that it should use tun5 interface to reach that IP address rather than the system's default. This is a piece from the Networking course, called Routing. All devices in the network have a routing table. When an application issues a network connection request to some IP address, OS uses this routing table to find out which network interface knows how to find that IP. We need to tell our client's OS that the 172.14.0.27 address is accessible via tun5 interface - we need to add a route to the routing table. In Linux this can be done by running a command: `sudo route add -host 172.14.0.27 dev tun5`. And that should be all.

For this feature to work sshd_config must contain **`PermitTunnel yes`**


## X11 forwarding
X11 is a Linux/Unix graphics engine (actually a bit more than that). It renders graphical components of applications, intercepts I/O peripherals. One beautiful thing about X is that it is not bound to your monitor. Even better - using tools like `xpra` graphical applications can be run in headless mode and attached multiple clients - multiple displays, even over the network. It's one flexible, although old and on its long way to deprecation, graphics engine! SSH can run graphical applications on a remote server, ask X to render their graphics, download those rendered graphics, and use a local X engine to display them. Simply put, you can launch `./idea.sh` on a remote server and have the graphical part of IntelliJ IDEA on your desktop's screen with all inputs and outputs working properly.

Unless you're a Linux desktop user you'll rarely ever need this feature. But there are cases when you'll need it in servers. One such situation is setting up Oracle DB: `dbca`, `netca`. Run a command `ssh -l my_username db.test.application.com -X` and you should be all set. In an interactive remote shell issue `echo ${DISPLAY}` command. It should output something similar to 'localhost:10.0'. If you see this or similar output you can run graphical applications on a remote server and see their GUIs on your desktop.

There are a few preconditions that must be met for this feature to work.

- X must be installed on the remote system
- X forwarding must be enabled in sshd_config: **`X11Forwarding yes`**
- a client system must support X. This should not be a problem in Linux desktops, but Windows will require a 3rd-party X server, like `Xming`


## Master socket
This feature is especially useful when scripting pipelines or some automation tools. Using master sockets you can connect to a server once using your credentials to create the master socket, and then reuse that master socket for subsequent connections and skip the 'enter credentials' step. An example command would be `ssh -l my_username app.test.application.com -oControlPersist=30m -oControlMaster=auto -oControlPath="${HOME}/.ssh/master_%r@%h:%p"`. Once authenticated subsequent runs of this command will not require to authenticate for 30 minutes (the ControlPersist=30m option). The master socket file is created as a /home/my_username/.ssh/master_my_username@app.test.application.com:22 file. Not only this feature saves you from typing your credentials each time you connect, but it'll also save some time -- connections using the master socket are a bit faster. The best part is these sockets can be reused for the whole OpenSSH suite: ssh, scp, sftp. Utilizing the master socket feature could participate in speeding-up your CI/CD pipelines and securing them a tad more by reducing the number of steps using your credentials (be it a password or a private key).

# Sum-up
SSH is often believed to be a remote-shell tool for OPS. However, it's a lot more than just a remote shell. One can think of SSH as a swiss-army-knife for most IT engineers. When it comes to networking and making things possible, ssh is the tool. It can be used to set up a proxy server in a few seconds, to make some "hidden" ports accessible or to quickly spin up a VPN to another environment. These features can be combined, used on top of each other to build some sophisticated network pipelines.


# Sources
- `man 1 ssh`
- https://debian-administration.org/article/539/Setting_up_a_Layer_3_tunneling_VPN_with_using_OpenSSH
- https://wikileaks.org/ciav7p1/cms/page_16384684.html
- https://wiki.archlinux.org/index.php/VPN_over_SSH
- https://kb.iu.edu/d/bdnt
- https://la11111.wordpress.com/2012/09/24/layer-2-vpns-using-ssh/`


> Written with [StackEdit](https://stackedit.io/).
