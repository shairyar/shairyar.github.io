---
title: "Kubernetes Learning Path: Understanding Namespaces"
date: 2025-10-20 10:00:00 +0500
categories: [Kubernetes, Tutorial]
tags: [kubernetes, k3d, namespaces, configmap]
series: kubernetes-learning-path
series_order: 3
---

# Kubernetes Learning Path: Understanding Namespaces

In the [previous posts](/posts/kubernetes-learning-path-configmaps-and-secrets/), we've deployed apps and managed configuration. But what happens when you want to run multiple environments on the same cluster? Maybe you're running dev, staging, and prod all together, or different teams need to share the cluster without accidentally deleting each other's stuff.

That's where namespaces come in. They're basically Kubernetes's way of keeping things organized.

## What Are Namespaces?

Think of namespaces like separate folders on your computer. You could put all your files in one big folder, but that gets messy fast. Instead, you organize them into different folders—work stuff here, personal stuff there, projects over there.

Namespaces do the same thing for your Kubernetes resources. They give you:
- **Isolation** - Resources in one namespace don't interfere with resources in another
- **Organization** - Group related resources together (dev, staging, prod)
- **Access Control** - Different teams can have permissions to different namespaces
- **Resource Limits** - You can set quotas per namespace

But here's the thing—namespaces aren't completely isolated. Pods in different namespaces can still talk to each other (unless you set up network policies). Think of namespaces more like organizational boundaries than security walls.

## See Your Current Namespaces

Kubernetes creates a few namespaces by default:

```bash
kubectl get namespaces
```

Output:
```
NAME              STATUS   AGE
default           Active   2d
kube-node-lease   Active   2d
kube-public       Active   2d
kube-system       Active   2d
```

