---
title: "Kubernetes Learning Path: Deploying Rails 8 with SolidQueue on Raspberry Pi k3s"
date: 2025-10-31 14:00:00 +0500
categories: [Kubernetes, Tutorial]
tags: [kubernetes, k3s, raspberry-pi, ruby-on-rails, solidqueue, postgresql, deployment]
series: kubernetes-learning-path
series_order: 6
---

# Kubernetes Learning Path: Deploying Rails 8 with SolidQueue on Raspberry Pi k3s

After setting up my 3-node Raspberry Pi k3s cluster in the previous post, I wanted to deploy something real—not just nginx demos, but an actual production-grade application. So I decided to deploy a Ruby on Rails 8 application with all the modern bells and whistles: PostgreSQL, background jobs with SolidQueue, database-backed caching with SolidCache, and proper health checks.

This wasn't a weekend afternoon project. It took me a few days of trial and error, reading documentation, debugging architecture mismatches, and learning a lot about how Rails 8's new features work in a Kubernetes environment. But once everything clicked into place and I saw my Rails app running across multiple pods, processing background jobs on separate workers, all on my little Pi cluster—it felt incredible.

Let me walk you through what I built, what went wrong, and what I learned.

## Why Rails 8?

I'll be honest—I chose Rails because I'm already familiar with it. The goal here was to deploy a complete, production-grade application stack on Kubernetes, not to learn a new framework at the same time. Trying to learn Kubernetes *and* a new language/framework simultaneously would have been overwhelming.

Rails was the obvious choice because I know how it works, I understand its conventions, and I can focus on the Kubernetes side of things without getting lost in framework documentation.

**The Pleasant Surprise: Rails 8's Built-in Features**

What I didn't expect was how well Rails 8 fits into Kubernetes. It comes with SolidQueue and SolidCache built-in—database-backed solutions for background jobs and caching. Before Rails 8, you'd typically need Redis for Sidekiq (background jobs) and Redis or Memcached for caching.

With Rails 8, everything uses PostgreSQL:
- Application data → PostgreSQL
- Background job queue → PostgreSQL (via SolidQueue)
- Cache storage → PostgreSQL (via SolidCache)

## The Architecture We're Building

Here's what I deployed in this phase:

```
┌─────────────────────────────────────────────┐
│         k3s Cluster (3 Raspberry Pis)       │
│                                             │
│  ┌────────────┐  ┌────────────┐            │
│  │ Rails Web  │  │ Rails Web  │ (Pods)     │
│  │  Pod x3    │  │  Pod x3    │            │
│  └─────┬──────┘  └─────┬──────┘            │
│        │                │                    │
│        └────────┬───────┘                    │
│                 │                            │
│        ┌────────▼────────┐                  │
│        │  Rails Service  │                  │
│        │   (ClusterIP)   │                  │
│        │ Internal Only   │                  │
│        └────────┬────────┘                  │
│                 │                            │
│  ┌──────────────▼──────────────┐            │
│  │   SolidQueue Workers (x2)   │            │
│  │ (Background Job Processors) │            │
│  └──────────────┬──────────────┘            │
│                 │                            │
│        ┌────────▼────────┐                  │
│        │  PostgreSQL 17  │                  │
│        │  (StatefulSet)  │                  │
│        │ + Persistent    │                  │
│        │     Storage     │                  │
│        └─────────────────┘                  │
│                                             │
└─────────────────────────────────────────────┘

Access via: kubectl port-forward (for now)
```

**What I deployed:**
- **3 Rails web server pods** (Deployment)
- **2 SolidQueue worker pods** (Deployment)
- **1 PostgreSQL pod** (StatefulSet with 5GB persistent storage)
- **2 Services:** rails-service (ClusterIP) and postgres (Headless)

**What's next (future posts):**
- Traefik Ingress for external access
- Persistent volumes for file uploads
- Monitoring with Mission Control Jobs
- SSL/TLS with cert-manager

## What You'll Need

Before starting, make sure you have:

- A working k3s cluster on Raspberry Pi (from [Part 5](/posts/kubernetes-learning-path-k3s-on-raspberry-pi/))
- Docker installed on your laptop for building images
- A Docker Hub account (free tier works fine)
- `kubectl` configured to access your cluster
- Basic understanding of Rails (helpful but not required)

## The Big Challenge: Cross-Architecture Builds

