---
title: "Kubernetes Learning Path: Understanding Port Mapping in k3d"
date: 2025-10-23 10:00:00 +0500
categories: [Kubernetes, Tutorial]
tags: [kubernetes, k3d, port-mapping, nodeport, loadbalancer]
series: kubernetes-learning-path
series_order: 4
---

# Kubernetes Learning Path: Understanding Port Mapping in k3d

In the [previous posts](/posts/kubernetes-learning-path-namespaces/), we've been using `kubectl port-forward` to access our applications. That works fine for testing, but it's manual and you can only forward one service at a time. What if you want to run multiple services and access them all through different ports on your localhost?

When I first started with k3d, I thought I could just create a cluster, deploy some apps, and hit localhost:8080 or localhost:3000 to see them. Nope. Services running inside the cluster aren't automatically accessible from your host machine. You need to set up port mapping when you create the cluster.

Here's where it gets tricky—I spent way too long trying to map multiple services using LoadBalancer, only to get 404 errors. It took me a while to figure out that LoadBalancer in k3d requires Ingress configuration to actually route traffic. Once I discovered NodePort as a simpler alternative for local development, everything clicked.

In this post, I'll show you what I learned about port mapping in k3d, including the LoadBalancer mistake I made and the NodePort solution that actually worked.

## The Problem: Services Are Inside the Cluster

Your k3d cluster runs inside Docker containers. When you create a service, it gets an IP address inside the Docker network, not on your host machine. So when you try to access localhost:8080, nothing happens—there's no service listening there.

Think of your cluster like a virtual network inside your computer. To access services in that network, you need a way to route traffic from your localhost into the cluster—that's what port mapping does.

## Understanding Port Mapping

Port mapping tells k3d: "when traffic hits localhost:8080 on my machine, forward it to a specific port inside the cluster." You set this up when you create the cluster.

The basic syntax looks like this:

```bash
--port "8080:30080@server:0"
```

This breaks down to:
- `8080` - Port on your localhost
- `30080` - Port inside the cluster
- `@server:0` - Target the first server node

But here's where it gets confusing—what port do you map to? LoadBalancer port 80? A NodePort? I made the mistake of using LoadBalancer, and that didn't work the way I expected.

## My First Attempt: Using LoadBalancer

I thought I could create a cluster with multiple LoadBalancer port mappings, like this:

```bash
k3d cluster create learning \
  --port "8080:80@loadbalancer" \
  --port "3000:80@loadbalancer"
```

Output:
```
INFO[0000] Prep: Network                                
INFO[0000] Created network 'k3d-learning'               
INFO[0000] Created image volume k3d-learning-images     
INFO[0001] Creating node 'k3d-learning-server-0'        
INFO[0002] Creating LoadBalancer 'k3d-learning-serverlb'
INFO[0003] Cluster 'learning' created successfully!
```

Then I deployed two different nginx applications with LoadBalancer services, making sure to give them custom content so I could tell them apart. Let me show you exactly what I did so you can see where it goes wrong.

First, create ConfigMaps with different content for each app:

```bash
kubectl create configmap app1-html --from-literal=index.html='<html><body><h1>APP 1</h1><p>This should only appear on port 8080</p></body></html>'
kubectl create configmap app2-html --from-literal=index.html='<html><body><h1>APP 2</h1><p>This should only appear on port 3000</p></body></html>'
```

Output:
```
configmap/app1-html created
configmap/app2-html created
```

Create the first deployment and service. Create a file called `app1-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html                # References the volume defined below
          mountPath: /usr/share/nginx/html  # Where nginx looks for HTML files
      volumes:
      - name: html                  # Volume name referenced above
        configMap:
          name: app1-html           # References the ConfigMap we created
---
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  type: LoadBalancer              # Using LoadBalancer type
  selector:
    app: app1                     # Routes traffic to pods with label app=app1
  ports:
  - port: 80                      # Port the service listens on inside the cluster
    targetPort: 80                # Port on the pod containers
```

Apply it:

```bash
kubectl apply -f app1-deployment.yaml
```

Output:
```
deployment.apps/app1 created
service/app1-service created
```

