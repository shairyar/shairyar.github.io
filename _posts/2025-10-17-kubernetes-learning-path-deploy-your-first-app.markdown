---
title: "Kubernetes Learning Path: Deploy Your First App"
date: 2025-10-17 16:00:00 +0500
categories: [Kubernetes, Tutorial]
tags: [kubernetes, k3d, deployment, docker]
series: kubernetes-learning-path
series_order: 1
---

# Kubernetes Learning Path: Deploy Your First App

I have been wanting to learn Kubernetes for a while now, and I came across a tool called <a href="https://k3d.io" target="_blank">k3d</a>. It is a great tool that makes it easy to run Kubernetes clusters locally inside Docker containers, making it perfect for learning and development without needing complex infrastructure setup.

In this post, I'll walk you through deploying your first application to Kubernetes using k3d. We'll start from scratch and work our way up to a running nginx server.

## Quick Setup: Install k3d

First, make sure you have Docker running on your machine. Then install k3d:

```bash
# macOS (using Homebrew)
brew install k3d

# Or using curl
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

Verify the installation:

```bash
k3d version
```

## Create Your First Cluster

A cluster is a group of machines (nodes) working together to run your applications—think of it like a team of servers that can share the workload.

Creating a cluster is simple:

```bash
# Create a cluster named "learning"
k3d cluster create learning
```

Output:
```
INFO[0000] Prep: Network                                
INFO[0000] Created network 'k3d-learning'               
INFO[0000] Created image volume k3d-learning-images     
INFO[0000] Starting new tools node...                   
INFO[0001] Creating node 'k3d-learning-server-0'        
INFO[0001] Pulling image 'ghcr.io/k3d-io/k3d-tools:5.6.0'
INFO[0002] Creating LoadBalancer 'k3d-learning-serverlb'
INFO[0003] Cluster 'learning' created successfully!     
INFO[0003] You can now use it like this:                
kubectl cluster-info
```

This takes just a few seconds! Now check that your cluster is running:

```bash
k3d cluster list
```

Output:
```
NAME       SERVERS   AGENTS   LOADBALANCER
learning   1/1       0/0      true
```

Check the cluster nodes:
```bash
kubectl get nodes
```

Output:
```
NAME                     STATUS   ROLES                  AGE   VERSION
k3d-learning-server-0    Ready    control-plane,master   30s   v1.27.4+k3s1
```

You should see one node in "Ready" state. Now you're ready to deploy.

## Deploy Your First App

So you've got a local Kubernetes cluster running with k3d? Great! Let's deploy something to it. We'll use nginx as our first application—it's simple, reliable, and perfect for understanding how Kubernetes deployments work.

I know that in real life you'll be deploying much more complex applications than just a static nginx server, but we all have to start somewhere. Once you understand the basics with nginx, the same patterns apply to any containerized application. We'll work our way up to more complex deployments in future posts.

## What You'll Deploy

We're going to create two things:
- A **Deployment** that runs 2 nginx containers
- A **Service** that makes nginx accessible from your browser

## Step 1: Create the Deployment

Create a file called `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  labels:
    app: nginx  # <--- Label for the Deployment itself (for organizing Deployments)
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx  # <--- Deployment manages pods with this label
  template:
    metadata:
      labels:
        app: nginx  # <--- These labels will be stamped on each pod
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

**What does this do?**
- Creates 2 identical nginx pods (containers running nginx)
- Uses the lightweight `nginx:alpine` image
- Exposes port 80 in each container

**Understanding the Three Label Sections:**

When I first saw three different places with `app: nginx`, I was confused about why we needed so many labels. Here's what each one does:

1. **`metadata.labels`** - Labels for the Deployment resource itself (helps you organize Deployments with commands like `kubectl get deployments -l app=nginx`)
2. **`spec.selector.matchLabels`** - Tells the Deployment which pods it should manage
3. **`template.metadata.labels`** - Labels that get stamped on each pod created by this Deployment

The selector and template labels **must match**, otherwise the Deployment won't know which pods to manage. We'll use these pod labels to connect the Service in Step 2.

Deploy it:

```bash
kubectl apply -f nginx-deployment.yaml

# Watch your pods start
kubectl get pods -w
```

First, you'll see your 2 pods being created:

```
NAME                          READY   STATUS              RESTARTS   AGE
nginx-demo-7d4c9d8c9b-4xk2p   0/1     ContainerCreating   0          2s
nginx-demo-7d4c9d8c9b-7m8qz   0/1     ContainerCreating   0          2s
```

The `0/1` means 0 out of 1 containers are ready yet—they're still starting up.

Then, after a few seconds, both pods will be running:

```
NAME                          READY   STATUS              RESTARTS   AGE
nginx-demo-7d4c9d8c9b-4xk2p   1/1     Running             0          5s
nginx-demo-7d4c9d8c9b-7m8qz   1/1     Running             0          5s
```

Now `1/1` shows both pods are fully ready and running. 

The `-w` flag watches for changes in real-time (similar to `tail -f`), so press `Ctrl+C` to stop watching and return to your command prompt.

## Step 2: Expose with a Service

Your pods are running, but you can't access them yet. So why do we need a Service anyway?

**Why Services are Required**

Think of it like a hotel receptionist. Guests don't need to know which room each staff member is in—they just ask at the front desk, and the receptionist routes them to whoever can help.