Here's what they're for:
- **default** - Where your resources go if you don't specify a namespace
- **kube-system** - System components like DNS and dashboard live here
- **kube-public** - Public resources readable by everyone (rarely used)
- **kube-node-lease** - Heartbeat information from nodes (you won't need to touch this)

When you've been running `kubectl get pods` in the previous tutorials, you've actually been looking at the `default` namespace this whole time. You just didn't know it.

## Creating Namespaces

Let's create two namespaces for development and production environments:

```bash
kubectl create namespace dev
kubectl create namespace prod
```

Output:
```
namespace/dev created
namespace/prod created
```

Check your namespaces again:

```bash
kubectl get namespaces
```

Output:
```
NAME              STATUS   AGE
default           Active   2d
dev               Active   5s
kube-node-lease   Active   2d
kube-public       Active   2d
kube-system       Active   2d
prod              Active   5s
```

Your new namespaces are ready to use.

## Real Example: Deploy App to Multiple Namespaces

Let's deploy a simple app to both dev and prod namespaces. We'll use different ConfigMaps for each environment to show how namespaces keep things separate.

### Step 1: Create ConfigMaps for Each Environment

First, create a ConfigMap for dev with development-specific settings:

```bash
kubectl create configmap app-config \
  --from-literal=ENVIRONMENT=development \
  --from-literal=DEBUG_MODE=true \
  --from-literal=API_URL=http://dev-api.example.com \
  --namespace=dev
```

Output:
```
configmap/app-config created
```

Now create a different ConfigMap for prod with production settings:

```bash
kubectl create configmap app-config \
  --from-literal=ENVIRONMENT=production \
  --from-literal=DEBUG_MODE=false \
  --from-literal=API_URL=http://api.example.com \
  --namespace=prod
```

Output:
```
configmap/app-config created
```

Notice we used the same name `app-config` in both namespaces. You might think this would cause a conflict, but it doesn't—resources with the same name can exist in different namespaces without any issues. They're completely separate.

View the dev ConfigMap:

```bash
kubectl get configmap app-config -n dev -o yaml
```

Output:
```yaml
apiVersion: v1
data:
  API_URL: http://dev-api.example.com
  DEBUG_MODE: "true"
  ENVIRONMENT: development
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
```

The `-n` flag is shorthand for `--namespace`. You'll use it a lot.

### Step 2: Create the Application Deployment

Create a file called `app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        env:
        # Environment variables pulled from ConfigMap
        # Each env variable references a key from the ConfigMap in the SAME namespace
        - name: ENVIRONMENT              # Name of the environment variable inside the container
          valueFrom:
            configMapKeyRef:
              name: app-config           # ConfigMap name (must exist in the same namespace)
              key: ENVIRONMENT           # Key from the ConfigMap to get the value from
        - name: DEBUG_MODE               # Another environment variable in the container
          valueFrom:
            configMapKeyRef:
              name: app-config           # Same ConfigMap as above
              key: DEBUG_MODE            # Different key from the same ConfigMap
        - name: API_URL                  # Third environment variable
          valueFrom:
            configMapKeyRef:
              name: app-config           # Still the same ConfigMap
              key: API_URL               # Another key from the ConfigMap
```

This is a simple nginx deployment that pulls configuration from a ConfigMap. The cool part? We can use this exact same YAML file in both namespaces, and each deployment will automatically grab its own namespace-specific ConfigMap. Same file, different configs.

Deploy to dev:

```bash
kubectl apply -f app-deployment.yaml --namespace=dev
```

Output:
```
deployment.apps/webapp created
```

Deploy to prod (using the same file):

```bash
kubectl apply -f app-deployment.yaml --namespace=prod
```

Output:
```
deployment.apps/webapp created
```

### Step 3: Verify the Deployments

Check pods in the dev namespace:

```bash
kubectl get pods -n dev
```

Output:
```
NAME                      READY   STATUS    RESTARTS   AGE
webapp-7d8c9f5b4-2x9k7    1/1     Running   0          15s
webapp-7d8c9f5b4-5m2p9    1/1     Running   0          15s
```

Check pods in the prod namespace:

```bash
kubectl get pods -n prod
```

Output:
```
NAME                      READY   STATUS    RESTARTS   AGE
webapp-7d8c9f5b4-6n3q8    1/1     Running   0          10s
webapp-7d8c9f5b4-8p4r2    1/1     Running   0          10s
```

Same deployment name, same pod names—but they're in different namespaces, so there's no conflict.

### Step 4: Verify Different Configurations

Let's check that each deployment is actually using its own ConfigMap. Check the dev environment variables:

```bash
kubectl exec -n dev deployment/webapp -- sh -c 'echo "ENV: $ENVIRONMENT" && echo "DEBUG: $DEBUG_MODE" && echo "API: $API_URL"'
```

Output:
```
ENV: development
DEBUG: true
API: http://dev-api.example.com
```

Now check prod:

```bash
kubectl exec -n prod deployment/webapp -- sh -c 'echo "ENV: $ENVIRONMENT" && echo "DEBUG: $DEBUG_MODE" && echo "API: $API_URL"'
```

Output:
```
ENV: production
DEBUG: false
API: http://api.example.com
```

There we go! Each deployment is using its own ConfigMap values. The dev environment has debug mode enabled and points to a dev API, while prod has debug mode disabled and points to the production API. Same deployment file, totally different behavior.

## Working with Namespaces

### View Resources in a Specific Namespace

```bash
# Get all pods in dev namespace
kubectl get pods -n dev

# Get all resources in prod namespace
kubectl get all -n prod
```

Output:
```
NAME                          READY   STATUS    RESTARTS   AGE
pod/webapp-7d8c9f5b4-6n3q8    1/1     Running   0          5m
pod/webapp-7d8c9f5b4-8p4r2    1/1     Running   0          5m

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/webapp   2/2     2            2           5m
```

### View Resources Across All Namespaces

```bash
kubectl get pods --all-namespaces
```

Or use the shorthand:

```bash
kubectl get pods -A
```

Output:
```
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
dev           webapp-7d8c9f5b4-2x9k7                    1/1     Running   0          10m
dev           webapp-7d8c9f5b4-5m2p9                    1/1     Running   0          10m
prod          webapp-7d8c9f5b4-6n3q8                    1/1     Running   0          8m
prod          webapp-7d8c9f5b4-8p4r2                    1/1     Running   0          8m
kube-system   coredns-59b4f5bbd5-8xk2p                  1/1     Running   0          2d
```

### Set a Default Namespace

Typing `-n dev` or `-n prod` every single time gets annoying real quick. You can set a default namespace for your current context so you don't have to:

```bash
kubectl config set-context --current --namespace=dev
```

Output:
```
Context "k3d-learning" modified.
```

Now all your commands will default to the dev namespace:

```bash
kubectl get pods
```

Output (without needing -n dev):
```
NAME                      READY   STATUS    RESTARTS   AGE
webapp-7d8c9f5b4-2x9k7    1/1     Running   0          15m
webapp-7d8c9f5b4-5m2p9    1/1     Running   0          15m
```

Switch back to default when you're done:

```bash
kubectl config set-context --current --namespace=default
```

### Describe a Namespace

```bash
kubectl describe namespace dev
```

Output:
```
Name:         dev
Labels:       kubernetes.io/metadata.name=dev
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```

This shows you any resource quotas or limits applied to the namespace (none in our case).

## Namespace Communication

Here's something that caught me by surprise—pods in different namespaces can still talk to each other by default. Namespaces don't block network traffic, they just change how DNS works.

Let's say you have a service called `api-service` in the prod namespace. Pods can reach it using:
- `api-service` - Only works from within the same namespace
- `api-service.prod` - Works from any namespace
- `api-service.prod.svc.cluster.local` - Full DNS name (also works from anywhere)

So namespaces aren't security walls—they're more like organizational folders. If you need actual network isolation, you'd have to set up Network Policies, but that's beyond the scope of this post.

## Practical Tips

### Specify Namespace in YAML Files

Instead of using `-n` flags all the time, you can specify the namespace in your YAML files:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev    # <-- Specify namespace here
data:
  ENVIRONMENT: development
```

This is clearer and less error-prone, especially when managing multiple resources.

### Use Namespaces for Environments

A common pattern is to use namespaces for different environments on the same cluster:
- `dev` namespace for development
- `staging` namespace for staging
- `prod` namespace for production

But remember—if you have the resources, separate clusters for production are often better. Namespaces provide logical separation, not physical isolation.

### Don't Overdo It

I've seen people create a namespace for every little thing, and honestly, it just makes life harder. Start simple:
- One or two namespaces for different environments
- Maybe separate namespaces for completely different applications
- Don't create namespaces just because you can

If you find yourself constantly switching between 10+ namespaces, you've probably gone too far. Keep it simple.

## Clean Up

Delete the deployments from both namespaces:

```bash
kubectl delete deployment webapp -n dev
kubectl delete deployment webapp -n prod
```

Output:
```
deployment.apps "webapp" deleted
deployment.apps "webapp" deleted
```

Delete the ConfigMaps:

```bash
kubectl delete configmap app-config -n dev
kubectl delete configmap app-config -n prod
```

Output:
```
configmap "app-config" deleted
configmap "app-config" deleted
```

Delete the namespaces themselves (this also deletes everything inside them):

```bash
kubectl delete namespace dev
kubectl delete namespace prod
```

Output:
```
namespace "dev" deleted
namespace "prod" deleted
```

When you delete a namespace, Kubernetes automatically deletes everything inside it—all pods, deployments, services, everything. Super handy for cleanup, but also scary if you accidentally delete the wrong namespace. Double-check before you hit enter on that delete command.

## What You Learned

✅ What namespaces are and why they're useful  
✅ How to create and manage namespaces  
✅ How to deploy the same application to different namespaces  
✅ How to use namespace-specific ConfigMaps  
✅ How to work with resources across namespaces  

That's pretty much it for namespaces. They're not complicated—just think of them as folders for organizing your cluster. Once you start using them, you'll wonder how you managed without them.

## What's Next?

In **Part 4** of this series, we'll explore **Persistent Storage** in Kubernetes. You'll learn how to:
- Use volumes to persist data beyond pod lifecycles
- Work with Persistent Volumes and Persistent Volume Claims
- Deploy stateful applications like databases

Stay tuned!

---

## Series Navigation

**Part 1:** [Deploy Your First App](/posts/kubernetes-learning-path-deploy-your-first-app/)  
**Part 2:** [ConfigMaps and Secrets](/posts/kubernetes-learning-path-configmaps-and-secrets/)  
**Part 3:** Understanding Namespaces ← You just finished this!  
**Part 4:** Persistent Storage (Coming soon)  
**Part 5:** Ingress and Load Balancing (Coming soon)

---

Found a mistake or have questions? Feel free to [open an issue here](https://github.com/shairyar/shairyar.github.io/issues/new).