This was my first major roadblock. I develop on an x86_64 laptop (Intel/AMD architecture), but Raspberry Pi uses ARM64 architecture. If you build a Docker image on your laptop without specifying the platform, it won't run on the Pi.

The symptom? Pods that immediately fail with cryptic errors like `exec format error` or `exec /bin/sh: exec format error`.

### Docker Hub Setup (One-Time)

Before building and pushing images, you need a Docker Hub account and to log in:

```bash
# Create a free account at https://hub.docker.com if you don't have one
# Free tier includes unlimited public repos

# Login to Docker Hub (do this once on each machine you build from)
docker login
# Enter your Docker Hub username and password
```

Once logged in, you can push images to your Docker Hub account.

**Solution 1: Build on the Raspberry Pi directly**

I found out this to be the most reliable method:

```bash
# SSH to your master node
ssh pi@192.168.18.49

# If you haven't logged in on the Pi yet:
docker login

# Build the image directly on ARM64 hardware
docker build -t your-dockerhub-username/my-rails-app:v1 .
docker push your-dockerhub-username/my-rails-app:v1
```

Replace `your-dockerhub-username` with your actual Docker Hub username.

Building on the Pi takes about 15-20 minutes because, well, it's a Raspberry Pi. But you'll get a native ARM64 image that works perfectly.

**Solution 2: Cross-compile on your laptop with QEMU**

You can also cross-compile on your laptop if you don't want to SSH to the Pi. This requires QEMU emulation:

```bash
# On Arch Linux (my setup)
sudo pacman -S qemu-user-static qemu-user-static-binfmt
sudo systemctl start systemd-binfmt.service

# Verify QEMU is working
ls /proc/sys/fs/binfmt_misc/ | grep qemu
# Should show: qemu-aarch64, qemu-arm, etc.

# Now build for ARM64 from your laptop
docker buildx build --platform linux/arm64 \
  -t your-dockerhub-username/my-rails-app:v1 \
  --push .
```

**Note:** In my testing, building directly on the Pi was actually *faster* than cross-compiling with QEMU emulation. QEMU adds overhead, and the Pi's native ARM64 build was more efficient. The cross-compile option is mainly useful if you can't SSH to the Pi or want to automate builds from your laptop.

**Lesson learned:** Always specify `--platform linux/arm64` when building for Raspberry Pi. I wasted several hours debugging "exec format error" before realizing my x86_64 images wouldn't run on ARM64.

## Step 1: Prepare the Rails Application

I created a Rails 8.1.1 application with all the modern defaults. If you want to follow along, here's the quick setup:

```bash
# Create new Rails 8 app
rails new my-rails-app --database=postgresql

# Rails 8 automatically includes:
# ✅ Solid Queue (background jobs)
# ✅ Solid Cache (caching)
# ✅ Dockerfile (production-ready, multi-stage)
# ✅ Health check endpoint (/up)
```

The generated Dockerfile is already optimized for production. Rails 8 does a great job here.

### Important Configuration Changes

Rails 8 production mode assumes SSL by default, which caused CSRF errors when I tried to access the app via `kubectl port-forward` (which uses HTTP, not HTTPS).

**Make SSL configurable in `config/environments/production.rb`:**

```ruby
# Allow disabling SSL for testing (keep enabled in real production)
config.assume_ssl = ENV.fetch("RAILS_ASSUME_SSL", "true") == "true"
config.force_ssl = ENV.fetch("RAILS_FORCE_SSL", "true") == "true"
```

**Fix database connection in `config/database.yml`:**

Rails defaults to Unix socket connections for PostgreSQL, but in Kubernetes we need TCP connections:

```yaml
production:
  <<: *default
  database: myapp_production
  username: <%= ENV.fetch("DATABASE_USERNAME") { "rails_user" } %>
  password: <%= ENV["DATABASE_PASSWORD"] %>
  host: <%= ENV.fetch("DATABASE_HOST") { "localhost" } %>
  port: <%= ENV.fetch("DATABASE_PORT") { "5432" } %>
```

The key here is `host:` reading from `DATABASE_HOST` environment variable, which will point to our PostgreSQL service.

## Step 2: Build and Push the Docker Image

I built the image on my Raspberry Pi master node for guaranteed compatibility:

```bash
# SSH to the Pi
ssh pi@192.168.18.49

# Clone your app or copy it over
# Then build:
cd my-rails-app
docker build -t your-username/my-rails-app:v1 .
docker push your-username/my-rails-app:v1
```

