---
title: "Kubernetes Learning Path: Setting Up k3s on Raspberry Pi"
date: 2025-10-29 10:00:00 +0500
categories: [Kubernetes, Tutorial]
tags: [kubernetes, k3s, raspberry-pi, deployment]
series: kubernetes-learning-path
series_order: 5
---

# Kubernetes Learning Path: Setting Up k3s on Raspberry Pi

After going through the first four parts of this series with k3d, I felt ready to take things to the next level. k3d is great for learning and local development, but there's something different about running Kubernetes on actual hardware. You deal with real networking, real resource constraints, and real high availability scenarios that you just can't simulate properly on a single machine.

So I decided to set up a 3-node Kubernetes cluster on Raspberry Pi devices using k3s. This would give me a production-like environment at home where I could practice everything I learned with k3d, but on real distributed hardware.

I'll be honest—this took me a couple of hours and involved some head-scratching moments. But once everything clicked into place and I saw all three nodes showing "Ready" status, it felt amazing. Let me walk you through what I did, what went wrong, and what I learned.

## Why Raspberry Pi and k3s?

Before we dive in, you might be wondering why I chose this setup.

I went with **Raspberry Pi** because they're affordable, don't cost much to run 24/7, and honestly it's just fun to build things with them. Plus, dealing with ARM architecture and limited resources teaches you a lot about how Kubernetes manages resources—you can't just throw CPU and RAM at problems like you might on a beefy server.

As for **k3s**, it's basically Kubernetes but lightweight enough to actually run on a Pi without choking it. The best part? It has the same API as full Kubernetes, so everything you learn transfers directly. I wasn't interested in running some toy version of Kubernetes—I wanted the real thing, just smaller.

## What You'll Need

Here's what I used for my setup:

**Hardware:**
- 3× Raspberry Pi 4 (I'd recommend 4GB+ RAM each)
- 3× SSD drives (You can try using an SD card, but I did not want to use it)
- Ethernet cables for all Pis
- A network switch or router with enough ports
- Power supplies for each Pi

**Software:**
- Ubuntu Server 24.04 LTS on all devices
- SSH access to all Pis

I went with Ubuntu Server because I'm already familiar with it. You could probably use Raspberry Pi OS too, but I stuck with what I know.

## The Big Picture

Before we start, here's what we're building:

```
My Network (192.168.18.0/24)
│
├── k3s-master (192.168.18.49)      [Control Plane]
├── k3s-worker-1 (192.168.18.51)    [Worker Node]
└── k3s-worker-2 (192.168.18.52)    [Worker Node]
```

The master node runs the Kubernetes control plane (API server, scheduler, etc.), and the worker nodes run your actual application workloads. All three nodes are connected via ethernet for stability.

## Step 1: Network Setup—The Foundation That Matters

This was the most important step, and I learned it the hard way. If your master node's IP address changes after you've set up the cluster, the whole thing breaks. Worker nodes connect to the master using its IP, and that IP is hardcoded during the join process.

So the very first thing you need to do is set up static IP addresses on all your Raspberry Pis.

### Configure Static IPs with Netplan

On each Raspberry Pi, I configured a static IP using netplan. Here's what I did on the master node:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

I replaced the contents with this configuration:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.18.49/24  # Master node IP
      routes:
        - to: 0.0.0.0/0
          via: 192.168.18.1  # Your router IP
          metric: 100
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

**Important things to note:**
- Replace `192.168.18.49` with your desired static IP
- Replace `192.168.18.1` with your router's IP (usually 192.168.1.1 or 192.168.0.1)
- Use different IPs for each node (.49 for master, .51 for worker-1, .52 for worker-2)

Apply the configuration:

```bash
sudo netplan apply
```

Verify the IP address:

```bash
ip addr show eth0
```

You should see your static IP address listed. If you don't have network access after applying, double-check that your router IP and subnet are correct.

> **Pro tip:** I initially had both WiFi and ethernet active on my Pis, which caused routing conflicts. I disabled WiFi completely and used only ethernet. Much more stable.
{: .prompt-tip }

### Set Up Hostname Resolution

To make life easier, I set up `/etc/hosts` on all nodes so I could use hostnames instead of IP addresses:

```bash
sudo nano /etc/hosts
```

Add these lines on **all three nodes**:

```
192.168.18.49   k3s-master
192.168.18.51   k3s-worker-1
192.168.18.52   k3s-worker-2
```

Now you can ping by hostname:

```bash
ping k3s-master
```

This makes troubleshooting and managing the cluster much easier.

## Step 2: Preparing All Nodes

Before installing k3s, each Raspberry Pi needs some preparation. I had to do these steps on **all three nodes**.

### System Updates

Always start with updates:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl vim git
```

This took about 10-15 minutes per Pi on my internet connection.

### Set Hostnames

On each Pi, set the appropriate hostname:

```bash
# On master node:
sudo hostnamectl set-hostname k3s-master

# On worker-1:
sudo hostnamectl set-hostname k3s-worker-1

# On worker-2:
sudo hostnamectl set-hostname k3s-worker-2
```

Verify:

```bash
hostname
```

### Enable Legacy iptables

This step confused me at first. Modern Ubuntu uses something called nftables by default, but k3s needs the older iptables-legacy for its networking to work. I didn't really understand why until I forgot to do it on one node and spent an hour debugging why pods couldn't talk to each other.

Run this on **all nodes**:

```bash
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

Lesson learned: don't skip this step.

### Enable cgroup Memory

Kubernetes needs cgroup support for resource management (CPU and memory limits). This requires editing the boot parameters:

```bash
sudo nano /boot/firmware/cmdline.txt
```

You'll see a long line of parameters. Add these to the **end of the existing line** (don't create a new line):

