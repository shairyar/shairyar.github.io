---
title: "Kubernetes Learning Path: Setting Up Ingress Controller for External Access"
date: 2025-11-06 09:00:00 +0500
categories: [Kubernetes, Tutorial]
tags: [kubernetes, k3s, traefik, ingress, nginx-ingress, networking]
series: kubernetes-learning-path
series_order: 7
---

# Kubernetes Learning Path: Setting Up Ingress Controller for External Access

After deploying my Rails application with PostgreSQL and SolidQueue workers in the [previous post](/posts/kubernetes-learning-path-rails-solidqueue-on-k3s/), I had a fully functional application running on my Raspberry Pi k3s cluster. But there was one problem: I could only access it using `kubectl port-forward`.

Port-forwarding is fine for testing and debugging, but it's not a real solution. It requires keeping a terminal open, it only works from the machine running kubectl, and it's definitely not how you'd let other people access your application. I needed a proper way to expose my Rails app to the outside world.

That's where Ingress controllers come in.

## What is an Ingress Controller?

Think of an Ingress controller as the front desk receptionist at a hotel. When visitors arrive, they don't go directly to individual rooms—they check in at the front desk, and the receptionist routes them to the right place based on their reservation.

In Kubernetes:
- Your **Services** are like hotel rooms (internal addresses for your apps)
- The **Ingress controller** is the front desk (accepts external HTTP/HTTPS traffic)
- **Ingress resources** are the routing rules (which requests go to which services)

Without an Ingress controller, your services are only accessible within the cluster or through workarounds like port-forwarding. With an Ingress controller, you can access apps via actual domain names (like `myapp.local` or `api.example.com`), route multiple apps through a single IP address, and handle SSL/TLS termination. You can even do URL-based routing—sending `/api` requests to your API service and `/admin` requests to your admin panel.

## Using Traefik Ingress Controller

When I started looking into Ingress controllers, I discovered that k3s comes with **Traefik** pre-installed. Perfect! One less thing to install.

Traefik is modern, has a beautiful web dashboard, and includes built-in Let's Encrypt support for automatic SSL certificates. Since it's already running in our k3s cluster, we just need to create Ingress resources to tell Traefik how to route traffic.

## Step 1: Verify Traefik is Running

Let's confirm that Traefik is running in your k3s cluster:

```bash
kubectl get pods -n kube-system | grep traefik
```

Output:
```
traefik-7cd4fcff68-9xk2p   1/1     Running   0          3d
```

Check the Traefik service:

```bash
kubectl get svc -n kube-system traefik
```

Output:
```
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
traefik   LoadBalancer   10.43.45.123   192.168.18.49    80:30080/TCP,443:30443/TCP   3d
```

See that `EXTERNAL-IP`? That's the IP address of your k3s master node (in my case, `192.168.18.49`). This is where Traefik is listening for incoming HTTP and HTTPS requests.

Perfect! Traefik is ready to go. Now let's create an Ingress resource to route traffic to our Rails application.

## Step 2: Create an Ingress Resource

An Ingress resource is just a set of rules that tells Traefik where to send incoming requests.

Create a file called `rails-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rails-ingress
  annotations:
    # Tells Traefik to use the "web" entrypoint (HTTP port 80)
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: rails.home  # ← Domain name: requests to "rails.home" will match this rule
    http:
      paths:
      - path: /  # ← Match all paths starting with "/" (everything)
        pathType: Prefix  # ← "/" matches "/", "/tasks", "/admin", etc.
        backend:
          service:
            name: rails-service  # ← Routes to the "rails-service" we created in Part 6
                                 # (from rails-service.yaml in the previous post)
            port:
              number: 80  # ← Connects to port 80 of rails-service
                         # rails-service then forwards to Rails pods on port 3000
```

Apply it:

```bash
kubectl apply -f rails-ingress.yaml
```

Output:
```
ingress.networking.k8s.io/rails-ingress created
```

### Verify the Ingress

```bash
kubectl get ingress rails-ingress
```

Output:
```
NAME            CLASS    HOSTS        ADDRESS         PORTS   AGE
rails-ingress   <none>   rails.home   192.168.18.49   80      30s
```

You should see your host (`rails.home`) and the IP address where it's accessible. Perfect! Now we just need to configure DNS.

