---
title: "Kubernetes Learning Path: Understanding Persistent Storage"
date: 2025-11-12 10:00:00 +0500
categories: [Kubernetes, Tutorial]
tags: [kubernetes, persistent-storage, pvc, statefulset, postgresql, storage]
series: kubernetes-learning-path
series_order: 6
---

# Kubernetes Learning Path: Understanding Persistent Storage

When I first started deploying applications on Kubernetes, I quickly ran into a problem: my database data kept disappearing every time a pod restarted. That's when I realized I needed to understand persistent storage.

In Kubernetes, pods are temporary—they can be created, destroyed, and recreated at any time. This is great for things like web servers that don't need to remember anything. But what about databases? They need to remember all your data even when the pod restarts. That's where persistent storage comes in.

In this post, I'll walk you through how to set up persistent storage in Kubernetes. We'll use something called a PersistentVolumeClaim (or PVC for short—think of it as asking for storage) and a StatefulSet (which is like a special type of pod that can use storage). We'll use a PostgreSQL database as our example.

## Why Persistent Storage Matters

If you deploy a PostgreSQL database using a regular Deployment, every time the pod restarts or gets recreated, all your data disappears. The container's filesystem is temporary. It only exists for the lifetime of the pod, and when that pod dies, so does everything inside it.

Persistent storage fixes this. Your data survives pod restarts, which means databases and other stateful applications actually work the way they're supposed to. You can also move data between nodes when needed.

## The Components: PVC and PV

Before jumping into the configuration, here's what you need to know:

**PersistentVolume (PV):** This is the actual storage space in your cluster. Think of it like a real hard drive that exists somewhere. Someone (either an admin or Kubernetes itself) sets this up.

**PersistentVolumeClaim (PVC):** This is you asking for storage. It's like going to a restaurant and saying "I need a table for 4 people." You're making a request. You tell Kubernetes "I need 5 gigabytes of storage" and Kubernetes finds or creates the actual storage (the PV) for you.

Here's how it works: You create a PVC (this is the configuration you provide). Kubernetes reads your PVC configuration and uses it to automatically create the actual storage (the PV). You don't manually create PVs—you just provide the PVC configuration, and Kubernetes handles creating the PV for you. But remember: if you don't provide a PVC configuration first, Kubernetes won't create any storage. The PVC configuration is what tells Kubernetes what storage to create.

## Setting Up Persistent Storage for PostgreSQL

I'll walk through a complete PostgreSQL setup with persistent storage. I'll go through each configuration file and explain what's happening.

### 1. PersistentVolumeClaim (PVC)

First up, we need to request storage. Here's the PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc              # Name of this PVC resource - we'll use this exact name in the StatefulSet below to connect them
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce                # Volume can be mounted read-write by a single node
  resources:
    requests:
      storage: 5Gi                 # Requesting 5 gigabytes of storage
```

The important parts here:
- **`name: postgres-pvc`:** This is the name we're giving to this storage resource. The StatefulSet (which contains the PostgreSQL database pod) will use this exact name to connect to this storage. Think of the PVC as the hard drive, and the StatefulSet as the computer that needs to use that hard drive. We'll reference this name in the StatefulSet configuration below.
- **`accessModes: ReadWriteOnce`:** This means only one computer (node) can use this storage at a time, and it can both read and write. This is perfect for databases. There are other options (like letting multiple computers use it), but for databases, you almost always want ReadWriteOnce.
- **`storage: 5Gi`:** We're asking for 5 gigabytes. Change this to whatever you actually need.

### 2. StatefulSet for PostgreSQL

For databases and other apps that need to remember things, you need a StatefulSet, not a regular Deployment. The main difference? StatefulSets can use persistent storage, and they give each pod a stable name that doesn't change. This is important for databases.

Here's the StatefulSet:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: default
spec:
  serviceName: postgres            # This name must match the Service name we'll create below (connects StatefulSet to Service)
  replicas: 1
  selector:
    matchLabels:
      app: postgres                 # This label must match the label in the pod template below (in this same file)
  template:
    metadata:
      labels:
        app: postgres               # This label must match the selector above (in this same file) and the Service selector below
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - secretRef:
            name: postgres-secret   # References the Secret created earlier (loads all keys as env vars)
        volumeMounts:
        - name: postgres-storage    # This name must match the volume name in the volumes section below (in this same file)
          mountPath: /var/lib/postgresql/data  # Where PostgreSQL stores its data inside the container
          subPath: postgres         # Creates a subdirectory to prevent permission issues
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)      # Uses environment variable from postgres-secret
            - -d
            - $(POSTGRES_DB)        # Uses environment variable from postgres-secret
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - $(POSTGRES_USER)      # Uses environment variable from postgres-secret
            - -d
            - $(POSTGRES_DB)        # Uses environment variable from postgres-secret
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage      # This name must match the volumeMounts.name above (in this same file) - connects the volume to the mount
        persistentVolumeClaim:
          claimName: postgres-pvc   # This is the name from the PVC we created earlier - connects this StatefulSet to that storage
```

