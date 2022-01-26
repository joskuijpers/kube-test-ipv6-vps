# IPv6-only Kubernetes on Bare Metal VPSs (trial and error)

This repository shows a setup a did for Kubernetes:

- An edge server with a public ipv4 and ipv6 address.
- Control plane and workers nodes that only have public ipv6 addresses. (These came cheaper than those with ipv4)
- A private network with all these nodes together.

Below I'll explain the setup in more detail, but first some important notes:

*Note 1*: This setup is not Highly Available! If the edge node goes down there is no access nor Kubernetes API communication.
To solve this, a second edge node can be created and together they can be in front of an elastic/floating ip. I never got to this point.

*Note 2*: My VPS provider did not have attachable volumes, so I wanted to use Ceph to create a replicated storage plane across nodes. I never got to this because I found too many issues. Another idea was to add a non-HA storage node with NFS and/or iSCSI. This would likely cause less IO traffic on the worker nodes but is not HA.

The kubernetes and calico config are perfectly usable though.

# Design

### Network

The edge node allows for both ipv4 and ipv6 ingress. Using haproxy it forwards port 6443 to the master nodes and port 80 and 443 to worker nodes. This was all using the PROXY protocol and TCP forwarding.

The private network is ipv6 only, and the edge node functions as router by running radvd to announce the subnet. It is not used as gateway thought, all egress traffic goes through each node public ipv6 interface.

To make all ipv6-only nodes function properly with Kubernetes, ipv4 egress access is needed (DockerHub requires it): this is why I added a DNS server (BIND9) to the edge node, and force this DNS server with radvd on all nodes. This DNS server uses a NAT64/DNS64 server for forwarding. I found one that works in Finland. I wanted names for each node, so I added a new zone and made each node announce its name and ip using DDNS updates.

### Containers

This setup uses the CRIO container runtime in systemd mode.

### Kubernetes

Kubernetes is set up in ipv6-only mode by supplying all required values in the kubadm configuration. Then Calico was added, also set up with the correct and matching values.
The setup for Calico here is custom and does not have a controller. IP pools are added manually.

### Ingress

Ingress is handled by the official Kubernetes Ingress Controller together with cert-manager. Nginx is configured to accept the PROXY protocol from haproxy, and calculate the correct headers. X-Real-IP now always points to the client that requested the page from haproxy.

### Kuard

A kuard yaml is supplied as an example with an ingress set up. Change the domain to something that works for you. Also switch TLS to 'prod' to make it work in real browsers.


# Notes

Here some other notes about what I did (wrong):

### Cattle

This was a setup I learned a lot from but is not good per se. There is a bunch of single failure points. Also it treats nodes as 'friends' instead of 'cattle' (see naming, DNS).

### SSH Bastion and ProxyJump

I did not have IPv6  access from my office so I needed to do a proxy jump with SSH through a ipv4 and ipv6 capable host. If you also have this issue, this is how you do it in `.ssh/config`:

```
Host my_node
    User ubuntu
    # ipv6 of the node
    HostName x.y.z.x.x.x.x.x
    # node you will jump through
    ProxyJump bastion

Host bastion
    User ubuntu
    # Ipv4 address of bastion node.
    HostName 1.2.3.4
```

The ipv6 node will still use your ssh agent, so there is no need to add an SSH key from the bastion in your node.

# Tips

To reset a cluster:
```
kubeadm reset -f && rm -rf /etc/kubernetes/ && systemctl daemon-reload && systemctl restart kubelet && rm -f /etc/cni/net.d/10-calico.conflist && rm -f /etc/cni/net.d/
```
