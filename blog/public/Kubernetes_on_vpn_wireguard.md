
# Kubernetes on VPN (WireGuard)

## Goal
A multi-node k8s cluster with only 1 port exposed to the internet and that port is UDP.

All nodes could be scattered around the globe, across different service providers. Behind NATs or directly accessible - doesn't matter. 

Only 1 UDP port will be exposed to the public, no matter how wide the cluster. That port will be used to connect to the VPN.

Only nodes in the same VPN can join the cluster and only devices in the same VPN can access the k8s API (and use kubectl).


## Wireguard
Since Wireguard is a native Linux VPN, runs in kernel space and relies on UDP (and I only had a good experience with it!), I'll be setting it up to bring all the nodes together. If anything, it might be the faster and the most lightweight VPN out there! Not to mention the ease of configuration!

### Installation

WireGuard is natively supported since the Linux kernel v5.6, but AFAIK Ubuntu has backported it back to 5.4. I like to just go ahead and install a mainline kernel to both stay up to date and have the newest features available. Don't be surprised if you're running some 4.xx kernel and wg doesn't work. This is what it looks like when I'm saying saying "I told you!"

```
[root@static vpn]# wg-quick up wg0 
[#] IP link add wg0 type wireguard 
Error: Unknown device type. 
Unable to access interface: Protocol not supported 
[#] IP link delete dev wg0 
Cannot find device "wg0"
```