In Kubernetes:
- Each pod gets its own IP address that changes when it restarts
- We have 2 nginx pods running, and more could be added or removed
- The Service gives you one stable address and automatically routes traffic to healthy pods
- When pods restart or scale up/down, the Service updates its routing automatically

Without a Service, you'd have to track pod IPs manually every time something changes.

Create `nginx-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx  # <--- This selector matches pods with label "app: nginx"
  ports:
  - port: 80
    targetPort: 80
```

**How the Service finds your Pods**  
See the `selector: app: nginx`? The Service looks for all pods with the label `app: nginx` (the same labels we set in Step 1) and automatically sends traffic to them. When you add more pods with this label, the Service finds them. When you remove pods, the Service stops sending traffic to them. It's all based on matching labels.

Apply it:

```bash
kubectl apply -f nginx-service.yaml

# Check the service
kubectl get service nginx-service
```

You should see:

```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-service   LoadBalancer   10.43.123.456   <pending>     80:32000/TCP   5s
```

The service is now routing traffic to your pods. The `EXTERNAL-IP` shows `<pending>` in k3d—don't worry about it, we'll use port-forwarding to access the app.

## Step 3: Access Your Application

Use port-forwarding to access nginx:

```bash
kubectl port-forward service/nginx-service 8080:80
```

Open your browser and visit `http://localhost:8080`. You should see the nginx welcome page.

```
Welcome to nginx!

If you see this page, the nginx web server is successfully installed and 
working. Further configuration is required.

For online documentation and support please refer to nginx.org.
Commercial support is available at nginx.com.

Thank you for using nginx.
```

Your application is now running on Kubernetes and accessible from your browser.

## Quick Experiments

### Scale Your Application

```bash
# Scale to 5 pods
kubectl scale deployment nginx-demo --replicas=5
```

Output:
```
deployment.apps/nginx-demo scaled
```

Now watch the new pods appear:
```bash
kubectl get pods -w
```

You'll see 3 additional pods being created:
```
NAME                          READY   STATUS              RESTARTS   AGE
nginx-demo-7d4c9d8c9b-4xk2p   1/1     Running             0          5m
nginx-demo-7d4c9d8c9b-7m8qz   1/1     Running             0          5m
nginx-demo-7d4c9d8c9b-9n2kx   0/1     ContainerCreating   0          2s
nginx-demo-7d4c9d8c9b-6p4mw   0/1     ContainerCreating   0          2s
nginx-demo-7d4c9d8c9b-8r5tn   0/1     ContainerCreating   0          2s
```

Scale back to 2:
```bash
kubectl scale deployment nginx-demo --replicas=2
```

Kubernetes will automatically terminate the extra pods.

### Check the Logs

```bash
# Get pod name
kubectl get pods
```

Output:
```
NAME                          READY   STATUS    RESTARTS   AGE
nginx-demo-7d4c9d8c9b-4xk2p   1/1     Running   0          10m
nginx-demo-7d4c9d8c9b-7m8qz   1/1     Running   0          10m
```

Now view logs from one of the pods:
```bash
kubectl logs nginx-demo-7d4c9d8c9b-4xk2p
```

Output (nginx access logs):
```
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Configuration complete; ready for start up
127.0.0.1 - - [17/Oct/2025:10:30:15 +0000] "GET / HTTP/1.1" 200 615 "-" "Mozilla/5.0"
```

These are the nginx startup messages and any HTTP requests. Your logs might be different.

### Execute Commands

```bash
# Check nginx version (replace with your actual pod name)
kubectl exec nginx-demo-7d4c9d8c9b-4xk2p -- nginx -v
```

Output:
```
nginx version: nginx/1.25.3
```

Open a shell inside the pod:
```bash
kubectl exec -it nginx-demo-7d4c9d8c9b-4xk2p -- /bin/sh
```

You'll get an interactive shell prompt:
```
/ # pwd
/
/ # ls
bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
/ # exit
```

You can run any commands inside the container. Type `exit` to leave the shell.

## Clean Up

When you're done experimenting, it's good practice to clean up the resources. This also helps you practice the delete commands:

```bash
kubectl delete -f nginx-service.yaml
kubectl delete -f nginx-deployment.yaml
```

Output:
```
service "nginx-service" deleted
deployment.apps "nginx-demo" deleted
```

Verify everything is gone:
```bash
kubectl get all
```

Output:
```
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   30m
```

Your nginx resources are gone! The only thing remaining is the default `kubernetes` service, which is a system service that's always present.

## What You Learned

✅ How to create a Kubernetes Deployment  
✅ How to expose an app with a Service  
✅ How to scale applications up and down  
✅ How to inspect pods (check logs and execute commands)  

You've successfully deployed your first application to Kubernetes. The same pattern works for any containerized application—just swap the nginx image for your own.

## What's Next?

In **Part 2** of this series, we'll explore **ConfigMaps and Secrets** to manage configuration and sensitive data in your Kubernetes applications. You'll learn how to:
- Store configuration separately from your code
- Manage environment variables
- Handle sensitive information securely

Stay tuned!

---

## Series Navigation

**Part 1:** Deploy Your First App ← You just finished this!  
**Part 2:** ConfigMaps and Secrets (Coming soon)  
**Part 3:** Persistent Storage (Coming soon)  
**Part 4:** Ingress and Load Balancing (Coming soon)

---

Found a mistake or have questions? Feel free to [open an issue here](https://github.com/shairyar/shairyar.github.io/issues/new).