A few things to note:

**`serviceName: postgres`:** This tells the StatefulSet to use a headless service (we'll create that next) for stable network identity.

**`volumeMounts`:** This is where we connect the storage to the container. The `mountPath` is just a folder path inside the container where PostgreSQL will save its data (`/var/lib/postgresql/data` is PostgreSQL's default data folder). The `subPath: postgres` part creates a subfolder inside the storage—this prevents permission problems. I learned this the hard way when PostgreSQL couldn't write to the storage because of permission issues.

**`volumes`:** This is where we tell the StatefulSet which storage to use. We reference the PVC name (`postgres-pvc`) that we created earlier. This connects the storage to the container.

**Health checks:** These check if PostgreSQL is actually working, not just if the container is running. They use `pg_isready` which is a PostgreSQL command. Notice we're using `$(POSTGRES_USER)` and `$(POSTGRES_DB)`—these are environment variables that come from the Secret. This way, if we change the Secret, the health checks automatically use the new values.

### 3. Headless Service

StatefulSets need a special type of service called a "headless service." A normal service gives pods a single IP address to share, but a headless service doesn't—it lets each pod have its own address. This is important for StatefulSets because each pod needs its own identity. Here's what it looks like:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres                   # This name must match the StatefulSet spec.serviceName above (connects Service to StatefulSet)
  namespace: default
spec:
  selector:
    app: postgres                  # This label must match the label in StatefulSet template.metadata.labels above (connects Service to pods)
  ports:
  - port: 5432                     # Port exposed by the service
    targetPort: 5432               # Port on the container (matches containerPort in StatefulSet)
  clusterIP: None                  # Headless service - required for StatefulSets
```

The `clusterIP: None` is what makes it "headless"—it means there's no shared IP address. Instead, each pod gets its own address, which is what StatefulSets need.

### 4. Secret for Database Credentials

We'll store the database credentials in a Secret (I covered this in [Part 2](/posts/kubernetes-learning-path-configmaps-and-secrets/)):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret             # This name is referenced in StatefulSet envFrom.secretRef
  namespace: default
type: Opaque
stringData:
  # PostgreSQL initialization variables (used by postgres image)
  POSTGRES_USER: rails_user
  POSTGRES_PASSWORD: secure_password_change_me
  POSTGRES_DB: myapp_production
```

## How These Pieces Fit Together

Before we deploy, let me explain how these three components connect to each other. Think of it like this:

**PersistentVolumeClaim (PVC)** = Your storage (like a hard drive)  
**StatefulSet** = Your PostgreSQL database pod (the computer that needs the hard drive)  
**Service** = The way other pods can find and talk to your database

Here's how they link together:

```
┌─────────────────┐
│   Service       │  ← Gives your database a stable address
│   (postgres)    │     so other pods can find it
└────────┬────────┘
         │ (finds pods by label: app=postgres)
         │
         ▼
┌─────────────────┐
│  StatefulSet    │  ← Your PostgreSQL database pod
│  (postgres)     │     - Uses Service name: "postgres" (connects to Service above)
└────────┬────────┘     - Has label: app=postgres (Service uses this to find it)
         │
         │ (uses PVC name: "postgres-pvc")
         ▼
┌─────────────────┐
│      PVC        │  ← Your storage (where database saves data)
│  (postgres-pvc) │
└─────────────────┘
```

In simple terms:
1. **StatefulSet connects to PVC**: The StatefulSet says "I need storage" and uses the PVC name `postgres-pvc` to get it. This is like plugging a hard drive into your computer.
2. **Service connects to StatefulSet**: The Service helps other pods find your database. It looks for pods with the label `app=postgres` (which is what the StatefulSet creates). Also, the Service name must be `postgres` because that's what the StatefulSet is looking for.
3. **Everything stays in sync**: When the StatefulSet creates a pod, that pod has the label `app=postgres`, so the Service automatically finds it. The pod also gets the storage from the PVC, so your data persists.

It's like a chain: Service → finds → StatefulSet pods → uses → PVC storage.

## Deploying Everything

With all the pieces ready, time to deploy:

```bash
# Apply all configurations
kubectl apply -f postgres-secret.yaml
kubectl apply -f postgres-pvc.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f postgres-statefulset.yaml
```

Or if all files are in the same directory:

```bash
kubectl apply -f postgres-secret.yaml postgres-pvc.yaml postgres-service.yaml postgres-statefulset.yaml
```

## Verifying the Setup

Check that everything is working:

```bash
# Check the PVC status
kubectl get pvc

# Output should show:
# NAME           STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# postgres-pvc   Bound    pvc-xxx  5Gi        RWO            local-path     10s
```

You should see `STATUS` as `Bound`—that means your request for storage was successful! Kubernetes found or created the actual storage and connected it to your PVC. If it says "Pending," something went wrong.

```bash
# Check the StatefulSet and pods
kubectl get statefulset
kubectl get pods -l app=postgres

# Check pod details to see the volume mount
kubectl describe pod postgres-0
```

In the pod description, look for the volume mount under `Mounts:`. If you see it listed there, that means the storage is actually connected to your pod and ready to use.

## Testing Persistence

The real test is whether your data actually survives a pod restart. Here's how to verify:

```bash
# Connect to the database and create some test data
kubectl exec -it postgres-0 -- psql -U rails_user -d myapp_production

# Inside psql:
CREATE TABLE test_persistence (id SERIAL PRIMARY KEY, message TEXT);
INSERT INTO test_persistence (message) VALUES ('This should persist!');
SELECT * FROM test_persistence;
\q
```

Now delete the pod and watch Kubernetes recreate it:

```bash
# Delete the pod
kubectl delete pod postgres-0

# Watch it get recreated
kubectl get pods -l app=postgres -w
```

Once the new pod is running, check if the data is still there:

```bash
# Connect again and check the data
kubectl exec -it postgres-0 -- psql -U rails_user -d myapp_production -c "SELECT * FROM test_persistence;"
```

If you see your test data, you're all set! Persistent storage is working.

## Common Issues and Solutions

**Permission errors:**
- Make sure you're using `subPath` in your volume mount. This is usually the culprit.

**Storage not persisting:**
- Check that the PVC status is `Bound` with `kubectl get pvc` (if it says "Pending," the storage wasn't created)
- Verify the volume is actually mounted: `kubectl describe pod <pod-name>` and look for the mount in the output
- Make sure you're using a StatefulSet, not a Deployment. Regular Deployments can lose data when pods restart, even if you attach storage to them. StatefulSets are designed to work with persistent storage.

## Conclusion

Persistent storage is essential for running databases and other apps that need to remember things in Kubernetes. It adds a bit of complexity compared to simple web apps, but once you understand how PVCs (your storage request), PVs (the actual storage), and StatefulSets (the pods that use the storage) work together, it's pretty straightforward.

## What's Next?

I'm planning to cover backup and restore strategies for persistent volumes, and how to set up multi-replica databases with persistent storage. Stay tuned!

---

## Series Navigation

**Part 1:** [Deploy Your First App](/posts/kubernetes-learning-path-deploy-your-first-app/)  
**Part 2:** [ConfigMaps and Secrets](/posts/kubernetes-learning-path-configmaps-and-secrets/)  
**Part 3:** [Understanding Namespaces](/posts/kubernetes-learning-path-namespaces/)  
**Part 4:** [Understanding Port Mapping in k3d](/posts/kubernetes-learning-path-port-mapping/)  
**Part 5:** [Setting Up k3s on Raspberry Pi](/posts/kubernetes-learning-path-k3s-on-raspberry-pi/)  
**Part 6:** Understanding Persistent Storage ← You just finished this!

---

