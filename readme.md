# Containerized Homelab Networking

This repo contains some utilities (primarily Dockerfiles) for hosting my networking infrastructure in containers on my kubernetes cluster.


## Background

Traditionally, I have run my homelab as a series of virtual machines attached to different VLANs, and I used a Unifi router to route between them. However, as my infrastructure got more complicated, Unifi got harder to manage. I understand networking already, but it seemed like Unifi was trying to "simplify" things a little too much and actually just made them significantly more complicated. Due to some design constraints set by my family (namely, I can't replace Unifi as the router), the turn-key solutions were also fairly complicated. Due to the number of specific requirements and my lack of experience with systems like PFSense, I decided to raw-dog it and just install a Linux VM.

I set up pi-hole as my DNS & DHCP server since it has a nice GUI, and the fact that it's a fork of dnsmasq means I can dig into config files for more advanced features. At the time, all of my VMs were managed in Proxmox Virtual Environment. A while later, I started learning Kubernetes and fell in love with Calico, which gives me the ability to install firewalls on every individual application and create strict rules governing their access. I built out my cluster in a set of VMs running inside PVE.

\[ Enter Ampere \]

My servers are powerful. I often run fairly heavy workloads on them and need that extra performance. However, I don't want the power draw that comes with it. I decided to switch to arm. Most of my workloads already support it, and I've seen a lot of glowing reviews on Ampere Altra CPUs, so I decided to give it a shot, and... wow... I almost didn't even need a fan. Even under intense load, a cheap crappy 40mm fan did perfectly well, and I wasn't compromising on performance. All that with a fraction of the load of my old servers.

Time to go multi-arch! Thankfully, Kubernetes already supports multi-arch, but Proxmox doesn't. So, what do I do now? Bare-metal deployment! My servers already have IPMI on a closed network piped into my bedroom, so that's good enough for me. If I can get everything moved into Kubernetes, I don't need the features that Proxmox gets me. Not everything can be migrated though. I have a router VM and a VPS VM that I host for a friend.

\[ Enter Kubevirt \]

Kubevirt is a kubernetes operator that allows me to define and run VMs *inside* the cluster. That's flippin' awesome! A `minikube start` and a few `kubectl apply -f ...` commands later, and I have a VM running in Kubernetes on my laptop! Awesome! I spend that weekend getting my friend's VPS moved over. It took me a while to figure out that I had to enable a feature flag because Kubevirt actually doesn't officially support multi-arch yet, but I got it working with a feature flag and a `nodeSelector`.

\[ Enter my own curiosity \]

I really wondered how Kubernetes was doing all this magic to make networking function properly. I know Linux is a router, and I know nft can provide firewall rules, but how is the actual plumbing done? My experience with raw-dogging containers so far has only been mount namespaces and bubblewrap, which literally just disables the network if you unshare the network namespace. How do you actually get networking in there?

I eventually found a [blog post](https://www.gilesthomas.com/2021/03/fun-with-network-namespaces) detailing how you can create a network namespace, create a virtual patch cable, send one end into the container, and set up routing. I dug into man pages and read a lot about how this stuff works, and created a proof of concept with 4 namespaces wired in a daisy-chain, 2 of which acted like routers, and got packets all the way across! I even experimented with nftables, and as it turns out, each network namespace gets its own set of firewall rules! I was able to replicate my routing stack using namespaces!

Now I'm taking it a step further. A container is just a collection of namespaces that form a semi-isolated environment, so lets just shove all that in a container image! That's what this project is. Instead of virtualizing my router and having to worry about multiple architectures, I'll just let cri-o and Kubernetes do the work for me! If I can build a container image capable of doing the inter-VLAN routing in my house, then cri-o will make sure the proper binaries get run no matter the architecture!

All this being said, I'm not sure I would attach this to the internet. When I can eventually get rid of my unifi gateway and free up a slot in my rack for a dedicated firewall, I'll likely end up doing so, but for now my design constraints require at least two routers, so I'm containerizing one.
