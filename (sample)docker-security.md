# Docker Security Hardening Guide

## Step 1: Stop Running as Root

This is the single most impactful change you can make.

By default, processes inside Docker containers run as root. Not a limited user. Actual root with UID 0.

**Why does this matter?**

If an attacker exploits your application and gets code execution, they have root access inside the container. They can read any file, modify anything, and if combined with other vulnerabilities, potentially escape to the host.

Running as a non-root user doesn't prevent the initial exploit. But it severely limits what an attacker can do afterward.

**How to fix it:**

Create a dedicated user in your Dockerfile and switch to it:

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Create a non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy files and set ownership
COPY --chown=appuser:appgroup . .

# Install dependencies as root (if needed), then switch
RUN npm ci --only=production

# Switch to non-root user
USER appuser

CMD ["node", "server.js"]
```

The key points:

- `addgroup` and `adduser` create a system group and user
- `-S` flag creates a system account (no password, no home directory clutter)
- `--chown` flag sets proper ownership when copying files
- `USER` instruction switches all subsequent commands to that user

Some applications complain about permission issues after this change. That's usually because they're trying to write to directories they shouldn't. Fix the app, don't remove the security.

**Quick check:**

Run your container and verify:

```bash
docker run myapp whoami
```

If it says `root`, you have work to do.

## Step 2: Use Minimal Base Images

Every package in your container is potential attack surface. More software means more vulnerabilities, more CVEs to patch, more ways in.

I regularly see containers based on full Ubuntu or Debian images. Hundreds of packages installed. Shells, compilers, network tools, stuff the application never uses but attackers love.

**The fix:**

Use minimal base images.

**Alpine Linux:**

```dockerfile
FROM node:20-alpine
```

Alpine is tiny, around 5MB base. It uses musl libc instead of glibc, which occasionally causes compatibility issues, but for most applications it works perfectly.

**Distroless images:**

```dockerfile
FROM gcr.io/distroless/nodejs20-debian12
```

Distroless takes it further. No shell, no package manager, nothing except your application and its runtime dependencies. An attacker who gets code execution can't even run basic commands because there's no shell to run them with.

**Slim variants:**

```dockerfile
FROM python:3.12-slim
```

If Alpine is too restrictive, slim variants strip out unnecessary packages while keeping glibc compatibility.

**The impact:**

That 1.2GB Ubuntu-based image? Often compresses to under 100MB with Alpine. Fewer packages, fewer vulnerabilities, faster pulls, smaller attack surface.

## Step 3: Scan Your Images for Vulnerabilities

Your base image has vulnerabilities. Your dependencies have vulnerabilities. This isn't pessimism, it's reality.

The question is whether you know about them before attackers do.

**Built-in scanning with Docker Scout:**

```bash
docker scout cves myapp:latest
```

This shows you known vulnerabilities in your image, sorted by severity.

**Trivy (free and thorough):**

```bash
trivy image myapp:latest
```

Trivy scans for OS package vulnerabilities, application dependencies, misconfigurations, and secrets accidentally baked into images.

**What to do with results:**

You'll probably see a wall of CVEs the first time. Don't panic. Focus on:

- Critical and high severity issues first
- Vulnerabilities in packages your application actually uses
- Issues with known exploits in the wild

Update your base image, update your dependencies, rebuild. Many vulnerabilities disappear just by using current versions.

**Automate it:**

Put scanning in your CI pipeline. Block deployments if critical vulnerabilities are found. This isn't optional anymore — it's basic hygiene.

```bash
# Example GitHub Actions step
- name: Scan image
  run: |
    trivy image --exit-code 1 --severity CRITICAL myapp:${{ github.sha }}
```

## Step 4: Never Put Secrets in Images

I still find hardcoded secrets in Dockerfiles. API keys in ENV statements. Passwords in build args. Private keys copied into images.

This isn't just bad practice. It's a ticking time bomb.

Anyone with access to the image can extract those secrets. They're visible in image history, stored in your registry, cached in CI systems. Even if you delete them in a later layer, they exist in previous layers.

**What not to do:**

```dockerfile
# NEVER do this
ENV DATABASE_PASSWORD=supersecretpassword
ENV AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG

