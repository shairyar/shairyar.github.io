---
title: "Kubernetes Learning Path: ConfigMaps and Secrets"
date: 2025-10-19 10:00:00 +0500
categories: [Kubernetes, Tutorial]
tags: [kubernetes, k3d, configmap, secrets]
series: kubernetes-learning-path
series_order: 2
---

# Kubernetes Learning Path: ConfigMaps and Secrets

In the [previous post](/posts/kubernetes-learning-path-deploy-your-first-app/), we deployed our first application to Kubernetes. But here's the thing—real applications need configuration. Database URLs, API keys, feature flags, and other settings that change between environments.

You definitely don't want to hardcode these values into your container images. That would mean rebuilding your entire image just to change a database URL or update an API key. That's where ConfigMaps and Secrets come in.

## What Are ConfigMaps and Secrets?

Think of ConfigMaps and Secrets like configuration files that live inside Kubernetes. Instead of baking configuration into your container images, you store it separately and inject it into your pods at runtime.

**ConfigMaps** are for non-sensitive configuration data—things like application settings, feature flags, or database names.

**Secrets** are for sensitive data—like passwords, API keys, or certificates. They're similar to ConfigMaps but have some extra protection (though they're still just base64-encoded, not encrypted by default).

Let's see how to use them with some practical examples.

## Working with ConfigMaps

### Create a ConfigMap from Literal Values

The simplest way to create a ConfigMap is from literal values on the command line:

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info
```

Output:
```
configmap/app-config created
```

Check what you just created:

```bash
kubectl get configmap app-config -o yaml
```

Output:
```yaml
apiVersion: v1
data:
  APP_ENV: production
  LOG_LEVEL: info
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
```

See how your key-value pairs are stored? Now let's create one from a file.

### Create a ConfigMap from a File

First, create a simple config file:

```bash
cat > app.properties << EOF
database.name=myapp
database.pool.size=20
cache.enabled=true
EOF
```

Now create a ConfigMap from this file:

```bash
kubectl create configmap app-config-file --from-file=app.properties
```

Output:
```
configmap/app-config-file created
```

View it:

```bash
kubectl describe configmap app-config-file
```

Output:
```
Name:         app-config-file
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
app.properties:
----
database.name=myapp
database.pool.size=20
cache.enabled=true
```

The entire file content is stored as a single key called `app.properties`.

### Use ConfigMap as Environment Variables

Now let's use our ConfigMap in a pod. Create a file called `pod-with-configmap.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-pod
spec:
  containers:
  - name: demo
    image: alpine:latest
    command: ["sh", "-c", "echo APP_ENV=$APP_ENV && echo LOG_LEVEL=$LOG_LEVEL && sleep 3600"]
    env:
    - name: APP_ENV  # Environment variable name inside the container
      valueFrom:
        configMapKeyRef:
          name: app-config  # References the ConfigMap we created earlier
          key: APP_ENV      # Pulls the value of "APP_ENV" key from that ConfigMap
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config  # Same ConfigMap as above
          key: LOG_LEVEL    # Pulls the value of "LOG_LEVEL" key
```

Apply it:

```bash
kubectl apply -f pod-with-configmap.yaml
```

Output:
```
pod/config-demo-pod created
```

Check the logs to see if the environment variables were injected:

```bash
kubectl logs config-demo-pod
```

Output:
```
APP_ENV=production
LOG_LEVEL=info
```

Great! The values from your ConfigMap are now available as environment variables inside the container.

### Use ConfigMap as a Volume

Sometimes you want configuration files, not just environment variables. You can mount a ConfigMap as a volume. Create `pod-with-config-volume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-volume-pod
spec:
  containers:
  - name: demo
    image: alpine:latest
    command: ["sh", "-c", "cat /config/app.properties && sleep 3600"]
    volumeMounts:
    - name: config-volume    # References the volume defined below
      mountPath: /config     # Where to mount the files inside the container
  volumes:
  - name: config-volume      # Volume name (must match volumeMounts above)
    configMap:
      name: app-config-file  # References the ConfigMap created from app.properties file