Building took about 18 minutes on my Pi 4. Grab some coffee.

**Important:** This single image is reused for:
- Rails web server pods (default CMD)
- SolidQueue worker pods (override CMD)
- Database migration init container (runs before web pods start)

One image, multiple purposes. That's the beauty of the Rails 8 Dockerfile.

## Step 3: Deploy PostgreSQL to Kubernetes

Before Rails can start, we need the database running. I created four YAML files for PostgreSQL deployment.

### PostgreSQL Secret

First, we need to store database credentials securely.

**Why we need this:** PostgreSQL needs a username, password, and database name to initialize. Kubernetes Secrets store this sensitive data (base64-encoded, can be encrypted at rest).

**Where it's used:** The PostgreSQL StatefulSet reads these values as environment variables to create the database and user.

**`postgres-secret.yaml`:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_USER: rails_user
  POSTGRES_PASSWORD: change_this_password_123
  POSTGRES_DB: myapp_production
```

**⚠️ Important:** Change that password! Never use default passwords in production.

Apply it:

```bash
kubectl apply -f postgres-secret.yaml
```

### Persistent Storage

Databases need storage that survives pod restarts.

**Why we need this:** Without persistent storage, all database data would be lost when the PostgreSQL pod restarts or crashes. PersistentVolumeClaims (PVCs) request storage from Kubernetes that persists independently of pods.

**Where it's used:** The PostgreSQL StatefulSet mounts this volume to `/var/lib/postgresql/data` where PostgreSQL stores its data files.

**`postgres-pvc.yaml`:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce  # Single node can mount for read/write
  resources:
    requests:
      storage: 5Gi  # Request 5GB of storage
```

Apply it:

```bash
kubectl apply -f postgres-pvc.yaml

# Wait for it to bind
kubectl get pvc -w
# Press Ctrl+C when STATUS shows "Bound"
```

k3s automatically provisions local storage on one of your Pi nodes.

### PostgreSQL StatefulSet

Now for the actual database pod.

**Why we need this:** We use a StatefulSet (not a Deployment) because databases need:
- Stable pod names (always `postgres-0`, not random names)
- Ordered startup and shutdown
- Stable persistent storage that follows the pod

**Where it's used:** This creates the PostgreSQL pod that Rails and SolidQueue will connect to for all database operations.

**`postgres-statefulset.yaml`:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres  # ← Links to postgres-service above
  replicas: 1
  selector:
    matchLabels:
      app: postgres  # ← StatefulSet manages pods with this label
  template:
    metadata:
      labels:
        app: postgres  # ← Each pod gets this label
                       # postgres-service selector matches this to route traffic
    spec:
      containers:
      - name: postgres
        image: postgres:17-alpine
        ports:
        - containerPort: 5432  # ← Service targetPort forwards here
        envFrom:
        - secretRef:
            name: postgres-secret  # ← Injects POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
          subPath: postgres
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc  # ← Mounts storage from postgres-pvc.yaml above
```

Apply it:

```bash
kubectl apply -f postgres-statefulset.yaml

# Watch PostgreSQL start
kubectl get pods -l app=postgres -w
# Wait for STATUS: Running, READY: 1/1
```

### PostgreSQL Service

Now let's create a stable DNS name for the database.

**Why we need this:** Pod IP addresses change when they restart. A Service gives us a stable DNS name (`postgres`) that Rails can use to connect. Even if the PostgreSQL pod restarts and gets a new IP, the service name stays the same.

**Where it's used:** Rails' `database.yml` will use `host: postgres` to connect. Kubernetes DNS automatically resolves this to the PostgreSQL pod's IP.

**`postgres-service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres  # ← DNS name: pods can connect via "postgres:5432"
spec:
  selector:
    app: postgres  # ← Finds pods with label "app: postgres"
                   # (matches labels in postgres-statefulset.yaml below)
  ports:
  - port: 5432        # ← Service listens on this port
    targetPort: 5432  # ← Forwards to pod's containerPort 5432
  clusterIP: None     # ← Headless service for StatefulSet
```

Apply it:

```bash
kubectl apply -f postgres-service.yaml
```

Now any pod can connect to PostgreSQL using the hostname `postgres` on port `5432`. This is what we configured in `database.yml`.

### Verify PostgreSQL is Running

```bash
# Check pod status
kubectl get pods -l app=postgres