## Step 3: Configure DNS Resolution

Right now, your Ingress is configured to route traffic for `rails.home`, but your computer doesn't know what `rails.home` means or where to find it. We need to tell your computer that `rails.home` should point to your k3s cluster IP.

### Option 1: Edit /etc/hosts (Quick and Easy)

The simplest solution is to add an entry to your `/etc/hosts` file. This works immediately and is perfect for local development.

```bash
# Find your k3s cluster IP first
kubectl get svc -n kube-system traefik
# Look for the EXTERNAL-IP column

# Add to /etc/hosts (replace with your actual IP)
echo "192.168.18.49    rails.home" | sudo tee -a /etc/hosts
```

### Option 2: Router DNS Configuration (Network-Wide)

If you want everyone on your network to access `rails.home` without editing their hosts files, you'll need to configure DNS at the router level or set up a local DNS server.

**The challenge:** Many ISP-provided routers don't allow you to customize DNS entries. My router definitely didn't give me this option.

**The workaround:** Set up dnsmasq on your Raspberry Pi as a local DNS server.

#### Install dnsmasq on Raspberry Pi

```bash
# SSH to your k3s master node
ssh pi@192.168.18.49

# Install dnsmasq
sudo apt update
sudo apt install -y dnsmasq

# Edit configuration
sudo nano /etc/dnsmasq.conf
```

Add these lines at the end:

```
# Listen on localhost and your Pi's IP
listen-address=127.0.0.1,192.168.18.49

# DNS entry for Rails app
address=/rails.home/192.168.18.49

# Forward other DNS queries to your router
server=192.168.18.1
```

Save and exit (Ctrl+X, then Y, then Enter).

Restart dnsmasq:

```bash
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

Verify it's working:

```bash
# Test DNS resolution from the Pi itself
nslookup rails.home 127.0.0.1
```

Output:
```
Server:		127.0.0.1
Address:	127.0.0.1#53

Name:	rails.home
Address: 192.168.18.49
```

#### Configure Your Router (If Supported)

Now you need to tell your router to use the Pi as the DNS server:

1. Log into your router admin panel (usually `192.168.1.1` or `192.168.0.1`)
2. Look for DHCP settings
3. Find "Primary DNS Server" or "DNS Server"
4. Change it to your Pi's IP: `192.168.18.49`
5. Save and reboot the router

**Important:** Many ISP routers don't let you change the DHCP DNS server. If yours doesn't, you'll need to either manually configure DNS on each device or just stick with the `/etc/hosts` method.

#### Manually Configure DNS on Individual Devices

If router configuration doesn't work, you can manually configure DNS on your Linux machine:

```bash
# Edit network connection (replace "eth0" with your interface)
sudo nmcli connection modify eth0 ipv4.dns "192.168.18.49"
sudo nmcli connection down eth0 && sudo nmcli connection up eth0
```

## Step 4: Test Your Ingress

### Test DNS Resolution

First, verify that your DNS is resolving correctly:

```bash
ping rails.home
```

Output:
```
PING rails.home (192.168.18.49): 56 data bytes
64 bytes from 192.168.18.49: icmp_seq=0 ttl=64 time=2.3 ms
64 bytes from 192.168.18.49: icmp_seq=1 ttl=64 time=2.1 ms
```

If you see ping responses from your k3s IP address, DNS is working!

If you get "Name or service not known" or "cannot resolve rails.home", your DNS isn't configured correctly. Go back to Step 3.

### Test HTTP Access

Now try accessing your Rails app via the Ingress:

```bash
curl http://rails.home
```

You should see your Rails application's HTML response!

Or open your browser and visit: `http://rails.home`

You should see your Rails app loading in the browser—no port-forwarding required!

### Test the Host Header (Troubleshooting)

If you're getting a 404 error when accessing via domain name but you know the service is running, the Ingress might be working but the host routing isn't matching.

Test with the Host header explicitly:

```bash
curl -H "Host: rails.home" http://192.168.18.49
```

If this works but `http://rails.home` doesn't, the issue is DNS resolution, not your Ingress configuration.

## Step 5: Access the Traefik Dashboard (Bonus)

One of Traefik's best features is its built-in dashboard that shows all your routes, services, and middleware in real-time. Let me show you how to set it up, because I ran into some interesting issues along the way.