# Also bad - build args are visible in image history
ARG API_KEY
ENV API_KEY=$API_KEY
```

**How to handle secrets properly:**

**Runtime environment variables:**

Pass secrets when starting the container, not when building it:

```bash
docker run -e DATABASE_PASSWORD="$DATABASE_PASSWORD" myapp
```

Better yet, use an env file that's not committed to git:

```bash
docker run --env-file .env.production myapp
```

**Docker secrets (for Swarm):**

```bash
echo "mysecretpassword" | docker secret create db_password -
```

Then in your compose file:

```yaml
services:
  app:
    secrets:
      - db_password
secrets:
  db_password:
    external: true
```

**BuildKit secret mounts (for build-time secrets):**

Sometimes you need secrets during build, like private npm registry tokens. BuildKit handles this securely:

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

The secret is available during that specific RUN command but never persisted in any layer.

**External secret managers:**

For production, use proper secret management:

- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Google Secret Manager

Your application fetches secrets at runtime. Nothing sensitive in the image.

## Step 5: Make Your Filesystem Read-Only

If an attacker gets into your container, one of the first things they'll try is writing files. Dropping malware, modifying application code, creating persistence mechanisms.

A read-only filesystem stops this cold.

**How to enable it:**

```bash
docker run --read-only myapp
```

In Docker Compose:

```yaml
services:
  app:
    image: myapp
    read_only: true
```

**Handling applications that need to write:**

Most applications need to write somewhere — temp files, caches, logs. Create specific writable directories using tmpfs mounts:

```bash
docker run --read-only --tmpfs /tmp --tmpfs /app/cache myapp
```

In Compose:

```yaml
services:
  app:
    image: myapp
    read_only: true
    tmpfs:
      - /tmp
      - /app/cache
```

The application can write to `/tmp` and `/app/cache`, but those are in-memory filesystems that don't persist. Everything else is locked down.

**Why this matters:**

Even if an attacker gets code execution, they can't:

- Modify your application code
- Drop persistent malware
- Change configuration files
- Create new executable files

They're stuck with whatever is already in the container.

## Step 6: Drop Unnecessary Capabilities

Linux capabilities are a way to give processes specific root-like powers without full root access. Docker containers start with a reduced set, but it's still more than most applications need.

For example, by default containers can:

- Change file ownership (`CAP_CHOWN`)
- Bypass file permission checks (`CAP_DAC_OVERRIDE`)
- Bind to privileged ports (`CAP_NET_BIND_SERVICE`)

Your Node.js API probably doesn't need any of these.

**Drop all capabilities, add back only what's needed:**

```bash
docker run --cap-drop=ALL myapp
```

If your application needs specific capabilities, add only those:

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

In Compose:

```yaml
services:
  app:
    image: myapp
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

**What about `--privileged`?**

Never use `--privileged` in production. Ever.

```bash
# DON'T DO THIS
docker run --privileged myapp
```

This flag gives the container almost complete access to the host. It's occasionally needed for specific development scenarios like running Docker inside Docker, but in production it's essentially disabling container isolation entirely.

## Step 7: Set Resource Limits

A container without resource limits can consume all available CPU and memory on the host. This isn't just a performance concern — it's a security issue.

An attacker can intentionally exhaust resources (denial of service). A crypto miner can peg your CPUs. A memory leak can take down other containers on the same host.

**Set memory limits:**

```bash
docker run --memory=512m --memory-swap=512m myapp
```

The `--memory-swap` equal to `--memory` prevents swap usage, which is usually what you want.

In Compose:

```yaml
services:
  app:
    image: myapp
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
```

**Set CPU limits:**

```bash
docker run --cpus=1.5 myapp
```

This limits the container to 1.5 CPU cores.

**Why this matters:**

If a container gets compromised, resource limits contain the damage. The attacker can't mine Bitcoin using your entire server's capacity. They can't crash other services through resource exhaustion.

## Step 8: Isolate Your Networks

By default, containers on the same Docker network can talk to each other freely. Your frontend can reach your database directly. Every container can reach every other container.