# Test connection
kubectl exec -it postgres-0 -- psql -U rails_user -d myapp_production -c "SELECT version();"
```

You should see PostgreSQL version info. Database is ready!

## Step 4: Configure Rails for Kubernetes

Rails needs configuration and secrets to run in Kubernetes. I split these into ConfigMaps (non-sensitive config) and Secrets (sensitive stuff like passwords).

### Rails ConfigMap

**Why we need this:** Rails needs various environment variables to run (database host, Rails environment, logging settings, etc.). ConfigMaps store non-sensitive configuration that can be shared across all Rails and worker pods.

**Where it's used:** Both the Rails web deployment and SolidQueue worker deployment will inject these environment variables into their pods.

**`rails-configmap.yaml`:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rails-config  # ← Referenced by rails-deployment and solid-queue-deployment
data:
  RAILS_ENV: "production"
  RAILS_LOG_TO_STDOUT: "true"
  DATABASE_HOST: "postgres"  # ← Kubernetes DNS resolves "postgres" to postgres-service
                             # which routes to postgres pods
  DATABASE_PORT: "5432"
  RAILS_ASSUME_SSL: "false"  # For testing without SSL
  RAILS_FORCE_SSL: "false"
  SOLID_QUEUE_DISPATCHERS_POLLING_INTERVAL: "1"
  SOLID_QUEUE_WORKERS_POLLING_INTERVAL: "0.1"
```

### Rails Secret

**Why we need this:** Rails requires sensitive data like the secret key (for encrypting sessions/cookies) and database credentials. Unlike ConfigMaps, Secrets are base64-encoded and can be encrypted at rest for better security.

**Where it's used:** The Rails web pods and SolidQueue workers need these secrets to connect to the database and encrypt user sessions.

First, generate a secret key. You can do this from **your laptop** or from the Pi—doesn't matter:

```bash
# Option 1: Using Rails (run this on your laptop)
docker run --platform linux/arm64 --rm your-username/my-rails-app:v1 ./bin/rails secret

# Option 2: Using OpenSSL (simpler - run this on your laptop)
openssl rand -hex 64
```

You'll get a long random string like:
```
f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9
```

Copy this output—you'll paste it into the YAML file in the next step.

Now create the secret file:

**`rails-secret.yaml`:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rails-secret  # ← Referenced by rails-deployment and solid-queue-deployment
type: Opaque
stringData:
  SECRET_KEY_BASE: "your-generated-secret-key-paste-here"
  DATABASE_PASSWORD: "change_this_password_123"  # ← Must match postgres-secret.yaml
  DATABASE_USERNAME: "rails_user"
  DATABASE_NAME: "myapp_production"
```

**⚠️ Critical:** `DATABASE_PASSWORD` must exactly match what you set in `postgres-secret.yaml`!

Apply both:

```bash
kubectl apply -f rails-configmap.yaml
kubectl apply -f rails-secret.yaml
```

## Step 5: Deploy Rails Web Application

Now for the main event—deploying Rails itself.

### Rails Deployment with Init Container

This is where it gets interesting. The init container runs database migrations before the main Rails container starts.

**Why we need this:** This is the core web application. The Deployment creates 3 Rails pods running Puma (the web server) to handle HTTP requests. We use a Deployment (not StatefulSet) because web servers are stateless—any pod can handle any request.

**The init container trick:** Before the web server starts, an init container runs database migrations. This ensures the database schema is up-to-date before Rails starts serving requests.

**Where it's used:** These pods handle all HTTP requests to your Rails application. The Service (next step) will load-balance traffic across all 3 pods.

**`rails-deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-app
spec:
  replicas: 3  # Three web server pods
  selector:
    matchLabels:
      app: rails  # ← Deployment manages pods with this label
  template:
    metadata:
      labels:
        app: rails  # ← Each pod gets this label
                    # rails-service.yaml selector "app: rails" finds these pods
    spec:
      initContainers:
      - name: db-setup
        image: your-username/my-rails-app:v1
        command:
          - sh
          - -c
          - |
            bundle exec rails db:create || echo "Databases may exist"
            bundle exec rails db:prepare
            bundle exec rails solid_queue:install:migrations
            bundle exec rails solid_cache:install:migrations
            bundle exec rails db:migrate
        envFrom:
        - configMapRef:
            name: rails-config  # ← Injects DATABASE_HOST="postgres" etc.
        - secretRef:
            name: rails-secret  # ← Injects DATABASE_PASSWORD, SECRET_KEY_BASE
      containers:
      - name: web
        image: your-username/my-rails-app:v1
        ports:
        - containerPort: 3000  # ← rails-service targetPort forwards here
        envFrom:
        - configMapRef:
            name: rails-config  # ← Rails reads DATABASE_HOST to connect to postgres
        - secretRef:
            name: rails-secret  # ← Rails reads DATABASE_PASSWORD to authenticate
        livenessProbe:
          httpGet:
            path: /up
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /up
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