```
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

The entire line might look something like this:

```
console=serial0,115200 console=tty1 root=PARTUUID=12345678-02 rootfstype=ext4 elevator=deadline rootwait cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

> **Warning:** These parameters must be on the same line as the existing boot parameters. If you put them on a new line, they won't work.
{: .prompt-warning }

Save the file and reboot:

```bash
sudo reboot
```

After the reboot, verify cgroup memory is enabled:

```bash
cat /proc/cgroups | grep memory
```

You should see output showing memory cgroup support.

## Step 3: Installing k3s on the Master Node

Okay, now for the fun part—actually installing k3s. After all that prep work, the installation itself is almost anticlimactic. On the master node:

```bash
curl -sfL https://get.k3s.io | sh -
```

That's it. The script downloads and installs k3s automatically. It took about 2-3 minutes on my Pi.

### Verify the Master Node

Check that k3s is running:

```bash
sudo systemctl status k3s
```

You should see output showing k3s is "active (running)".

Check the node status:

```bash
sudo k3s kubectl get nodes
```

Output:

```
NAME         STATUS   ROLES                  AGE   VERSION
k3s-master   Ready    control-plane,master   30s   v1.27.7+k3s1
```

Seeing "Ready" status means the master node is operational.

### Get the Join Token

Worker nodes need a token to join the cluster. Get it from the master:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

You'll see a long token like:

```
K107f8a9b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0z::server:a1b2c3d4e5f6g7h8i9j0
```

Copy this token—you'll need it in the next step. I saved it to a text file for easy reference.

### Set Up kubectl Access

By default, you need `sudo` to run kubectl commands. Let's fix that:

```bash
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
```

Now you can run kubectl without sudo:

```bash
kubectl get nodes
```

Much better.

### Access the Cluster from Your Laptop

Honestly, SSHing into the master node every time you want to run a kubectl command gets old fast. Let's set it up so you can manage the cluster directly from your laptop.

First, make sure you have kubectl installed on your local machine. If you don't:

```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Now, copy the kubeconfig from the master node to your laptop. On your **local machine**, run:

```bash
# Create .kube directory if it doesn't exist
mkdir -p ~/.kube