> **How ConfigMap keys become files:** When you mount a ConfigMap as a volume, Kubernetes automatically converts each key-value pair into a file. The ConfigMap key (`index.html`) becomes the filename, and the value (the HTML content) becomes the file contents. So when we mount this ConfigMap at `/usr/share/nginx/html`, Kubernetes will create `/usr/share/nginx/html/index.html` inside the container with our HTML content. This is why nginx can find and serve the file—it's automatically created in the directory where nginx looks for HTML files.
{: .prompt-tip }

Create the second deployment and service. Create `app2-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html                # References the volume defined below
          mountPath: /usr/share/nginx/html  # Where nginx looks for HTML files
      volumes:
      - name: html                  # Volume name referenced above
        configMap:
          name: app2-html           # References app2's ConfigMap
---
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  type: LoadBalancer              # Using LoadBalancer type
  selector:
    app: app2                     # Routes traffic to pods with label app=app2
  ports:
  - port: 80                      # Port the service listens on inside the cluster
    targetPort: 80                # Port on the pod containers
```

Apply it:

```bash
kubectl apply -f app2-deployment.yaml
```

Output:
```
deployment.apps/app2 created
service/app2-service created
```

Check your services:

```bash
kubectl get services
```

Output:
```
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
app1-service   LoadBalancer   10.43.123.45    <pending>     80:31234/TCP   1m
app2-service   LoadBalancer   10.43.234.56    <pending>     80:31567/TCP   30s
kubernetes     ClusterIP      10.43.0.1       <none>        443/TCP        5m
```

Both services are created! I opened my browser and tried:
- `http://localhost:8080` - Shows "404 page not found"
- `http://localhost:3000` - Shows... "404 page not found"

Wait, what? I'm getting 404 errors from both ports. The services are running (I can see them in `kubectl get services`), but I can't access them. What's going on?

## What I Figured Out About LoadBalancer Mapping

After some testing and digging, here's what I learned: k3d uses Traefik as its built-in load balancer. When you map ports to `@loadbalancer`, those ports forward traffic to Traefik. But here's the catch—**Traefik needs Ingress resources to know how to route traffic to your services**.

Without Ingress configuration, Traefik has no routing rules and just returns 404.

**"But wait, didn't port-forward work in the previous tutorials?"**

Yes! And that confused me at first too. Here's the key difference:

- **`kubectl port-forward`** creates a direct tunnel from your machine to the pod through the Kubernetes API server. It completely bypasses all cluster networking, load balancers, and ingress controllers. It's purely a development tool that works immediately with zero configuration.

- **LoadBalancer with port mapping** routes traffic through k3d's Traefik load balancer, which expects proper Ingress resources with routing rules (hostnames, paths, etc.) to know which service should handle the traffic.

So to make LoadBalancer work in k3d, you'd need to:
1. Create Ingress resources for each service
2. Configure host-based routing (like `app1.localhost` and `app2.localhost`)
3. Set up DNS or host file entries
4. Possibly configure TLS certificates

For local development, that seemed like way too much work just to access my apps. I needed something simpler, and that's when I discovered NodePort.

(We'll explore Ingress and Traefik in a future post—they're powerful for production setups, but overkill for basic local development.)

## What Worked for Me: Using NodePort

What ended up working was using NodePort instead of LoadBalancer, and mapping my localhost ports directly to specific NodePorts. This way, each localhost port goes straight to a specific service without going through a shared load balancer.

First, let's clean up the old cluster:

```bash
k3d cluster delete learning
```

Output:
```
INFO[0000] Deleting cluster 'learning'                  
INFO[0001] Cluster 'learning' deleted successfully!
```

Now create a new cluster with NodePort mappings:

```bash
k3d cluster create learning \
  --port "8080:30080@server:0" \
  --port "3000:30081@server:0"
```

Output:
```
INFO[0000] Prep: Network                                
INFO[0000] Created network 'k3d-learning'               
INFO[0000] Created image volume k3d-learning-images     
INFO[0001] Creating node 'k3d-learning-server-0'        
INFO[0002] Creating LoadBalancer 'k3d-learning-serverlb'
INFO[0003] Cluster 'learning' created successfully!
```

The port mapping syntax means:
- Traffic to localhost:8080 → forwards to port 30080 inside the cluster
- Traffic to localhost:3000 → forwards to port 30081 inside the cluster
- `@server:0` targets the first server node (the control plane)