**What's happening here:**
1. **Init container** runs first: creates databases, installs Solid Queue/Cache migrations, runs migrations
2. Only after init succeeds, **main container** starts: Rails web server (Puma)
3. **Health checks** ensure Rails is responding before routing traffic
4. **Resource limits** prevent any pod from hogging the Pi's limited RAM

Apply it:

```bash
kubectl apply -f rails-deployment.yaml

# Watch pods start (this takes longer due to init container)
kubectl get pods -l app=rails -w
```

You'll see pods in `Init:0/1` status while migrations run. Check init logs:

```bash
# Get pod name
kubectl get pods -l app=rails

# Check init container logs
kubectl logs <pod-name> -c db-setup
```

You should see migration output. Once init completes, pods transition to `Running` status.

### A Nasty Bug I Hit: Multi-Database Setup

Rails 8 uses separate databases for cache and queue by default:
- `myapp_production` (primary)
- `myapp_production_cache` (cache)
- `myapp_production_queue` (queue)
- `myapp_production_cable` (action cable)

PostgreSQL only auto-creates `myapp_production`. The init container failed with "Database myapp_production_cache does not exist".

**Solution:** Add `db:create` before `db:prepare` in the init container:

```bash
bundle exec rails db:create || echo "Databases may exist"
bundle exec rails db:prepare
```

The `|| echo` prevents failure if databases already exist. Took me an hour to figure this out.

### Rails Service

Time to create a load balancer for the web pods.

**Why we need this:** Just like with PostgreSQL, we need a stable way to access the Rails pods. This Service gives us a single DNS name (`rails-service`) and automatically distributes incoming requests across all healthy Rails pods.

**Where it's used:** We'll use `kubectl port-forward service/rails-service` to access the app from our laptop. Later, an Ingress controller would route external traffic to this service.

**`rails-service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rails-service  # ← DNS name for accessing Rails internally
spec:
  type: ClusterIP  # ← Internal only (use Ingress for external access)
  selector:
    app: rails  # ← Finds pods with label "app: rails"
                # (matches labels in rails-deployment.yaml)
  ports:
  - port: 80          # ← Service listens on port 80
    targetPort: 3000  # ← Forwards to pod's containerPort 3000 (Puma)
```

Apply it:

```bash
kubectl apply -f rails-service.yaml

# Check endpoints (should list all 3 Rails pod IPs)
kubectl get endpoints rails-service
```

The service automatically distributes traffic across your 3 Rails pods.

## Step 6: Deploy SolidQueue Workers

> **Note:** This is where I stopped in my initial deployment. Steps 7-8 below cover testing and troubleshooting what I've deployed so far. Future work like Ingress and persistent storage for uploads will be covered in upcoming posts.

Background jobs need dedicated worker processes.

**Why we need this:** Background jobs (sending emails, processing uploads, etc.) shouldn't run on the web server pods—they'd compete for resources and slow down HTTP responses. SolidQueue workers are dedicated pods that poll the database for jobs and process them.

**The clever part:** We use the **exact same Docker image** (`your-username/my-rails-app:v1`) for both web servers and workers. The only difference is what command we tell the container to run:

- **Web pods:** Run the default command from the Dockerfile → Puma web server starts (`rails server`)
- **Worker pods:** Override the command in the deployment YAML → SolidQueue starts (`bundle exec rake solid_queue:start`)

This means you only need to build and maintain one Docker image for your entire Rails stack. Same codebase, different processes.

**Where it's used:** These workers continuously poll the `solid_queue_jobs` table in PostgreSQL, pick up new jobs, process them, and mark them complete.