In production, this is too permissive.

**Create separate networks:**

```yaml
services:
  frontend:
    networks:
      - frontend-net

  backend:
    networks:
      - frontend-net
      - backend-net

  database:
    networks:
      - backend-net

networks:
  frontend-net:
  backend-net:
```

Now the frontend can reach the backend, and the backend can reach the database, but the frontend can't directly access the database.

If the frontend gets compromised, the attacker can't immediately pivot to the database.

**Disable inter-container communication when not needed:**

For truly isolated containers:

```bash
docker network create --driver bridge -o com.docker.network.bridge.enable_icc=false isolated-net
```

Containers on this network can only communicate with the outside world, not each other.

## Step 9: Use Security Profiles

Docker supports security frameworks like Seccomp and AppArmor that restrict what system calls containers can make.

**Seccomp (Secure Computing Mode):**

By default, Docker applies a seccomp profile that blocks about 44 dangerous system calls. You can make this stricter.

```bash
docker run --security-opt seccomp=custom-profile.json myapp
```

For most applications, the default profile is good. Don't disable it:

```bash
# DON'T DO THIS
docker run --security-opt seccomp=unconfined myapp
```

**AppArmor:**

AppArmor profiles restrict file access, network access, and capabilities. Docker applies a default profile, but you can create custom ones.

```bash
docker run --security-opt apparmor=docker-custom myapp
```

**No new privileges:**

This prevents processes from gaining additional privileges through setuid binaries or other mechanisms:

```bash
docker run --security-opt=no-new-privileges:true myapp
```

In Compose:

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
```

This should be enabled for almost every production container.

## Step 10: Enable Docker Content Trust

When you pull an image, how do you know it's the real thing? How do you know it hasn't been tampered with?

Docker Content Trust uses digital signatures to verify image publishers.

**Enable it:**

```bash
export DOCKER_CONTENT_TRUST=1
```

With this enabled, Docker will only pull signed images. If an image isn't signed, the pull fails.

**For your own images:**

Sign them when pushing:

```bash
docker trust sign myregistry/myapp:v1.2.3
```

This requires setting up signing keys, but it ensures that images in your registry are verified.

## Step 11: Keep Everything Updated

Outdated Docker daemon, outdated base images, outdated application dependencies — they all accumulate vulnerabilities over time.

**Update Docker Engine:**

```bash
# Check current version
docker version

# Update (varies by installation method)
sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io
```

**Update base images regularly:**

Pin versions, but update those pins monthly or quarterly.

**Automate dependency updates:**

Use tools like Dependabot or Renovate to automatically create PRs when dependencies have security updates.

## A Complete Hardened Dockerfile Example

Putting it all together:

```dockerfile
# Use specific, minimal base image
FROM node:20.11.1-alpine3.19 AS builder

WORKDIR /app

# Copy dependency files first (layer caching)
COPY package.json package-lock.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Build if needed
RUN npm run build

# Production stage - even smaller image
FROM node:20.11.1-alpine3.19

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

WORKDIR /app

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy built application with proper ownership
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

# Set environment
ENV NODE_ENV=production

# Use non-root user
USER appuser

# Expose port (documentation)
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

**Running it with security options:**

```bash
docker run -d \
  --name myapp \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --security-opt=no-new-privileges:true \
  --memory=512m \
  --cpus=1 \
  --user appuser \
  -p 3000:3000 \
  myapp:v1.2.3
```

## Security Checklist

Before deploying any container to production, verify:

- [ ] Running as non-root user
- [ ] Using minimal base image (alpine, distroless, slim)
- [ ] No secrets in Dockerfile or image layers
- [ ] Image scanned for vulnerabilities
- [ ] Resource limits set (memory, CPU)
- [ ] Read-only filesystem where possible
- [ ] Capabilities dropped (at minimum `--cap-drop=ALL` then add back what's needed)
- [ ] No privileged mode
- [ ] `no-new-privileges` enabled
- [ ] Health checks configured
- [ ] Networks isolated appropriately
- [ ] Base images and dependencies regularly updated