```

Apply it:

```bash
kubectl apply -f pod-with-config-volume.yaml
```

Output:
```
pod/config-volume-pod created
```

Check the logs:

```bash
kubectl logs config-volume-pod
```

Output:
```
database.name=myapp
database.pool.size=20
cache.enabled=true
```

The file from your ConfigMap is now mounted inside the container at `/config/app.properties`. This is handy when your app expects actual config files instead of environment variables.

## Working with Secrets

Secrets work almost the same way as ConfigMaps, but they're meant for sensitive data.

### Create a Secret from Literal Values

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=super-secret-password
```

Output:
```
secret/db-secret created
```

View it:

```bash
kubectl get secret db-secret -o yaml
```

Output:
```yaml
apiVersion: v1
data:
  password: c3VwZXItc2VjcmV0LXBhc3N3b3Jk
  username: YWRtaW4=
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque
```

Notice the values are base64-encoded. That's not encryption—it's just encoding. Anyone with access to your cluster can decode them:

```bash
echo "YWRtaW4=" | base64 -d
```

Output:
```
admin
```

So Secrets aren't super secure by default, but they're better than ConfigMaps because Kubernetes handles them differently—they're not written to disk on nodes unless needed, and you can enable encryption at rest if you want.

### Create a Secret from a File

Create a file with sensitive data:

```bash
echo "my-super-secret-api-key" > api-key.txt
```

Create a Secret from it:

```bash
kubectl create secret generic api-secret --from-file=api-key.txt
```

Output:
```
secret/api-secret created
```

### Use Secret as Environment Variables

Create `pod-with-secret.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo-pod
spec:
  containers:
  - name: demo
    image: alpine:latest
    command: ["sh", "-c", "echo DB_USER=$DB_USER && echo DB_PASS=$DB_PASS && sleep 3600"]
    env:
    - name: DB_USER       # Environment variable name inside the container
      valueFrom:
        secretKeyRef:
          name: db-secret  # References the Secret we created earlier
          key: username    # Pulls the value of "username" key from that Secret
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret  # Same Secret as above
          key: password    # Pulls the value of "password" key
```

Apply it:

```bash
kubectl apply -f pod-with-secret.yaml
```

Output:
```
pod/secret-demo-pod created
```

Check the logs:

```bash
kubectl logs secret-demo-pod
```

Output:
```
DB_USER=admin
DB_PASS=super-secret-password
```

The secret values are automatically decoded and injected as plain text into your environment variables.

### Use Secret as a Volume

Just like ConfigMaps, you can mount Secrets as volumes. Create `pod-with-secret-volume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: demo
    image: alpine:latest
    command: ["sh", "-c", "cat /secrets/api-key.txt && sleep 3600"]
    volumeMounts:
    - name: secret-volume    # References the volume defined below
      mountPath: /secrets    # Where to mount the secret files inside the container
      readOnly: true         # Good security practice for secrets
  volumes:
  - name: secret-volume      # Volume name (must match volumeMounts above)
    secret:
      secretName: api-secret # References the Secret created from api-key.txt file
```

Apply it:

```bash
kubectl apply -f pod-with-secret-volume.yaml
```

Output:
```
pod/secret-volume-pod created
```

Check the logs:

```bash
kubectl logs secret-volume-pod
```

Output:
```
my-super-secret-api-key
```

The secret file is mounted and readable inside the container. Notice we set `readOnly: true`—that's a good practice for security.

## Practical Example: nginx with Custom Config

Let's put it all together. We'll deploy nginx with a custom config file from a ConfigMap, and also inject both ConfigMap and Secret values as environment variables. This shows how you can use multiple ConfigMaps and Secrets in a single deployment.

First, create a simple custom nginx config file:

```bash
cat > default.conf << EOF
server {
    listen 80;
    location / {
        return 200 'Hello from Kubernetes!\nThis config came from a ConfigMap.\n';
        add_header Content-Type text/plain;
    }
}
EOF
```

Create a ConfigMap from it:

```bash
kubectl create configmap nginx-config --from-file=default.conf
```

Output:
```
configmap/nginx-config created
```