**`solid-queue-deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: solid-queue-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: solid-queue-worker  # ← Deployment manages pods with this label
  template:
    metadata:
      labels:
        app: solid-queue-worker  # ← Each worker pod gets this label
    spec:
      containers:
      - name: worker
        image: your-username/my-rails-app:v1  # ← Same image as rails-deployment
        command:  # ← Different command: runs SolidQueue instead of web server
          - bundle
          - exec
          - rake
          - solid_queue:start
        envFrom:
        - configMapRef:
            name: rails-config  # ← Workers use same DATABASE_HOST="postgres" to connect
        - secretRef:
            name: rails-secret  # ← Workers use same DATABASE_PASSWORD to authenticate
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

Apply it:

```bash
kubectl apply -f solid-queue-deployment.yaml

# Watch workers start
kubectl get pods -l app=solid-queue-worker -w
```

Check worker logs:

```bash
kubectl logs -f deployment/solid-queue-worker
```

You should see SolidQueue starting up and polling for jobs.

### Understanding the Command Difference

Now that you've deployed both the Rails web servers and SolidQueue workers, let's see exactly how Kubernetes runs different commands from the same Docker image.

**In the Rails deployment** (`rails-deployment.yaml`), we don't specify a `command:` field, so Kubernetes uses the default from the Dockerfile:
```yaml
containers:
- name: web
  image: your-username/my-rails-app:v1
  # No command: specified = uses Dockerfile default (Puma web server)
```

**In the worker deployment** (`solid-queue-deployment.yaml`), we override the command:
```yaml
containers:
- name: worker
  image: your-username/my-rails-app:v1  # Same image!
  command:  # This overrides the Dockerfile default
    - bundle
    - exec
    - rake
    - solid_queue:start
```

This is a powerful Kubernetes pattern: the `command:` field in your deployment YAML can override whatever CMD or ENTRYPOINT is defined in the Dockerfile. One image, multiple uses!

## Step 7: Test the Full Stack

Time to see if everything works together.

### Access Rails Application

Use port-forwarding to access the app:

```bash
kubectl port-forward service/rails-service 3000:80
```

Open `http://localhost:3000` in your browser. You should see your Rails app!

### Test Background Jobs

Open a Rails console:

```bash
kubectl exec -it deployment/rails-app -- rails console
```

In the console, queue a test job:

```ruby
# Create a simple job class if you don't have one
class TestJob < ApplicationJob
  def perform(message)
    puts "Processing: #{message}"
  end
end

# Queue the job
TestJob.perform_later("Hello from Kubernetes!")

# Check job count
SolidQueue::Job.count
# => 1

# Exit console
exit
```

Now check the worker logs:

```bash
kubectl logs -f deployment/solid-queue-worker
```

You should see the worker picking up and processing the job! If you see "Processing: Hello from Kubernetes!" in the logs, your entire stack is working.

## Step 8: Challenges I Hit and How I Solved Them

This deployment wasn't without its struggles. Here are the main issues I encountered and how I fixed them:

### 1. Cross-Architecture Exec Format Error

**Problem:** Built image on x86_64 laptop, pods failed with "exec /bin/sh: exec format error" on ARM64 Pi.

**Solution:** Build with `--platform linux/arm64` on laptop with QEMU, or build directly on Pi.

**Lesson:** Always specify target platform for cross-compilation.

### 2. PostgreSQL Socket Connection Error

**Problem:** Init container failed: "connection to socket '/var/run/postgresql/.s.PGSQL.5432' failed"

**Solution:** Rails was trying Unix socket. Fixed by ensuring `database.yml` reads `DATABASE_HOST` from env var (TCP connection).

**Lesson:** Kubernetes requires TCP connections between pods, not Unix sockets.

### 3. Multi-Database Creation

**Problem:** Init failed: "Database myapp_production_cache does not exist"

**Solution:** Added `rails db:create` before `db:prepare` in init container to create all databases.

**Lesson:** Rails 8 multi-database setup requires explicit database creation.

### 4. CSRF Errors with Port-Forward

**Problem:** Got "HTTP Origin header didn't match request.base_url" when accessing via port-forward.

**Solution:** Made `force_ssl` and `assume_ssl` configurable via env vars, disabled for local testing.

**Lesson:** Production Rails expects HTTPS. For local testing without SSL, make it configurable.

### 5. Image Caching Issues

**Problem:** Pushed updated image but pods still ran old cached version.

**Solution:** Used new tag (`v2`, `v3`) instead of reusing `v1`. Kubernetes caches by tag.