NodePort services listen on ports between 30000-32767 by default, so we're using 30080 and 30081.

## Deploy Apps with NodePort Services

Now let's deploy two apps with NodePort services that use specific ports. We'll also customize the content so we can tell them apart.

### Step 1: Create App 1 with Custom Content

First, create a ConfigMap with custom HTML for app1:

```bash
kubectl create configmap app1-html --from-literal=index.html='<html><body><h1>This is App 1 on localhost:8080</h1><p>NodePort: 30080</p></body></html>'
```

Output:
```
configmap/app1-html created
```

Create a file called `app1-nodeport.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html              # References the volume defined below
          mountPath: /usr/share/nginx/html  # Where nginx looks for HTML files
      volumes:
      - name: html                # Volume name referenced above
        configMap:
          name: app1-html         # References the ConfigMap we created
---
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  type: NodePort                  # Using NodePort instead of LoadBalancer
  selector:
    app: app1                     # Routes traffic to pods with label app=app1
  ports:
  - port: 80                      # Port the service listens on inside the cluster
    targetPort: 80                # Port on the pod containers
    nodePort: 30080               # Specific NodePort (must match cluster port mapping)
```

The key parts:
- The ConfigMap contains our custom HTML
- The volume makes the ConfigMap available to the pod
- The volumeMount puts that HTML where nginx expects to find it (`/usr/share/nginx/html`)
- The NodePort is set to 30080, which matches our cluster port mapping

Apply it:

```bash
kubectl apply -f app1-nodeport.yaml
```

Output:
```
deployment.apps/app1 created
service/app1-service created
```

### Step 2: Create App 2 with Different Content

Create a ConfigMap for app2:

```bash
kubectl create configmap app2-html --from-literal=index.html='<html><body><h1>This is App 2 on localhost:3000</h1><p>NodePort: 30081</p></body></html>'
```

Output:
```
configmap/app2-html created
```

Create `app2-nodeport.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html              # References the volume defined below
          mountPath: /usr/share/nginx/html  # Where nginx looks for HTML files
      volumes:
      - name: html                # Volume name referenced above
        configMap:
          name: app2-html         # References app2's ConfigMap
---
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  type: NodePort                  # Using NodePort instead of LoadBalancer
  selector:
    app: app2                     # Routes traffic to pods with label app=app2
  ports:
  - port: 80                      # Port the service listens on inside the cluster
    targetPort: 80                # Port on the pod containers
    nodePort: 30081               # Different NodePort for app2
```

Apply it:

```bash
kubectl apply -f app2-nodeport.yaml
```

Output:
```
deployment.apps/app2 created
service/app2-service created
```

### Step 3: Verify the Deployments

Check that everything is running:

```bash
kubectl get all
```

Output:
```
NAME                        READY   STATUS    RESTARTS   AGE
pod/app1-7d8c9f5b4-x9k2p    1/1     Running   0          1m
pod/app2-6n4m8r3c2-y7p5q    1/1     Running   0          45s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/app1-service   NodePort    10.43.123.45    <none>        80:30080/TCP   1m
service/app2-service   NodePort    10.43.234.56    <none>        80:30081/TCP   45s
service/kubernetes     ClusterIP   10.43.0.1       <none>        443/TCP        5m

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app1   1/1     1            1           1m
deployment.apps/app2   1/1     1            1           45s
```

Notice the PORT(S) column shows `80:30080/TCP` and `80:30081/TCP`—these are the specific NodePorts we configured.

### Step 4: Access Your Applications

Now open your browser and visit:

**http://localhost:8080**

You should see:
```
This is App 1 on localhost:8080
NodePort: 30080
```

**http://localhost:3000**

You should see:
```
This is App 2 on localhost:3000
NodePort: 30081
```

Perfect! Each port shows different content. The traffic flow is:
1. Browser hits localhost:8080
2. k3d forwards to NodePort 30080 on the cluster node
3. NodePort 30080 routes to app1-service
4. Service forwards to an app1 pod
5. Pod serves the custom HTML from the ConfigMap

And the same happens for localhost:3000 → NodePort 30081 → app2-service → app2 pod.

## Why I Prefer This Approach

