# frappe-lms-docker

Not-Production-ready Docker deployment for [Frappe LMS](https://github.com/frappe/lms) with all necessary patches and fixes applied.
DISCLAIMER: I Created this with Claude Pro, it was just meant to be a fix for my personal problem with installing frappe lms, i wont update this and i dont claim any rights on the product. I just thought this could help smb else out. Please check the code yourself and make backups!

## What's included

- **Frappe LMS** (`main` branch) + **Payments app** (`version-15`) bundled into a single custom Docker image
- **Razorpay fix** — installs `razorpay>=2.0.0` and `setuptools` to resolve `pkg_resources` errors
- **nginx DNS resolver fix** — uses Docker's internal DNS (`127.0.0.11`) for dynamic upstream resolution, preventing 502 errors after container restarts
- **host_name fix** — automatically sets the correct `host_name` during deployment so Frappe never redirects to `:8080`
- **SITES_RULE fix** — patches `easy-install.py` to include the missing `SITES_RULE` env variable required by Traefik

---

## Requirements

- Docker + Docker Compose
- Python 3.x
- A reverse proxy handling SSL (e.g. Nginx Proxy Manager) — this setup runs HTTP on port 8080

---

## Files

| File | Description |
|------|-------------|
| `apps.json` | Defines the apps baked into the custom image (payments + lms) |
| `images/custom/Containerfile` | Patched Containerfile with razorpay fix |
| `resources/core/nginx/nginx-template.conf` | Patched nginx template with Docker DNS resolver |
| `compose.yaml` | Patched compose template with host_name fix in configurator |
| `easy-install.py` | Patched easy-install script with SITES_RULE fix |

---

## Build the custom image

```bash
cd /frappe/frappe_docker

docker build \
    --progress=plain \
    --tag=ghcr.io/local/lms-with-payments:latest \
    --file=images/custom/Containerfile \
    --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
    --build-arg=FRAPPE_BRANCH=version-15 \
    --build-arg=PYTHON_VERSION=3.11.6 \
    --build-arg=NODE_VERSION=20.19.0 \
    "--build-arg=APPS_JSON_BASE64=$(base64 -w 0 /frappe/apps.json)" \
    .
```

---

## Deploy

```bash
docker compose -p learning_prod_setup down -v
rm ~/learning_prod_setup.env

python3 /frappe/easy-install.py deploy \
    --project=learning_prod_setup \
    --email=your@email.com \
    --image=ghcr.io/local/lms-with-payments \
    --version=latest \
    --app=lms \
    --sitename your-domain.com \
    --no-ssl \
    --http-port=8080
```

---

## Post-deploy steps

These run automatically via the patched `compose.yaml`. If needed manually:

```bash
# Enable scheduler
docker compose -p learning_prod_setup exec backend \
    bench --site your-domain.com enable-scheduler

# Set common_site_config
docker compose -p learning_prod_setup exec backend bash -c \
    'echo "{\"db_host\": \"db\", \"redis_cache\": \"redis://redis-cache:6379\", \"redis_queue\": \"redis://redis-queue:6379\", \"redis_socketio\": \"redis://redis-queue:6379\"}" \
    > /home/frappe/frappe-bench/sites/common_site_config.json'

# Set host_name (prevents :8080 redirect)
docker compose -p learning_prod_setup exec backend \
    bench --site your-domain.com set-config host_name "https://your-domain.com"

# Restart services
docker compose -p learning_prod_setup restart backend queue-short queue-long scheduler websocket frontend
```

---

## Nginx Proxy Manager config

Forward to: `<server-ip>:8080`  
WebSockets: **enabled**  
SSL: Let's Encrypt on NPM side

---

## Known issues fixed

### 502 after container restart
The original nginx config uses `upstream` blocks that cache backend IPs at startup. After a backend restart the IP changes but nginx still uses the old one → 502.

**Fix:** Replaced `upstream` blocks with `set $var` + `resolver 127.0.0.11 valid=10s` so nginx re-resolves the hostname on every request using Docker's internal DNS.

### Frappe redirects to :8080
Frappe uses `host_name` from site config to build redirects. If not set it appends the internal port.

**Fix:** Added `bench --site <site> set-config host_name "https://<domain>"` to the configurator command in `compose.yaml`.

### Missing payments app
The official `ghcr.io/frappe/lms:stable` image does not include the `payments` app which LMS depends on.

**Fix:** Custom image built with `apps.json` including both `payments` and `lms`.

### razorpay pkg_resources error
`razorpay` uses the deprecated `pkg_resources` API from `setuptools` which is no longer included by default.

**Fix:** Added `pip install "razorpay>=2.0.0" "setuptools"` to the Containerfile build step.