Now create a deployment that uses both ConfigMap and Secret. Create `nginx-deployment-with-config.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-config
spec:
  replicas: 1
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
        env:
        # Environment variable from ConfigMap
        - name: APP_ENV                      # Variable name in the container
          valueFrom:
            configMapKeyRef:
              name: app-config               # References ConfigMap created with --from-literal
              key: APP_ENV                   # Gets the "APP_ENV" key value
        # Environment variable from Secret
        - name: API_KEY                      # Variable name in the container
          valueFrom:
            secretKeyRef:
              name: api-secret               # References Secret created from api-key.txt
              key: api-key.txt               # Gets the file content as the value
        volumeMounts:
        - name: nginx-config                 # Must match volume name below
          mountPath: /etc/nginx/conf.d/default.conf  # Exact file path in container
          subPath: default.conf              # Mounts only this file, not entire directory
      volumes:
      - name: nginx-config                   # Volume name referenced above
        configMap:
          name: nginx-config                 # References ConfigMap created from default.conf
```

Notice the `subPath: default.conf`—this mounts just the single file instead of the entire directory, which prevents nginx from having issues with its default config.

Apply it:

```bash
kubectl apply -f nginx-deployment-with-config.yaml
```

Output:
```
deployment.apps/nginx-with-config created
```

Wait for the pod to be ready:

```bash
kubectl get pods -l app=nginx
```

Output:
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-with-config-7d8c9f5b4-x9k2p   1/1     Running   0          15s
```

Test it with port-forwarding:

```bash
kubectl port-forward deployment/nginx-with-config 8080:80
```

In another terminal:

```bash
curl http://localhost:8080
```

Output:
```
Hello from Kubernetes!
This config came from a ConfigMap.
```

Perfect! Now let's verify the environment variables are available inside the container:

```bash
kubectl exec deployment/nginx-with-config -- sh -c 'echo "APP_ENV=$APP_ENV" && echo "API_KEY=$API_KEY"'
```

Output:
```
APP_ENV=production
API_KEY=my-super-secret-api-key
```

Your nginx is now running with configuration from a ConfigMap and has access to both the ConfigMap and Secret values through environment variables.

## Quick Experiments

### Update a ConfigMap

You can update a ConfigMap and see how it affects running pods. Note that environment variables won't update automatically—you need to restart the pod. But volume-mounted configs can update automatically (though it might take a minute).

```bash
kubectl edit configmap app-config
```

This opens an editor where you can change the values. Change `APP_ENV` from `production` to `staging` and save.

For the changes to take effect in pods using environment variables, you need to restart them:

```bash
kubectl rollout restart deployment/nginx-with-config
```

Output:
```
deployment.apps/nginx-with-config restarted
```

### View Secret Values

You already saw that secrets are just base64-encoded. Let's decode one:

```bash
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

Output:
```
super-secret-password
```

This is why you still need to protect access to your Kubernetes cluster—secrets aren't truly encrypted without additional setup.

## Clean Up

Delete all the resources we created:

```bash
kubectl delete pod config-demo-pod config-volume-pod secret-demo-pod secret-volume-pod
kubectl delete deployment nginx-with-config
kubectl delete configmap app-config app-config-file nginx-config
kubectl delete secret db-secret api-secret
rm -f app.properties api-key.txt default.conf
```

Output:
```
pod "config-demo-pod" deleted
pod "config-volume-pod" deleted
pod "secret-demo-pod" deleted
pod "secret-volume-pod" deleted
deployment.apps "nginx-with-config" deleted
configmap "app-config" deleted
configmap "app-config-file" deleted
configmap "nginx-config" deleted
secret "db-secret" deleted
secret "api-secret" deleted
```

## What You Learned

✅ How to create ConfigMaps from literal values and files  
✅ How to inject ConfigMaps as environment variables and volumes  
✅ How to create and use Secrets for sensitive data  
✅ How to mount Secrets in pods  
✅ The difference between ConfigMaps and Secrets  

That's it! You can now keep configuration separate from your container images, which makes your apps way more flexible when moving between environments.

## What's Next?

In **Part 3** of this series, we'll explore **Persistent Storage** in Kubernetes. You'll learn how to:
- Use volumes to persist data beyond pod lifecycles
- Work with Persistent Volumes and Persistent Volume Claims
- Deploy stateful applications like databases

Stay tuned!

---

## Series Navigation

**Part 1:** [Deploy Your First App](/posts/kubernetes-learning-path-deploy-your-first-app/)  
**Part 2:** ConfigMaps and Secrets ← You just finished this!  
**Part 3:** Persistent Storage (Coming soon)  
**Part 4:** Ingress and Load Balancing (Coming soon)

---

Found a mistake or have questions? Feel free to [open an issue here](https://github.com/shairyar/shairyar.github.io/issues/new).