For my local k3d setup, I found NodePort to be more straightforward:
- Each localhost port maps to a specific NodePort
- Each NodePort routes to a specific service
- No shared load balancer to cause confusion
- Simple and predictable for local development

The trade-off is that you need to plan your port mappings when creating the cluster. If you want to add a third app later, you'd need to recreate the cluster with an additional port mapping. But for local development, I found that to be a reasonable compromise.

## Quick Experiments

### Add a Third Application

Let's add one more app to see how easy it is once you understand the pattern.

First, recreate the cluster with an additional port mapping:

```bash
k3d cluster delete learning
k3d cluster create learning \
  --port "8080:30080@server:0" \
  --port "3000:30081@server:0" \
  --port "8081:30082@server:0"
```

Output:
```
INFO[0000] Deleting cluster 'learning'                  
INFO[0001] Cluster 'learning' deleted successfully!
INFO[0000] Prep: Network                                
INFO[0000] Created network 'k3d-learning'               
INFO[0003] Cluster 'learning' created successfully!
```

Create a ConfigMap for app3:

```bash
kubectl create configmap app3-html --from-literal=index.html='<html><body><h1>This is App 3 on localhost:8081</h1><p>NodePort: 30082</p></body></html>'
```

Output:
```
configmap/app3-html created
```

Create `app3-nodeport.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app3
  template:
    metadata:
      labels:
        app: app3
spec:
  containers:
  - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html
    configMap:
          name: app3-html
---
apiVersion: v1
kind: Service
metadata:
  name: app3-service
spec:
  type: NodePort
  selector:
    app: app3
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30082
```

Apply it:

```bash
kubectl apply -f app3-nodeport.yaml
```

Output:
```
deployment.apps/app3 created
service/app3-service created
```

Redeploy app1 and app2 (since we recreated the cluster):

```bash
kubectl create configmap app1-html --from-literal=index.html='<html><body><h1>This is App 1 on localhost:8080</h1><p>NodePort: 30080</p></body></html>'
kubectl create configmap app2-html --from-literal=index.html='<html><body><h1>This is App 2 on localhost:3000</h1><p>NodePort: 30081</p></body></html>'
kubectl apply -f app1-nodeport.yaml
kubectl apply -f app2-nodeport.yaml
```

Output:
```
configmap/app1-html created
configmap/app2-html created
deployment.apps/app1 created
service/app1-service created
deployment.apps/app2 created
service/app2-service created
```

Now you have three apps accessible at:
- http://localhost:8080 (app1)
- http://localhost:3000 (app2)
- http://localhost:8081 (app3)

### Check Service Details

You can see all the NodePort mappings:

```bash
kubectl get services
```

Output:
```
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
app1-service   NodePort    10.43.123.45    <none>        80:30080/TCP   2m
app2-service   NodePort    10.43.234.56    <none>        80:30081/TCP   2m
app3-service   NodePort    10.43.111.22    <none>        80:30082/TCP   1m
kubernetes     ClusterIP   10.43.0.1       <none>        443/TCP        5m
```

You can see the NodePort for each service in the PORT(S) column—80:30080, 80:30081, and 80:30082.

### Test with curl

Instead of using a browser, you can test with curl:

```bash
curl http://localhost:8080
```

Output:
```
<html><body><h1>This is App 1 on localhost:8080</h1><p>NodePort: 30080</p></body></html>
```

```bash
curl http://localhost:3000
```

Output:
```
<html><body><h1>This is App 2 on localhost:3000</h1><p>NodePort: 30081</p></body></html>
```

```bash
curl http://localhost:8081
```

Output:
```
<html><body><h1>This is App 3 on localhost:8081</h1><p>NodePort: 30082</p></body></html>
```

Perfect! Each port serves different content.

### What Happens If You Skip the Port Mapping?

Let's see what happens if you try to access a NodePort that wasn't mapped when creating the cluster.