See the [official WireGuard docs](https://www.wireguard.com/install/) for installation instructions.

### VPN configs

I have a tiny script that generates WG configurations for an entire /24 network.

```bash
#!/bin/bash

NL=$'\n';
DATE=$(LANG=C date);

SERVER_TEMPLATE='
## Created at ${DATE}
[Interface]
Address = ${SERVER_IP}/24
PrivateKey = ${SERVER_PRIV}
#PublicKey = ${SERVER_PUB}
ListenPort = ${SERVER_PORT}

'

SERVER_PEER_TEMPLATE='
## Created at ${DATE}
[Peer]
AllowedIPs = ${PEER_IP}/32
PublicKey=${PEER_PUB}

'                        
PEER_TEMPLATE='
## Created at ${DATE}
[Interface]
Address = ${PEER_IP}/24
PrivateKey = ${PEER_PRIV}
#PublicKey = ${PEER_PUB}

[Peer]
Endpoint = ${ENDPOINT}
AllowedIPs = ${PEER_ALLOWED_IPS}/24
PublicKey = ${SERVER_PUB}

PersistentKeepAlive = 25

'

generate() {

    local network=${1:?Network missing. e.g. 10.10.4}
    local workdir=${2:-.}

    local ip=2; 
    local dir="${workdir}/wg/${network}.0";
    echo "Workdir: ${dir}"
    mkdir -p "${dir}/peers"; 
     
    SERVER_PRIV=${SERVER_PRIV:-$(wg genkey)}; 
    SERVER_PUB=$(wg pubkey <<<"${SERVER_PRIV}");
    SERVER_IP="${network}.1"
    SERVER_PORT=51820

    ENDPOINT="example.com:15220"

    local peer; 
    local server

    eval "server=\"${SERVER_TEMPLATE}\""

    while [ "${ip}" != "255" ] ; do 
        PEER_PRIV=$(wg genkey);
        PEER_PUB=$(wg pubkey <<<"${PEER_PRIV}"); 
        PEER_IP="${network}.${ip}"
        PEER_ALLOWED_IPS="${network}.0"
         
        eval "peer=\"${PEER_TEMPLATE}\""
        eval "server=\"${server}${SERVER_PEER_TEMPLATE}\""

        echo "${peer}" >"${dir}/peers/${ip}.conf"; 
        ip=$((ip+1)); 
    done; 
     
    echo "${server}" >"${dir}/wg0.conf"

    echo "Peers saved in: ${dir}/peers"
    echo "Server saved in: ${dir}/wg0.conf"
}

generate "${1}" "${2}"
```

It's very simple and I know I could do better, but that will cut it for now.
Run the script as

```bash
./wggen.bash 10.6.0
```

to generate all the configurations for IPs in the 10.6.0.0/24 network. These configurations will be saved in the `./wg/10.6.0.0/peers/` directory. And the VPN server configuration will be in the file `./wg/10.6.0.0/wg0.conf`. 

### Start the VPN

Now copy the wg0.conf file to the server and put it there: `/etc/wireguard/wg0.conf`. Enable the wireguard's systemd unit by running

```bash
systemd enable wg-quick@wg0
systemd start wg-quick@wg0
```

Now you should have the wg0 interface connected. Run the `wg` command to see the VPN server status.

As for clients - it's pretty much the same. If you used my script to generate all the configs, pick the IP you like from the /peers/ directory and copy the file contents to either of the VPN clients into the `/etc/wireguard/wg0.conf` file. I have a naming convention and I like to name server configuration `wg0.conf` and all the clients -- `wg0c.conf`, but that's just my thing.

Now, assuming iptables doesn't get in your way, you should have several nodes connected into a VPN.

## K8s challenges
I was kind of hoping k8s setup would work out of the box. It didn't. I guess a simple kubeadm setup is fine for a default k8s configuration, but if you want something slightly different - here be dragons.


### Weave CNI failed to start. Poor logging didn't help
My last k8s cluster was running perfectly fine on Weave cni, so naturally, I chose it as my CNI of choice for this experiment. However, I was disappointed. Weave pods were crashing and restarting on remote nodes. Logs only said that the readiness probe failed and it could not connect to (basically) localhost:some_port. I found in the master node that this port is to be exposed by one of the weave processes (it spawns ~5 processes), but that process was nowhere to be seen in remote nodes. I tried polling `ps -ef` hoping to at least see attempts to start it, but I was unsuccessful. 

Unfortunately, weave did not provide me with any more informative logs so I decided no no longer invest my time in weave and go back to the good-old flannel.

### Flannel CNI bound to default NIC
Flannel cni has a special requirement - it needs the k8s cluster to have a preassigned CIDR ([see here](https://github.com/flannel-io/flannel/issues/728#issuecomment-308878912)). IDK how to do that in a live cluster, as I had run `kubeadm reset` a few times already and I saw no reason not to recreate my cluster one again. And so I did. And while I was recreating the cluster, I also had a great opportunity to specify the IP address I wanted the k8s API to be accessible through.

```
kubeadm init --apiserver-advertise-address 10.6.0.1 --pod-network-cidr=10.240.0.0/16
```

Then I ssh'ed to remote nodes and ran the `kubeadm join <...>` command to populate my cluster. Then - applied the flannel YAML and waited eagerly to see all the `kube-system` pods start-up real nice.
and waited
and waited...
...
And it didn't happen.

Looking at flannel logs (THANK YOU FOR THOSE!!!) hinted to me that flanneld is trying to bind to a non-VPN interface. And that's not what I want. I want all the inter-node comms happening via the VPN. No LAN IPs, no WAN IPS - only VPN. So I had to tweak the flannel YAML and specify NICs I want flannel to bind to. 

```
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.13.1-rc2
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=wg0
        - --iface=wg0c
```

Notice the last 2 args ([Flanneld config docs FTR](https://github.com/flannel-io/flannel/blob/master/Documentation/configuration.md)). Since the same flanneld is deployed in all the nodes, the config is the same as well. I haven't found a way to configure flannel differently on different nodes, which I kind of need, because I have this naming convention: 
- wg0 -- the wireguard interface in the VPN server
- wg0c - the wireguard interface in the VPN client (wg0**c**)
If flannel fails to find an interface passed with the first `--iface` arg, it will try the second one, the third and so on, until either there are no more `--iface` params to try or one of them was found.

So I added those 2 params and redeployed flannel. This was quite enough to have Flannel pods finally start up.

### Joined nodes have LAN IPs assigned as INTERNAL_IP
Nodes, once joined, will be automatically assigned their default LAN IP from the routing table. 

```shell
root@netikras-hub:/tmp# kubectl -n kube-system get nodes -o wide                                                                                                                                                                               
NAME           STATUS   ROLES                  AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE          KERNEL-VERSION               CONTAINER-RUNTIME                                                                                  
netikras-hub   Ready    control-plane,master   99m   v1.20.4   192.168.1.xxx   <none>        Linux Mint 20.1   5.4.0-66-generic             docker://19.3.8                                                                                    
netikras-pc    Ready    <none>                 98m   v1.20.4   192.168.1.xxx   <none>        Linux Mint 19.1   5.11.6-051106-generic        docker://20.10.5                                                                                   
srvkag01       Ready    <none>                 95m   v1.20.4   157.9x.xxx.xx   <none>        CentOS Linux 8    5.11.6-1.el8.elrepo.x86_64   docker://20.10.5                                                                                   
root@netikras-hub:/tmp#
```

This means that if k8s wants to send a packet to a node, it will be sending that packet to that node's LAN IP. This is not what I want if I'm running in a VPN. I could block all the incoming LAN traffic (except for VPN) for all I care - the k8s cluster must still work!

I had to assign static IPs to each of the nodes. Master too.
Adding the KUBELET_EXTRA_ARGS=--node-ip=10.6.0.xxx to the kubelet config was the easiest way to achieve that. Yes, that means I'm assigning static VPN IPs to each node. But then again, I do exactly that with WireGuard VPN peer configurations...

```
[root@CentOS-83-64-minimal ~]# cat /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=--node-ip=10.6.0.101
[root@CentOS-83-64-minimal ~]# systemctl restart kubelet
[root@CentOS-83-64-minimal ~]# 

root@netikras-pc:~# cat /etc/default/kubelet
KUBELET_EXTRA_ARGS=--node-ip=10.6.0.110
root@netikras-pc:~# systemctl restart kubelet
root@netikras-pc:~# 

```

Note that the kubelet config might be in different locations in different linuxes.
Soon enough all the nodes got VPN IPs assigned:

```
root@netikras-hub:/tmp# kubectl -n kube-system get nodes -o wide
NAME           STATUS   ROLES                  AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE          KERNEL-VERSION               CONTAINER-RUNTIME
netikras-hub   Ready    control-plane,master   112m   v1.20.4   10.6.0.1      <none>        Linux Mint 20.1   5.4.0-66-generic             docker://19.3.8
netikras-pc    Ready    <none>                 112m   v1.20.4   10.6.0.110    <none>        Linux Mint 19.1   5.11.6-051106-generic        docker://20.10.5
srvkag01       Ready    <none>                 109m   v1.20.4   10.6.0.101    <none>        CentOS Linux 8    5.11.6-1.el8.elrepo.x86_64   docker://20.10.5
root@netikras-hub:/tmp#
```

### Remote nodes still not reachable
And still, we're running into problems when accessing pods running remotely. 

```
root@netikras-hub:/tmp# kubectl exec -it  gitlab-runner-gitlab-runner-75879f55b9-jnrkf -- sh
Error from server: error dialing backend: dial tcp 10.6.0.101:10250: connect: no route to host
```

That's because the iptables setup did not account for the VPN traffic. It was configured to accept traffic on the hardware NICs and drop everything else. So I could either remove the -A INPUT -j DROP rule, which is not recommended for security reasons, or I could insert one more rule to whitelist the VPN traffic

```
[root@CentOS-83-64-minimal ~]# ## iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited
[root@CentOS-83-64-minimal ~]# iptables -I INPUT -s 10.6.0.0/24 -d 10.6.0.0/24 -j ACCEPT
```

### CoreDNS: ECONNREFUSED

Now I can reach remote pods. But there's another problem -- pods can't reach the internet (sort of). Pods cannot resolve DNS names, as the CoreDNS resolver pod is refusing connections.

```
ERROR: Registering runner... failed                 runner=sQXh-Muy status=couldn't execute POST against https://gitlab.com/api/v4/runners: Post https://gitlab.com/api/v4/runners: dial tcp: lookup gitlab.com on 10.96.0.10:53: read udp 10.240.2.3:51496->10.96.0.10:53: read: connection refused
```

My GitLab runner failed to register because it couldn't resolve the **gitlab.com** domain. And that's because 10.240.2.3 attempt to query that DNS in 10.96.0.10:53/UDP was refused. Why is that so? 


That was my mistake. I was following a tutorial when setting up flannel and I assigned the k8s cluster my custom CIDR: 10.240.0.0/16, rather than the one suggested in the tutorial: 10.244.0.0/16. See, there's sometimes a reason tutorials pick particular values. In this case, the latter was the default CIDR flannel uses. So to adapt to my silly misconfiguration, I had to delete flannel from my cluster, update the kube-flannel.yml to reflect my CIDR

```
126   net-conf.json: |
127     {
128       "Network": "10.240.0.0/16",
129       "Backend": {
130         "Type": "vxlan"
131       }
132     }
```

and redeploy flannel again.  And then delete CoreDNS pods and wait for them to come back up.
Interestingly, only 1 out of 2 pods came up. And the fact that the one that failed was an offshore server hinted to me: "gotta be the iptables again". And it was. Removing the FORWARD ban

```bash
iptables -D FORWARD -j REJECT --reject-with icmp-host-prohibited
```

did the trick. I'll have to put it back up again and override with a more precisely matching rule to prevent my server from forwarding attacks.

At this point that was all. k8s nodes were communicating entirely via VPN and all the kube-system and default pods were running without any restarts.

## Gitlab Runner

If you're like me, you probably want to utilize your expensive k8s cluster as much as you can. I wanted to reuse the same cluster for GitLab builds as well as dev env deployment. I used to deploy things manually before configuring gitlab-ci, so I had the repository credentials stored in my local docker setup. [This is how you can transfer docker credentials to k8s](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#registry-secret-existing-credentials):

```bash
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson
```

 And if you're using helm to deploy your GL runner as described [here](https://docs.gitlab.com/runner/install/kubernetes.html#configuring-gitlab-runner-using-the-helm-chart), you'll probably also want to uncomment the below lines in your [values.yaml](https://gitlab.com/gitlab-org/charts/gitlab-runner/blob/master/values.yaml) and set "regcred" as the value for *name*.
 
```
 24 # imagePullSecrets:
 25 #   - name: ""
 26 
```

Assuming you followed the gl-runer helm instructions to the letter and set the gitlabUrl along with the runnerRegistrationToken, you are one 

```
helm install --namespace default gitlab-runner -f values.yaml gitlab/gitlab-runner
```

away from a well-working GitLab runner deployment in your k8s cluster.

P.S. Since I've got the luxury of a big fat Ryzen server, I also set this values.yaml value to 50. I might increase it even more later on.
```
 73 ##
 74 concurrent: 10
 75
```


> Written with [StackEdit](https://stackedit.io/).
