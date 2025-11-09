---
title: "Remote Access to Your k3s Homelab with Tailscale"
date: 2025-11-09 10:00:00 +0500
categories: [Kubernetes, Tutorial]
tags: [kubernetes, k3s, tailscale, vpn, networking, homelab, raspberry-pi]
---

# Remote Access to Your k3s Homelab with Tailscale

While working on my k3s homelab cluster, I ran into an interesting challenge: how could I access my cluster when I wasn't home? I wanted to run `kubectl` commands from anywhere, manage my deployments remotely, and check on my applications—but I didn't have a dedicated public IP address from my ISP.

That's when I discovered **Tailscale**.

[Tailscale](https://tailscale.com) is a modern VPN built on WireGuard that creates a secure, peer-to-peer network between your devices. The best part? It just works. No port forwarding, no complex router configuration, no messing with firewall rules. You install it on your devices, authenticate, and suddenly they can all talk to each other securely—no matter where they are.

In this post, I'll show you exactly how I set up Tailscale on my k3s master node to enable remote cluster access. By the end, you'll be able to manage your homelab cluster from anywhere—your office, a coffee shop, or even from another country—as long as you're connected to Tailscale.

## The Problem: No Dedicated IP Address

Most home internet connections use dynamic IP addresses that change periodically. Even if your IP stayed constant, exposing your Kubernetes API server directly to the internet felt scary.

There are solutions like ngrok or Cloudflare Tunnel, but while reading about them, I realized they're meant to expose your applications to the internet. I plan to look into them in the future when I'm ready to expose my deployed applications publicly. For now, I just wanted an easy way to connect to my cluster.

## The Solution: Tailscale VPN

I've never worked with a VPN before, so I was a little nervous about trying it out. But it turned out to be fun, and I'm glad I gave it a shot! Tailscale creates a secure mesh network between your devices. Once connected:

- Your k3s master node gets a Tailscale IP (in the 100.x.x.x range)
- Your laptop also gets a Tailscale IP
- These IPs are stable and don't change (I hope! 😄)
- All traffic between them is encrypted via WireGuard
- No router configuration needed

The beauty is that kubectl doesn't know the difference. It just connects to an IP address—whether that's your local network (192.168.x.x) or Tailscale network (100.x.x.x).

## Prerequisites

Before we start, make sure you have:

- A running k3s cluster (I'm using a Raspberry Pi 4)
- SSH access to your k3s master node
- A Tailscale account (free tier supports up to 20 devices)
- Tailscale installed on your laptop or workstation

If you need help setting up k3s, check out my post on [Setting Up k3s on Raspberry Pi](/posts/kubernetes-learning-path-k3s-on-raspberry-pi/).

## Step 1: Install Tailscale on Your k3s Master Node

First, let's get Tailscale installed on the k3s master node.

SSH into your master node:

```bash
ssh pi@k3s-master
# or use the IP address
ssh pi@192.168.18.49
```

Install Tailscale using the official installation script:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

This script detects your OS and installs the appropriate Tailscale package. On my Raspberry Pi running Ubuntu, it installed via apt.

Verify the installation:

```bash
tailscale version
```

Output:
```
1.56.1
```

## Step 2: Authenticate Tailscale

Now we need to connect this node to your Tailscale network.

Start Tailscale:

```bash
sudo tailscale up
```

This command will output a URL:

```
To authenticate, visit:

    https://login.tailscale.com/a/xxxxxxxxxxxxx
```

Copy that URL and open it in your browser. You'll be asked to:

1. Log in to your Tailscale account
2. Authorize the device to join your network
3. Give the device a name (I used "k3s-master")

Once authorized, you'll see a success message.

Back in your terminal, verify the connection:

```bash
tailscale status
```

Output:
```
100.77.77.77    k3s-master         pi@          linux   -
```

Great! Your k3s master node is now part of your Tailscale network.

## Step 3: Get Your Tailscale IP Address

Your master node now has a Tailscale IP in the 100.x.x.x range. Let's get it:

```bash
tailscale ip
```

Output:
```
100.77.77.77
```

**Write this down!** You'll need it for kubectl configuration later.

You can verify Tailscale is running:

```bash
sudo systemctl status tailscaled
```

**Important note:** The service is named `tailscaled` (with a 'd' at the end), not `tailscale`. This tripped me up at first when I was trying to check the service status.

## Step 4: Enable IP Forwarding

For Tailscale to route traffic properly, we need to enable IP forwarding on the master node.

Check if it's already enabled:

```bash
sysctl net.ipv4.ip_forward
```

If it shows `net.ipv4.ip_forward = 1`, you're good! If it shows `0`, enable it:

```bash
# Enable temporarily (until reboot)
sudo sysctl -w net.ipv4.ip_forward=1

# Make it persistent across reboots
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
```

## Step 5: Configure k3s Certificate to Include Tailscale IP

Here's something I learned the hard way: by default, the k3s API server certificate only includes your local IP addresses and hostnames. When you try to connect via Tailscale IP, you'll get TLS certificate errors.

The solution is to add your Tailscale IP to the k3s certificate's Subject Alternative Names (SANs).

On your k3s master node, check if the k3s config file exists:

```bash
ls /etc/rancher/k3s/config.yaml
```

If the file doesn't exist, create it:

```bash
sudo mkdir -p /etc/rancher/k3s
sudo nano /etc/rancher/k3s/config.yaml
```

Add your Tailscale IP to the `tls-san` list:

```yaml
tls-san:
  - 100.77.77.77  # Replace with YOUR actual Tailscale IP
  - k3s-master    # Optional: add hostname too
```

If you already have a config file with other settings, just add the `tls-san` section. Don't remove your existing configuration.

Restart k3s to regenerate the certificates:

```bash
sudo systemctl restart k3s
```

Wait about a minute for k3s to restart and regenerate certificates:

```bash
# Check if k3s is running
sudo systemctl status k3s

# Wait a bit for certificates to regenerate
sleep 60
```

This step is crucial. Without it, you'll get TLS certificate errors and you'll be tempted to use `kubectl --insecure-skip-tls-verify`, which isn't ideal.

## Step 6: Configure kubectl for Remote Access

Now comes the fun part—configuring kubectl on your laptop to connect via Tailscale.

### Copy the kubeconfig File

The kubeconfig file is located at `/etc/rancher/k3s/k3s.yaml` on the master node, but it's owned by root. We'll copy it to our laptop:

```bash
# On your laptop, run:
ssh pi@k3s-master "sudo cat /etc/rancher/k3s/k3s.yaml" > ~/.kube/k3s-remote-config
```

Replace `pi` with your actual SSH user and `k3s-master` with your master node's hostname or IP.

### Update the Server URL

Open the copied config file:

```bash
nano ~/.kube/k3s-remote-config
```

Find the `server:` line (should be near the top) and change it from:

```yaml
server: https://127.0.0.1:6443
```

To your Tailscale IP:

```yaml
server: https://100.77.77.77:6443  # Replace with YOUR Tailscale IP
```

Save and exit (Ctrl+X, then Y, then Enter).

### Test Remote Access

Make sure Tailscale is running on your laptop:

```bash
tailscale status
```

You should see both your laptop and k3s-master listed.

Now test kubectl access:

```bash
kubectl --kubeconfig=~/.kube/k3s-remote-config get nodes
```

Output:
```
NAME         STATUS   ROLES                  AGE   VERSION
k3s-master   Ready    control-plane,master   30d   v1.28.2+k3s1
k3s-worker1  Ready    <none>                 30d   v1.28.2+k3s1
k3s-worker2  Ready    <none>                 30d   v1.28.2+k3s1
```

Success! You're now accessing your k3s cluster remotely via Tailscale!

## Managing Multiple Contexts

Right now, you probably have two ways to access your cluster:

1. **Local access** (when at home): `https://192.168.18.49:6443`
2. **Remote access** (via Tailscale): `https://100.77.77.77:6443`

Let me show you how to easily switch between them by merging both configs into one kubeconfig file with different contexts:

```bash
# Backup your current config
cp ~/.kube/config ~/.kube/config.backup

# Merge configs
KUBECONFIG=~/.kube/config:~/.kube/k3s-remote-config kubectl config view --flatten > ~/.kube/merged-config

# Replace your config with merged version
mv ~/.kube/merged-config ~/.kube/config
```

Then switch contexts:

```bash
# View available contexts
kubectl config get-contexts

# Switch to local context
kubectl config use-context default

# Switch to remote context
kubectl config use-context k3s-remote
```

## Troubleshooting

Here are some issues that I ran into and how I fixed them.

### Issue 1: "connection refused" Error

**Problem:** kubectl returns "connection refused" when trying to connect via Tailscale IP.

**Diagnosis:**
```bash
# From your laptop, test connectivity
ping 100.77.77.77  # Use your Tailscale IP

# Test if port 6443 is accessible
curl -k https://100.77.77.77:6443
```

**Solutions:**
- Make sure Tailscale is running on both master node and laptop: `tailscale status`
- Verify IP forwarding is enabled on master: `sysctl net.ipv4.ip_forward`
- Check if k3s API server is listening: `sudo netstat -tlnp | grep 6443`

### Issue 2: Tailscale Service Not Found

**Problem:** `sudo systemctl status tailscale` returns "Unit tailscale.service could not be found"

**Solution:** The service is named `tailscaled` (with a 'd'):

```bash
sudo systemctl status tailscaled
sudo systemctl restart tailscaled
```

### Issue 3: Can Access Locally but Not Remotely

**Problem:** kubectl works fine on home network but times out via Tailscale.

**Diagnosis:**
```bash
# Check Tailscale connectivity
ping 100.77.77.77  # Your Tailscale IP

# Check Tailscale routes
tailscale status
```

**Solutions:**
- Verify both devices show as connected in `tailscale status`
- Check if you're actually connected to Tailscale on your laptop
- Try disconnecting and reconnecting: `sudo tailscale down && sudo tailscale up`

## Conclusion

Setting up Tailscale for remote k3s access has been one of my favorite homelab improvements. The setup took less than an hour, and now I can manage my cluster from anywhere without worrying about dynamic IPs, port forwarding, or complex VPN configurations.

The ability to run kubectl commands from my laptop at a coffee shop feels magical. It's like my homelab is right there with me, even though it's sitting on my desk at home.

If you're running a homelab cluster and want secure remote access without the headaches, I highly recommend giving Tailscale a try. The free tier is more than enough to get started, and the peace of mind knowing your cluster isn't exposed directly to the internet is worth it alone.

## What's Next?

This was just Phase 1 of my remote access setup. I'm also exploring:

- **Cloudflare Tunnel:** For exposing web services publicly without a dedicated IP

Stay tuned for more homelab adventures!

---

## Additional Resources

- [Tailscale Documentation](https://tailscale.com/kb/)
- [Tailscale Quick Start](https://tailscale.com/kb/1017/install)
- [k3s Documentation](https://docs.k3s.io/)

---

Found this helpful? Have questions? Feel free to [open an issue here](https://github.com/shairyar/shairyar.github.io/issues/new).