### Why Use IngressRoute Instead of Regular Ingress?

When I first tried to access the Traefik dashboard, I created a regular Kubernetes Ingress resource pointing to port 9000. That didn't work. After some debugging and reading the Traefik docs, I learned that:

- Traefik v2 uses **IngressRoute CRDs** (Custom Resource Definitions) for advanced routing
- The dashboard is exposed via Traefik's internal service called `api@internal`
- This internal service requires an IngressRoute, not a standard Kubernetes Ingress

### Create the Traefik Dashboard IngressRoute

Create a file called `traefik-dashboard-ingress.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1  # ← Traefik-specific CRD (not standard k8s Ingress)
kind: IngressRoute  # ← IngressRoute is Traefik's advanced routing resource
metadata:
  name: traefik-dashboard
  namespace: kube-system  # ← Must be in kube-system where Traefik is running
spec:
  entryPoints:
    - web  # ← Uses HTTP entrypoint (port 80)
  routes:
    - match: Host(`traefik.home`)  # ← Match requests to "traefik.home"
      kind: Rule  # ← This is a routing rule
      services:
        - name: api@internal  # ← Special Traefik internal service for dashboard
                              # This is NOT a Kubernetes service - it's Traefik's internal API
          kind: TraefikService  # ← Tells Traefik this is a Traefik-internal service
```