# Copy the config from master node
scp user@192.168.18.49:~/.kube/config ~/.kube/k3s-config
```

Replace `user` with your actual username on the Pi, and adjust the IP if needed.

Here's the important part: the config file references `127.0.0.1` (localhost), which works when you're on the master node but not from your laptop. You need to change it to the master node's actual IP.

Edit the file:

```bash
nano ~/.kube/k3s-config
```

Find this line:

```yaml
server: https://127.0.0.1:6443
```

Change it to:

```yaml
server: https://192.168.18.49:6443
```

Save and exit. Now tell kubectl to use this config:

```bash
export KUBECONFIG=~/.kube/k3s-config
```

Test it:

```bash
kubectl get nodes
```

Output:

```
NAME         STATUS   ROLES                  AGE   VERSION
k3s-master   Ready    control-plane,master   10m   v1.27.7+k3s1
```

It works! You're now managing the cluster from your laptop without SSH.

> **Pro tip:** If you want this to persist across terminal sessions, add `export KUBECONFIG=~/.kube/k3s-config` to your `~/.bashrc` or `~/.zshrc` file. Or, if you want to merge it with your default kubeconfig, you can copy the content into `~/.kube/config` and use context switching.
{: .prompt-tip }

This makes the whole experience way more convenient. I can be coding on my laptop, deploy to the cluster with `kubectl apply`, and check logs—all without leaving my editor or opening another terminal to SSH.

## Step 4: Joining Worker Nodes

Now let's add the worker nodes to the cluster. On **each worker node**, run this command (replace the token with your actual token from the master):

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.18.49:6443 \
  K3S_TOKEN=K107f8a9b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0z::server:a1b2c3d4e5f6g7h8i9j0 sh -
```

**What this does:**
- `K3S_URL` tells the worker where the master node is
- `K3S_TOKEN` authenticates the worker with the master
- The script installs k3s in agent mode (worker mode)

Installation takes a couple of minutes per worker.

### Verify Worker Nodes

On each worker, check that the k3s agent is running:

```bash
sudo systemctl status k3s-agent
```

You should see "active (running)".

Now go back to the **master node** and check all nodes:

```bash
kubectl get nodes
```

Output:

```
NAME           STATUS   ROLES                  AGE     VERSION
k3s-master     Ready    control-plane,master   5m      v1.27.7+k3s1
k3s-worker-1   Ready    <none>                 2m      v1.27.7+k3s1
k3s-worker-2   Ready    <none>                 1m      v1.27.7+k3s1
```

All three nodes showing "Ready"! This is the moment where all the preparation pays off. You now have a working Kubernetes cluster running on real hardware.

## Step 5: Testing the Cluster

Let's deploy something to make sure everything actually works. I'll deploy a simple nginx application with 3 replicas, which should spread across all the worker nodes.

Create a file called `nginx-test.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

Deploy it:

```bash
kubectl apply -f nginx-test.yaml
```

Output:

```
deployment.apps/nginx-test created
service/nginx-service created
```

Watch the pods come up:

```bash
kubectl get pods -o wide -w
```

Output:

```
NAME                          READY   STATUS    RESTARTS   AGE   NODE
nginx-test-7d4c9d8c9b-4xk2p   1/1     Running   0          10s   k3s-worker-1
nginx-test-7d4c9d8c9b-7m8qz   1/1     Running   0          10s   k3s-worker-2
nginx-test-7d4c9d8c9b-9n2kx   1/1     Running   0          10s   k3s-worker-1
```

Notice how the pods are distributed across the worker nodes automatically! Kubernetes is doing its job of spreading the workload.

### Access the Application

Since we used NodePort, we can access the application through any node's IP on port 30080:

```bash
curl http://192.168.18.49:30080
```

Or from your laptop's browser (if you're on the same network):

```
http://192.168.18.49:30080
```

You should see the nginx welcome page. Try accessing it through the other nodes too:

```bash
curl http://192.168.18.51:30080
curl http://192.168.18.52:30080
```

All three IPs work! NodePort makes the service available on all nodes, regardless of where the pods are actually running.

## Challenges I Ran Into

Let me share the issues I encountered so you can avoid them:

### WiFi and Ethernet Conflicts

Initially, I had both WiFi and ethernet enabled on my Pis. This caused weird routing issues where the cluster would work sometimes and fail other times. The solution was to disable WiFi completely in netplan:

```yaml
network:
  version: 2
  renderer: networkd
  # Wifi disabled - Only eth0
