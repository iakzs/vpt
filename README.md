# VPT (Virtual Private Tunnel)

VPT is an open-source, multi-node configuration-as-code tunneling platform. It allows developers to securely expose and manage local applications running behind NATs or private networks to a public cloud VPS relay—built entirely on top of a dynamic Caddy data-plane.

Think of it like an open-source, community-driven alternative to Cloudflare Tunnels or ngrok!

---

## The Vision & Feature Set

VPT is designed to bridge the gap between low-level networking multiplexing and modern GitOps/DevOps workflows. Target architectural blueprint includes:

### Infrastructure & Core Routing
* **Tunnel Federation:** Multi-node infrastructure supporting multiple VPS regions.
* **Smart Protocol Routing:** Multi-stream HTTP/WebSocket routing alongside raw Layer 4 TCP tunneling (for SSH, databases, game servers, and panels).
* **Advanced Edge Mapping:** Wildcard, path-based, and hostname-based private routing.
* **Resilient Connectivity:** Automated exponential connection backoff with active/passive health checks and connection quality tracking.

### Team & GitOps Workflows
* **Config as Code:** Single-file cluster setup via a unified `vpt.yml` structure.
* **GitHub Integration:** Automatic, temporary preview route allocation tied to pull requests with strict TTL expiration rules.
* **On-Demand Failover:** Automatic blue-green target switching and dynamic load balancing policies.
* **Built-in Access Layer:** Per-route authentication middleware (OIDC, GitHub, CIDR IP-locking) terminating right at the edge.

### Observability & DX
* **Real-time Metrics:** Live request streaming logs, latency graphs, and bandwidth/connection counters per route.
* **Rich UI Elements:** Beautiful terminal output via CLI/TUI wizards alongside a Flutter-powered mobile companion app for on-the-go tunnel health tracking.
* **Zero-Friction Deployment:** One-click automated DNS/wildcard certificate handling with out-of-the-box Docker and systemd installation helpers.

---

## The Design Blueprint (`vpt.yml`)

The snippet below maps out my idealized project configuration. 

*(Note: The `vpt.live` domains used here are entirely for architectural illustration. This structure represents my "perfect-case" target architecture and may evolve as code implementation begins.)*

```yaml
version: "1"
project: "VPT"

environment:
  name: "hackclub-shared-relay"
  auth:
    provider: "hackclub"
    # The control plane uses this token to authorize subdomain creation on the shared VPS
    token_env: "TOKEEEEEN"

# Shared Access Policies / Built-in Access Layer
policies:
  - name: internal-team
    type: oidc
    provider: github
    allowed_orgs: ["iavpt"]
  - name: ip-lock
    type: cidr
    allow: ["192.168.1.0/24", "10.0.0.0/8"]
  - name: hackclub-lock
    type: oauth
    allow: ["user1", "user2"]

# Tunnel definitions mapped to local infrastructure targets
tunnels:
  - name: main-app-server
    origin_lock: true # mTLS / Header check validation enabled
    reconnect_backoff: "exponential"
    
    routes:
      # Wildcard & Path-Based Routing with Auto HTTPS
      - hostname: "*-dev.vpt.live"
        path: "/api"
        target: "http://localhost:8080"
        websocket: true
        auth_policy: "internal-team"
        load_balancing: "round_robin"
        health_check:
          active: true
          path: "/healthz"
          interval: "10s"

      # Temporary Preview Route with Auto-Expiration
      - hostname: "preview-feat-xyz.vpt.live"
        target: "http://localhost:3000"
        expires_at: "2026-06-01T00:00:00Z"
        github_integration:
          repo: "iakzs/vpt"
          pr: 42

      # TCP Tunneling for backend systems
      - name: secure-ssh
        type: tcp
        listen_port: 2222 # Opened on the VPS relay
        target: "localhost:22"
        auth_policy: "ip-lock"

      # Private Network routing mapping via CIDR
      - name: internal-subnet
        type: cidr
        cidr: "10.10.0.0/16"

      # This subdomain is granted because of the auth block above
      - hostname: "cool-project.vpt.live"
        target: "http://localhost:3000"
        
      # Automatically spins up when a PR is opened in the test repo
      - hostname: "pr-*.preview.vpt.live"
        target: "http://localhost:3000"
        github_integration:
          repo: "iavpt/test"