**Lesson:** Use unique tags for each build, or set `imagePullPolicy: Always`.

## Monitoring Your Application

### Check Pod Status

```bash
# All pods
kubectl get pods

# Just Rails pods
kubectl get pods -l app=rails

# Just workers
kubectl get pods -l app=solid-queue-worker

# Detailed pod info
kubectl describe pod <pod-name>
```

### View Logs

```bash
# Rails web logs
kubectl logs -f deployment/rails-app

# Worker logs
kubectl logs -f deployment/solid-queue-worker

# PostgreSQL logs
kubectl logs postgres-0

# Logs from specific pod
kubectl logs <pod-name>

# Previous crashed container logs
kubectl logs <pod-name> --previous
```

### Rails Console Access

```bash
# Open console in a web pod
kubectl exec -it deployment/rails-app -- rails console

# Check database connection
> ActiveRecord::Base.connection.execute('SELECT 1')

# Check job queue
> SolidQueue::Job.count
> SolidQueue::Job.pending.count
> SolidQueue::Job.failed.count

# Exit
> exit
```

## What You Learned

✅ How to deploy a production Rails 8 application on Kubernetes  
✅ StatefulSets vs Deployments (databases vs stateless apps)  
✅ Persistent storage with PersistentVolumeClaims  
✅ Init containers for database migrations  
✅ ConfigMaps and Secrets for configuration management  
✅ Services for load balancing and service discovery  
✅ Health checks with liveness and readiness probes  
✅ Cross-architecture Docker builds (x86_64 → ARM64)  
✅ SolidQueue for background jobs in Kubernetes  
✅ Resource limits and requests on Raspberry Pi  

## Conclusion

Deploying Rails 8 with SolidQueue on my Raspberry Pi k3s cluster was one of the most educational projects I've done. It's one thing to deploy nginx demos; it's another to deploy a full production application stack with database, background jobs, and health checks—all running on $35 computers in a distributed cluster.

The challenges were real: architecture mismatches, database connection errors, multi-database setup, SSL configuration. But each problem taught me something valuable about how Kubernetes works and how modern Rails is designed to run in containerized environments.

What really impressed me was how well Rails 8 works in Kubernetes. The SolidQueue and SolidCache features reduce infrastructure complexity significantly—no Redis to manage means fewer moving parts, simpler networking, and lower resource usage on the Pi cluster.

And honestly? Seeing background jobs process on dedicated worker pods while the web server handles HTTP requests, all automatically distributed by Kubernetes services, with health checks ensuring everything stays healthy—that's pretty cool.

**This is just the beginning.** Right now I'm using `kubectl port-forward` to access the app, which isn't practical for long-term use. In upcoming posts, I'll be adding Ingress for proper external access, persistent storage for file uploads, monitoring with Mission Control Jobs, and possibly SSL/TLS with cert-manager. But the foundation is solid, and I have a working Rails cluster to build on.

## What's Next?

I have a working Rails cluster, but there's more to do. In future posts, I'll be exploring:

- **Traefik Ingress** for proper external HTTP access (no more port-forward!)
- **Persistent storage for uploads** using NFS or Longhorn
- **Mission Control Jobs** for monitoring background jobs via web UI
- **SSL/TLS with cert-manager** and Let's Encrypt
- **Monitoring with Prometheus and Grafana** on the Pi cluster
- **CI/CD pipelines** with GitHub Actions deploying to k3s
- **Database backups** and disaster recovery

The foundation is solid. Now it's time to make it production-ready.

Stay tuned!

---

## Series Navigation

**Part 1:** [Deploy Your First App](/posts/kubernetes-learning-path-deploy-your-first-app/)  
**Part 2:** [ConfigMaps and Secrets](/posts/kubernetes-learning-path-configmaps-and-secrets/)  
**Part 3:** [Understanding Namespaces](/posts/kubernetes-learning-path-namespaces/)  
**Part 4:** [Understanding Port Mapping in k3d](/posts/kubernetes-learning-path-port-mapping/)  
**Part 5:** [Setting Up k3s on Raspberry Pi](/posts/kubernetes-learning-path-k3s-on-raspberry-pi/)  
**Part 6:** Deploying Rails 8 with SolidQueue on k3s ← You just finished this!  
**Part 7:** Persistent Storage (Coming soon)

---

Found a mistake or have questions? Feel free to [open an issue here](https://github.com/shairyar/shairyar.github.io/issues/new).