```

### cgroup Parameters on Wrong Line

My first attempt at adding cgroup parameters, I put them on a new line in `cmdline.txt`. This doesn't work—boot parameters must be on a single line. The fix was to add them to the end of the existing line.

### Forgetting iptables on One Node

I accidentally skipped the iptables-legacy configuration on one worker node. Pods on that node couldn't communicate with pods on other nodes. The symptom was that some HTTP requests would work and others would timeout, depending on which pod handled the request. Once I set up iptables-legacy on all nodes and rebooted, everything worked.

## What I Practiced

Now that I have a real cluster, I've been practicing all the concepts from the previous posts:

**From the k3d tutorials:**
- ✅ Multi-replica deployments that actually run on different physical machines
- ✅ ConfigMaps and Secrets for configuration management
- ✅ Namespaces for environment separation
- ✅ Services with NodePort for external access

**New things with real hardware:**
- ✅ **Real high availability:** I powered off one worker node and watched Kubernetes automatically reschedule the pods to the remaining nodes. This is the kind of behavior you can't properly test on k3d.
- ✅ **Resource distribution:** Seeing how Kubernetes spreads pods across nodes based on available resources

The experience of seeing pods automatically move from a failed node to healthy ones was really cool. This is what Kubernetes is designed for.

## Things I Wish I'd Known Before Starting

Looking back, here's what would have saved me time:

**Get the networking sorted out first.** Seriously. Make sure those static IPs are stable before you even think about installing k3s. I can't stress this enough—if your master node's IP changes later, you're basically starting over.

**Skip WiFi, use ethernet.** I tried having both enabled at first and it caused all sorts of weird routing problems. Just use ethernet and call it a day.

**Write everything down.** IP addresses, hostnames, that join token—put it all in a text file. Future you will thank present you when something breaks at 11 PM and you can't remember which node is which.

**Test as you go.** Don't configure all three Pis and then try to figure out what went wrong. Do one, make sure it works, then move to the next.

**Pis are slow, that's okay.** If a pod takes 45 seconds to start, that's just how it is with Raspberry Pi. Don't panic thinking something's broken.


## What You Learned

✅ How to set up static networking on Raspberry Pi  
✅ How to install and configure k3s on multiple nodes  
✅ How to create a multi-node Kubernetes cluster on real hardware  
✅ The importance of proper preparation (iptables, cgroups, networking)  
✅ How to troubleshoot common cluster setup issues  
✅ The difference between local development (k3d) and real clusters  

## Conclusion

Setting up k3s on Raspberry Pi was one of the most rewarding things I've done in my Kubernetes learning journey. Yeah, it took a few hours and I definitely had moments where I questioned my life choices (looking at you, networking issues), but seeing all three nodes report "Ready" made it all worth it.

The best part? Now I have a production-like environment sitting on my desk where I can break things without worrying about cloud bills. Everything I learned with k3d applies here, except now when I mess up, it's on real hardware with actual network cables and blinking LEDs. There's something satisfying about that.

If you've been learning Kubernetes with k3d or minikube, I'd say take the jump to real hardware when you feel ready. The lessons you get from dealing with actual networking problems and watching a node physically fail (or accidentally unplugging one) are different from simulations. Plus, having your own cluster just feels cool.

## What's Next?

In future posts, I'll be documenting what I build on this cluster:

- Setting up Traefik Ingress with custom domains
- Deploying a full Rails application with PostgreSQL
- Configuring persistent storage with NFS
- Implementing monitoring and logging
- CI/CD pipelines targeting the cluster

Stay tuned!

---

## Series Navigation

**Part 1:** [Deploy Your First App](/posts/kubernetes-learning-path-deploy-your-first-app/)  
**Part 2:** [ConfigMaps and Secrets](/posts/kubernetes-learning-path-configmaps-and-secrets/)  
**Part 3:** [Understanding Namespaces](/posts/kubernetes-learning-path-namespaces/)  
**Part 4:** [Understanding Port Mapping in k3d](/posts/kubernetes-learning-path-port-mapping/)  
**Part 5:** Setting Up k3s on Raspberry Pi ← You just finished this!  
**Part 6:** Persistent Storage (Coming soon)

---