Create a service with NodePort 30083 (which we didn't map):

```bash
kubectl create deployment app4 --image=nginx:alpine
kubectl expose deployment app4 --type=NodePort --port=80 --name=app4-service --target-port=80
```

Output:
```
deployment.apps/app4 created
service/app4-service exposed
```

Check what NodePort it got:

```bash
kubectl get service app4-service
```

Output:
```
NAME           TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
app4-service   NodePort   10.43.98.77    <none>        80:31847/TCP   10s
```

It got assigned NodePort 31847 (a random port since we didn't specify one). Now try to access it:

```bash
curl http://localhost:31847
```

Output:
```
curl: (7) Failed to connect to localhost port 31847 after 0 ms: Couldn't connect to server
```

It doesn't work because we didn't map that port when creating the cluster. You can only access NodePorts that you explicitly mapped during cluster creation.

This is why planning your port mappings ahead of time is important with k3d.

## When to Use LoadBalancer vs NodePort

Based on what I've learned so far, here's when I'd use each:

**I use NodePort when:**
- I'm running k3d locally for development
- I want multiple services accessible on different localhost ports
- I want a direct, simple mapping without extra configuration
- I want something that works immediately without Ingress setup

**I'd use LoadBalancer when:**
- I'm ready to set up Ingress resources with proper routing rules (we'll cover this in a future post)
- I'm deploying to a real cloud environment (AWS, GCP, Azure) where LoadBalancer actually provisions external IPs
- I want a single entry point with host-based or path-based routing
- I'm building a production-ready setup with proper domain names

For my local k3d development with multiple services, NodePort with port mapping has been working really well. It's direct, predictable, and doesn't require any additional configuration beyond the cluster creation. But I'm sure there are other ways to handle this that I haven't discovered yet.

## Clean Up

Delete all the deployments and services:

```bash
kubectl delete deployment app1 app2 app3 app4
kubectl delete service app1-service app2-service app3-service app4-service
kubectl delete configmap app1-html app2-html app3-html
```

(Note: If you're cleaning up after the LoadBalancer example earlier, you'll only have app1, app2, and their ConfigMaps.)

Output:
```
deployment.apps "app1" deleted
deployment.apps "app2" deleted
deployment.apps "app3" deleted
deployment.apps "app4" deleted
service "app1-service" deleted
service "app2-service" deleted
service "app3-service" deleted
service "app4-service" deleted
configmap "app1-html" deleted
configmap "app2-html" deleted
configmap "app3-html" deleted
```

Remove the YAML files:

```bash
rm -f app1-deployment.yaml app2-deployment.yaml app1-nodeport.yaml app2-nodeport.yaml app3-nodeport.yaml
```

If you want to keep the cluster for future experiments, leave it running. Otherwise, delete it:

```bash
k3d cluster delete learning
```

Output:
```
INFO[0000] Deleting cluster 'learning'                  
INFO[0001] Cluster 'learning' deleted successfully!
```

## What You Learned

✅ Port mapping in k3d isn't automatic—you set it up when creating the cluster  
✅ LoadBalancer in k3d requires Ingress resources to route traffic (we'll cover this in a future post)  
✅ `kubectl port-forward` is different from LoadBalancer—it creates a direct tunnel bypassing all cluster networking  
✅ NodePort gives you direct port-to-service mappings without needing Ingress configuration  
✅ You need to plan your port mappings ahead of time with k3d  
✅ ConfigMaps can serve custom HTML content from volumes  
✅ Multiple services can run on different localhost ports with the right setup  

What I found is that for local k3d development, NodePort with explicit port mappings gave me the control and predictability I needed. I know exactly which localhost port goes to which service, with no surprises and no extra configuration. Your mileage may vary depending on your setup and requirements.

## What's Next?

In **Part 5** of this series, we'll explore **Persistent Storage** in Kubernetes. You'll learn how to:
- Use volumes to persist data beyond pod lifecycles
- Work with Persistent Volumes and Persistent Volume Claims
- Deploy stateful applications that need to keep data

Stay tuned!

---

## Series Navigation

**Part 1:** [Deploy Your First App](/posts/kubernetes-learning-path-deploy-your-first-app/)  
**Part 2:** [ConfigMaps and Secrets](/posts/kubernetes-learning-path-configmaps-and-secrets/)  
**Part 3:** [Understanding Namespaces](/posts/kubernetes-learning-path-namespaces/)  
**Part 4:** Understanding Port Mapping in k3d ← You just finished this!  
**Part 5:** Persistent Storage (Coming soon)

---

Found a mistake or have questions? Feel free to [open an issue here](https://github.com/shairyar/shairyar.github.io/issues/new).