A couple of important things: make sure you use `traefik.io/v1alpha1` (not `traefik.containo.us/v1alpha1`—that's the old API version). Also, I'm using `.home` instead of `.local` for the domain. I learned the hard way that `.local` has issues with mDNS, which I'll explain in the troubleshooting section.

Apply it:

```bash
kubectl apply -f traefik-dashboard-ingress.yaml
```

Output:
```
ingressroute.traefik.io/traefik-dashboard created
```

### Configure DNS

Add to `/etc/hosts`:

```bash
# Add this line (use your master node IP, not 127.0.0.1)
echo "192.168.18.49    traefik.home" | sudo tee -a /etc/hosts
```

**Important:** Use your master node's actual IP address here, not `127.0.0.1`, especially if you're accessing from a different machine.

### Access the Dashboard

```bash
# Test with curl
curl http://traefik.home/dashboard/

# Or open in your browser:
# http://traefik.home/dashboard/
```

You should see the beautiful Traefik dashboard showing all your routes, including the `rails.home` route we created earlier!

### Quick Reference

List all IngressRoutes:
```bash
kubectl get ingressroute -A
```

Check Traefik logs:
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=20
```

Port-forward alternative (if IngressRoute doesn't work):
```bash
kubectl port-forward -n kube-system svc/traefik 9000:9000
# Then access: http://localhost:9000/dashboard/
```

Now you have both your Rails app and the Traefik dashboard accessible through clean hostnames on the same cluster IP!

## Common Issues I Ran Into

### Setting Up Rails Ingress (rails.home)

#### Issue 1: "Site can't be reached" Error

**Problem:** Accessing `rails.home` in the browser shows "This site can't be reached".

**Diagnosis:**
```bash
# Test DNS resolution
ping rails.home
# If this fails with "Name or service not known", it's a DNS issue
```

**Solution:** Your DNS isn't resolving. Double-check your `/etc/hosts` file or dnsmasq configuration. Make sure you didn't typo the hostname or IP address.

#### Issue 2: 404 Not Found

**Problem:** `rails.home` loads, but shows a 404 error.

**Diagnosis:**
```bash
# Check if Ingress is configured correctly
kubectl describe ingress rails-ingress

# Check if service exists and has endpoints
kubectl get endpoints rails-service
```

**Solution:**
- If endpoints are empty, your service selector doesn't match any pods
- If the Ingress rules look wrong, check your YAML for typos in the host or service name
- Make sure `rails-service` actually exists: `kubectl get svc rails-service`

#### Issue 3: Mixed Content Warnings (HTTP vs HTTPS)

**Problem:** Your Rails app loads but some resources fail to load with mixed content warnings.

**Solution:** This happens when Rails thinks it's running on HTTPS but Ingress is serving HTTP. We covered this in the previous post—make sure you have these settings in `rails-configmap.yaml`:

```yaml
RAILS_ASSUME_SSL: "false"
RAILS_FORCE_SSL: "false"
```

We'll cover proper SSL/TLS setup with cert-manager in a future post.

### Setting Up Traefik Dashboard (traefik.home)

#### Issue 1: "Service port not found" Error

**Problem:** You tried to create a regular Ingress for the Traefik dashboard and got "service port not found" errors in the Traefik logs.

**Solution:** This means you tried to use a regular Ingress pointing to port 9000. The Traefik service in k3s doesn't expose port 9000 by default. Use an IngressRoute with `api@internal` service instead (as shown in Step 5).

#### Issue 2: API Version Errors

**Problem:** You get an error like "no matches for kind IngressRoute in version traefik.containo.us/v1alpha1".

**Solution:** Make sure you're using `traefik.io/v1alpha1` not the old `traefik.containo.us/v1alpha1`. Check the correct API version:

```bash
kubectl get crd ingressroutes.traefik.io -o jsonpath='{.spec.group}' && echo
```

#### Issue 3: The .local Domain Problem

**Problem:** Initially, I used `traefik.local` as the hostname. I added it to `/etc/hosts`, but DNS resolution completely failed. `rails.home` worked fine, but `traefik.local` didn't resolve at all.

**Why this happens:** The `.local` TLD is reserved for mDNS (multicast DNS/Bonjour). On systems with `systemd-resolved` or `avahi`, `.local` domains are intercepted by mDNS before checking `/etc/hosts`.

You can verify this by checking your `/etc/nsswitch.conf`:

```bash
cat /etc/nsswitch.conf | grep hosts
```

If you see `mdns_minimal [NOTFOUND=return]` before `files`, mDNS will intercept `.local` domains.

**Solution:** Use `.home`, `.lan`, or `.internal` TLD instead of `.local`. Much simpler than reconfiguring your system's DNS resolution order.

## Conclusion

Setting up an Ingress controller was a game-changer for my k3s cluster. No more fumbling with `kubectl port-forward` commands or keeping terminals open. Now my Rails app is accessible via a clean hostname (`rails.home`), and I can easily add more applications without worrying about port conflicts or IP addresses.

The `/etc/hosts` method is perfect for getting started quickly on your development machine. For more complex setups where you want network-wide access, dnsmasq on the Raspberry Pi works great—though I'll admit configuring it properly took some trial and error.

What I really appreciate is how Ingress decouples external access from your application. Your Rails pods can be completely internal (ClusterIP service), and the Ingress controller handles all the external routing. When you scale up or redeploy, the Ingress keeps working without any reconfiguration. That's exactly the kind of flexibility Kubernetes is known for.

**Next up:** I'll be exploring persistent storage for file uploads, because right now any files uploaded to my Rails app would be lost if the pod restarts.

## What's Next?

In future posts, I'll be exploring:

- **Persistent Storage for Uploads** - NFS, Longhorn, and ReadWriteMany volumes
- **SSL/TLS with cert-manager** - Automatic Let's Encrypt certificates for HTTPS
- **Mission Control Jobs** - Web UI for monitoring SolidQueue background jobs
- **Database Backups** - Automated PostgreSQL backups with CronJobs
- **Monitoring with Prometheus and Grafana** - Cluster-wide metrics and dashboards

Stay tuned!

---

## Series Navigation

**Part 1:** [Deploy Your First App](/posts/kubernetes-learning-path-deploy-your-first-app/)  
**Part 2:** [ConfigMaps and Secrets](/posts/kubernetes-learning-path-configmaps-and-secrets/)  
**Part 3:** [Understanding Namespaces](/posts/kubernetes-learning-path-namespaces/)  
**Part 4:** [Understanding Port Mapping in k3d](/posts/kubernetes-learning-path-port-mapping/)  
**Part 5:** [Setting Up k3s on Raspberry Pi](/posts/kubernetes-learning-path-k3s-on-raspberry-pi/)  
**Part 6:** [Deploying Rails 8 with SolidQueue on k3s](/posts/kubernetes-learning-path-rails-solidqueue-on-k3s/)  
**Part 7:** Setting Up Ingress Controller ← You just finished this!  
**Part 8:** Persistent Storage (Coming soon)

---

Found a mistake or have questions? Feel free to [open an issue here](https://github.com/shairyar/shairyar.github.io/issues/new).

