---
title: "Building out my homelab: My recent interests and findings"
date: 2025-03-05T14:30:00-08:00
draft: true
tags: ["gaming", "programming", "career"]
summary: "Building a K3s Cluster on my Homelab"
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
---
# Building My Home Lab: Getting Started with Proxmox and K3s

After weeks of research and messing around with an old Dell Thinkpad, I finally took the plunge and set up my own homelab. I purchased a mini PC - specifically the [Beelink Mini PC with Intel N5105](https://www.amazon.com/Beelink-Processor-Quad-Core-Computer-Ethernet/dp/B0BHZ117V6), which has been an excellent foundation for my experiments. 

This post documents my journey setting up Proxmox VE as my virtualization platform and deploying a K3s Kubernetes cluster on top of it.
It's truly remarkable what you can do self-hosting and the capabilities of it. 

Before diving into the setup process, here's what I needed to get started:

- **Hardware**: 
  - Beelink Mini PC with Intel N5105 processor
  - 16GB RAM and 512GB SSD
  - USB drive (8GB+) for Proxmox installation
  
- **Software**:
  - Proxmox VE ISO
  - Ubuntu Server 22.04 LTS (for VMs)
  - K3s (lightweight Kubernetes)
  
## What was required


## The Beginning
- Learning Lua for server scripts/ neovim.
- Troubleshooting server issues
- Building custom developer pipelines
- **What's needed?**
  - Basic networking knowledge

## Setting Up Proxmox VE

Proxmox Virtual Environment (VE) is an open-source virtualization platform that combines KVM (Kernel-based Virtual Machine) hypervisor and LXC (Linux Containers) with a web-based management interface.



After installation of Proxmox. You want to update the system:
   - In the shell, ran:
     ```bash
     apt update && apt upgrade -y
     ```
### Step 2: Setting up Ubuntu Server on my VMs

Here's what I did to get Ubuntu running on each VM:

1. First, I fired up the VM in Proxmox and clicked on the console
2. Selected "Try or Install Ubuntu Server" from the boot menu
3. Went through the basic Ubuntu setup:
   - Picked English (because that's all I know anyway)
   - Stuck with DHCP for now to keep things simple
   - Let Ubuntu handle the storage setup automatically
   - Created my user account with a password I'd actually remember
   - Made sure to check the OpenSSH server option (trust me, you want this)
4. After it installed, I rebooted and logged in
5. Updated everything first thing:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
6. Next, I set up static IPs because I hate when my servers play musical chairs with addresses:
   - Master: 192.168.1.101
   - Worker: 192.168.1.102
   
   I had to edit this file: `/etc/netplan/00-installer-config.yaml`
   ```yaml
   network:
     ethernets:
       ens18:
         addresses:
           - 192.168.1.101/24  # Changed this to .102 for the worker
         gateway4: 192.168.1.1
         nameservers:
           addresses: [8.8.8.8, 8.8.4.4]
     version: 2
   ```
   Then I ran `sudo netplan apply` and crossed my fingers

### Step 3: Installing K3s on the Master Node

This part was surprisingly easy:

1. SSH'd into the master VM
2. Ran this magic one-liner:
   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```
3. Grabbed the token I'd need for the worker:
   ```bash
   sudo cat /var/lib/rancher/k3s/server/node-token
   ```
4. Copied that somewhere safe (not a sticky note, I promise)

### Step 4: Adding the Worker Node

Same idea for the worker:

1. SSH'd into the worker VM
2. Another one-liner, but with the master's address and token:
   ```bash
   curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.101:6443 K3S_TOKEN=mysterylongtoken123 sh -
   ```

### Step 5: Making Sure It Actually Worked

On the master node, I ran:
```bash
sudo kubectl get nodes
```

And boom! Both nodes showed up as "Ready" - honestly, I was a bit surprised it worked on the first try.

## Actually Using the K3s Cluster

With the cluster running, I started setting up some useful stuff:

1. Set up kubectl on my laptop so I could control the cluster without SSHing:
   ```bash
   scp myuser@192.168.1.101:/etc/rancher/k3s/k3s.yaml ~/.kube/config
   ```
   Had to edit the config file to replace "localhost" with the actual IP, then tested it with `kubectl get nodes`

2. Added MetalLB for load balancing (because Kubernetes needs something to hand out IPs):
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
   ```
   
   Then I created a config file to give it an IP range:
   ```yaml
   apiVersion: metallb.io/v1beta1
   kind: IPAddressPool
   metadata:
     name: first-pool
     namespace: metallb-system
   spec:
     addresses:
     - 192.168.1.240-192.168.1.250
   ```

3. K3s already comes with Traefik for ingress, which saved me some work

## What's Next

Now that I've got the basic infrastructure working, here's what I'm planning:

- Set up Longhorn for storage (because I don't want to lose all my stuff when a pod restarts)
- Install Gitea so I can host my own Git repos (goodbye GitHub, maybe?)
- Figure out some kind of backup system before I inevitably break something
- Mess around with Lua for my Neovim config (my terminal should look cooler than my actual projects)
- Set up Prometheus and Grafana because I like pretty graphs even if I don't understand what they're telling me

## Things I Learned the Hard Way

- Back. Up. Everything. Before. You. Touch. It.
- Write down your IP addresses and passwords somewhere (not on sticky notes)
- Check if you have enough resources BEFORE you try to deploy something
- Test updates on a snapshot first (learned this after bricking my first setup)

## Final Thoughts

This mini PC homelab thing is pretty awesome. It's been a great learning experience, and it's crazy what you can run on a tiny box that uses less power than a light bulb. 

If you're thinking about setting up something similar, just go for it. Yeah, you'll probably break stuff (I sure did), but figuring out how to fix it is half the fun.

Stay tuned for more posts as I continue to tinker with this setup!

---

*What cool stuff are you running on your homelab? Tell me in the comments so I can steal your ideas!*
