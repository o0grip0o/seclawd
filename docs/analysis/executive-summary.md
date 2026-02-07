# Executive Summary: OpenClaw Platform Analysis

> **Date:** 2026-02-06
> **Scope:** Rust translation feasibility, security hardening, Docker containerization strategy
> **Status:** Initial Analysis & Planning Phase

---

## Table of Contents

1. [Platform Overview](#platform-overview)
2. [Rust Translation Assessment](#rust-translation-assessment)
3. [Security Hardening Assessment](#security-hardening-assessment)
4. [Docker Containerization Strategy](#docker-containerization-strategy)
5. [Cross-Cutting Concerns](#cross-cutting-concerns)
6. [Recommendations & Roadmap](#recommendations--roadmap)

---

## 1. Platform Overview

### What OpenClaw Is

OpenClaw is a **personal AI assistant platform** that connects to 17+ messaging channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Microsoft Teams, Matrix, etc.) and executes LLM-powered agent workflows with tool use and sandboxed execution.

### Architecture at a Glance

| Layer | Technology | LOC (est.) |
|-------|-----------|------------|
| Core runtime | TypeScript / Node.js 22+ | ~150K |
| Gateway server | Hono + WebSocket (ws) | ~30K |
| Agent engine | Pi framework (Anthropic Claude) | ~40K |
| CLI | Commander.js + Clack | ~20K |
| Extensions (30+) | TypeScript plugins (pnpm workspaces) | ~60K |
| Native apps | Swift (iOS/macOS), Kotlin (Android) | ~50K |
| Tests | Vitest (V8 coverage) | ~80K |
| **Total** | | **~300K-400K** |

### Key Dependencies

- **LLM clients:** Anthropic Pi SDK, OpenAI SDK, AWS Bedrock SDK
- **Messaging SDKs:** Baileys (WhatsApp), Grammy (Telegram), @slack/bolt, Discord API types
- **Native addons:** Sharp (image processing), sqlite-vec (vector search), node-pty (terminal)
- **Infrastructure:** Hono (HTTP), ws (WebSocket), AJV + Zod (validation), tslog (logging)

---

## 2. Rust Translation Assessment

### 2.1 Executive Summary

A full rewrite of OpenClaw from TypeScript to Rust is **not recommended as a monolithic effort**. The codebase is tightly coupled to Node.js ecosystem APIs and relies on several JavaScript-only SDKs (notably Baileys for WhatsApp) with no mature Rust equivalents. A **hybrid approach** - Rust core with Node.js compatibility layer - is the most viable path.

### 2.2 Opportunities

| Opportunity | Impact | Description |
|------------|--------|-------------|
| **Performance** | High | Gateway server, message routing, and agent orchestration would benefit from Rust's zero-cost abstractions and predictable latency |
| **Memory safety** | High | Eliminates classes of runtime errors (null/undefined, type coercion) at compile time |
| **Binary distribution** | High | Single static binary simplifies deployment (no Node.js runtime dependency) |
| **Concurrency** | High | Tokio async runtime offers superior WebSocket and I/O concurrency vs. Node.js event loop |
| **Well-typed protocol** | Low friction | TypeBox/JSON schemas in `src/gateway/protocol/` map cleanly to Rust structs via `serde` |
| **Existing Rust SDKs** | Moderate | Telegram (`teloxide`), Discord (`serenity`/`twilight`), Matrix (`matrix-sdk`) have mature Rust libraries |

### 2.3 Blockers

| Blocker | Severity | Description |
|---------|----------|-------------|
| **WhatsApp SDK (Baileys)** | Critical | Baileys is JavaScript-only (browser protocol reverse-engineering). No equivalent Rust library exists. This is the flagship channel and powers the primary use case. |
| **Plugin system** | Critical | Current system uses `jiti` for runtime TypeScript evaluation. Rust cannot dynamically load `.ts` files. Requires fundamental re-architecture to WASM plugins, shared libraries (.so/.dylib), or gRPC boundaries. |
| **Native addon parity** | High | Sharp (libvips), sqlite-vec (NAPI), node-pty have no drop-in Rust replacements. `image` crate is less feature-rich than Sharp; PTY crates are less mature than node-pty. |
| **Ecosystem breadth** | High | Slack, Line, Feishu/Lark, Zalo SDKs have no mature Rust equivalents. Each would require custom HTTP client implementations. |
| **Async model gap** | Moderate | 150+ files use Node.js async/await, streams, and event emitters. Migration to Tokio is feasible but requires careful refactoring of every stream consumer and WebSocket handler. |
| **Dynamic JSON handling** | Moderate | ~600+ `JSON.parse`/`stringify` calls across agent tools and config. Rust's `serde_json::Value` handles this, but loses compile-time type checking for dynamic payloads. |
| **macOS/iOS integration** | Moderate | Swift native apps communicate via Protocol Buffers. Rust gateway would need new FFI bindings or maintain the existing protocol layer. |

### 2.4 Effort Estimate

| Approach | Timeline | Team Size | Risk |
|----------|----------|-----------|------|
| Full rewrite (monolithic) | 18-24 months | 6-8 engineers | Very High |
| Hybrid core (Rust gateway + Node.js plugins) | 9-12 months | 4-6 engineers | Moderate |
| Incremental (Rust NAPI modules) | 6-9 months | 2-3 engineers | Low |

### 2.5 Recommended Approach: Hybrid Architecture

```
+---------------------------------------------+
|  Native Apps (Swift / Kotlin)               |
+-------------------+-------------------------+
                    | Protocol Buffers / WebSocket
+-------------------v-------------------------+
|  Rust Core (Gateway + Agent Orchestrator)   |
|  - WebSocket server (tokio-tungstenite)     |
|  - Message routing & session management     |
|  - Auth & token management                  |
|  - Config & secret handling                 |
|  - Protocol validation (serde)              |
+-------------------+-------------------------+
                    | FFI / gRPC / NAPI
+-------------------v-------------------------+
|  Node.js Plugin Layer                       |
|  - Baileys (WhatsApp)                       |
|  - Channel extensions (30+)                 |
|  - LLM provider SDKs                        |
|  - Media pipeline (Sharp, Playwright)       |
+---------------------------------------------+
```

**Phase 1 (months 1-3):** Rewrite gateway server and protocol layer in Rust. Keep all plugins in Node.js. Communication via Unix domain sockets or gRPC.

**Phase 2 (months 4-6):** Migrate agent orchestration, session management, and config to Rust. Introduce WASM plugin interface for new extensions.

**Phase 3 (months 7-12):** Migrate channels with Rust SDKs (Telegram, Discord, Matrix). Maintain Node.js bridge for Baileys and SDKs without Rust equivalents.

---

## 3. Security Hardening Assessment

### 3.1 Executive Summary

OpenClaw has a **solid security foundation** with defense-in-depth patterns already in place: timing-safe auth, tool permission profiles, sandbox path validation, environment variable blocklists, and input schema validation. However, several areas require hardening to meet enterprise security standards and address emerging AI-specific threats (prompt injection, tool abuse, data exfiltration).

### 3.2 Current Security Posture

| Area | Status | Details |
|------|--------|---------|
| Authentication | Strong | Timing-safe token comparison, multi-mode auth (token/password/Tailscale/device-token) |
| Input validation | Strong | AJV schema validation on all 30+ gateway RPC methods; Zod for config |
| Tool permissions | Good | 4-tier profile model (minimal/coding/messaging/full), owner-only tools, exec allowlists |
| Secret management | Good | Env var loading with redaction, detect-secrets CI scanning, baseline tracking |
| Network binding | Good | Loopback-first default, origin validation for browser requests |
| Sandbox isolation | Moderate | Path traversal protection, but Docker sandbox is optional (not enforced) |
| Prompt injection | Moderate | Basic sanitization exists, but no systematic prompt/response filtering |
| Dependency supply chain | Moderate | pnpm lock integrity, but no SBOM generation or dependency attestation |

### 3.3 Identified Vulnerabilities & Recommendations

#### 3.3.1 Prompt Injection (AI-Specific)

**Current state:** Basic text sanitization in `chat-sanitize.ts` and `sanitizeTextContent()`. No systematic prompt injection detection or mitigation framework.

**Recommendations:**

| # | Action | Priority | Effort |
|---|--------|----------|--------|
| PI-1 | Implement input/output filtering layer with known prompt injection patterns (jailbreak detection, instruction override detection) | High | Medium |
| PI-2 | Add structured prompt boundaries (system/user/assistant delimiters) that cannot be overridden by user input | High | Low |
| PI-3 | Implement output validation to detect data exfiltration attempts (URLs, encoded data in agent responses) | High | Medium |
| PI-4 | Add rate limiting per user/channel to prevent automated prompt injection attacks | Medium | Low |
| PI-5 | Log and alert on suspicious prompt patterns for post-incident analysis | Medium | Low |

#### 3.3.2 Tool Permission Hardening

**Current state:** Profile-based tool access with exec allowlists. However, the `full` profile grants unrestricted shell access.

**Recommendations:**

| # | Action | Priority | Effort |
|---|--------|----------|--------|
| TP-1 | Enforce Docker sandbox as mandatory for `full` and `coding` profiles (currently optional) | Critical | Medium |
| TP-2 | Implement tool invocation audit logging with tamper-evident storage | High | Medium |
| TP-3 | Add network egress controls for sandboxed tool execution (no arbitrary outbound connections) | High | Medium |
| TP-4 | Implement tool result size limits to prevent denial-of-service via large outputs | Medium | Low |
| TP-5 | Add time-bounded execution with hard kill for all tool invocations | Medium | Low |
| TP-6 | Implement least-privilege filesystem mounts (read-only workspace by default, explicit write grants) | Medium | Medium |

#### 3.3.3 Network & Transport Security

**Current state:** Loopback-first binding with browser origin checks. No TLS enforcement for LAN/Tailnet modes.

**Recommendations:**

| # | Action | Priority | Effort |
|---|--------|----------|--------|
| NT-1 | Enforce TLS for all non-loopback gateway connections (LAN, Tailnet modes) | Critical | Medium |
| NT-2 | Implement WebSocket connection rate limiting and max concurrent connection caps | High | Low |
| NT-3 | Add mutual TLS (mTLS) support for device-to-gateway authentication | Medium | High |
| NT-4 | Implement CORS headers for all HTTP endpoints (not just browser origin checks) | Medium | Low |
| NT-5 | Add Content-Security-Policy headers to the control UI | Medium | Low |

#### 3.3.4 Supply Chain Security

**Current state:** detect-secrets for secret scanning, pnpm lock integrity. No SBOM or attestation.

**Recommendations:**

| # | Action | Priority | Effort |
|---|--------|----------|--------|
| SC-1 | Generate SBOM (CycloneDX or SPDX) as part of CI/CD pipeline | High | Low |
| SC-2 | Enable npm provenance attestation for published packages | High | Low |
| SC-3 | Pin all GitHub Actions to SHA hashes (currently flagged by zizmor as disabled) | High | Low |
| SC-4 | Implement dependency review for PRs (auto-block known vulnerable additions) | Medium | Low |
| SC-5 | Add Sigstore signing for Docker images and release artifacts | Medium | Medium |

#### 3.3.5 Data Protection

**Current state:** Credentials stored in `~/.openclaw/credentials/`. Session logs in `~/.openclaw/sessions/`. No encryption at rest.

**Recommendations:**

| # | Action | Priority | Effort |
|---|--------|----------|--------|
| DP-1 | Encrypt credentials at rest using OS keychain (macOS Keychain, Linux Secret Service, Windows Credential Manager) | Critical | Medium |
| DP-2 | Implement session log encryption with per-user keys | High | Medium |
| DP-3 | Add configurable data retention policies with automatic purge | Medium | Medium |
| DP-4 | Implement PII detection and redaction in logs and session history | Medium | High |
| DP-5 | Add audit trail for all credential access and modification | Medium | Low |

#### 3.3.6 Runtime Security

**Current state:** Node.js 22.12.0+ required (CVE patches). Dangerous env var blocklist implemented.

**Recommendations:**

| # | Action | Priority | Effort |
|---|--------|----------|--------|
| RS-1 | Enable Node.js permission model (`--experimental-permission`) for production deployments | High | Medium |
| RS-2 | Implement process isolation between agent sessions (separate V8 isolates or worker threads) | High | High |
| RS-3 | Add memory limits per agent session to prevent resource exhaustion | Medium | Low |
| RS-4 | Implement syscall filtering (seccomp profiles) for Docker sandbox containers | Medium | Medium |
| RS-5 | Add integrity verification for plugin code before loading (hash or signature check) | Medium | Medium |

### 3.4 Compliance Mapping

| Standard | Relevant Controls | Current Coverage |
|----------|------------------|-----------------|
| OWASP LLM Top 10 (2025) | LLM01 (Prompt Injection), LLM02 (Insecure Output), LLM06 (Excessive Agency), LLM08 (Excessive Autonomy) | Partial |
| NIST AI RMF | Map 1.1 (AI risk identification), Measure 2.6 (AI security testing) | Not started |
| SOC 2 Type II | CC6.1 (Logical access), CC6.6 (External threats), CC7.2 (Monitoring) | Partial |
| CIS Docker Benchmark | 4.1 (Non-root), 4.5 (Content trust), 5.2 (Network segmentation) | Partial |

---

## 4. Docker Containerization Strategy

### 4.1 Executive Summary

OpenClaw already has **functional Docker infrastructure** (production Dockerfile, sandbox containers, docker-compose.yml, automated setup script). The path to a fully closed-loop containerized deployment requires extending the existing foundation with **orchestration, networking, monitoring, and security hardening**.

### 4.2 Current Docker Assets

| Asset | Status | Description |
|-------|--------|-------------|
| `Dockerfile` | Production-ready | Node 22, non-root user, multi-stage build |
| `Dockerfile.sandbox` | Functional | Minimal Debian image for agent tool execution |
| `Dockerfile.sandbox-browser` | Functional | Chromium + VNC + noVNC for browser automation |
| `docker-compose.yml` | Basic | Gateway + CLI services, volume mounts, port mapping |
| `docker-setup.sh` | Functional | Automated onboarding with token generation |
| `.dockerignore` | Present | Excludes node_modules, .git, apps, docs |

### 4.3 Target Architecture: Closed-Loop Containerized Deployment

```
+---------------------------------------------------------+
|                   Docker Compose / K8s                   |
+---------------------------------------------------------+
|                                                         |
|  +--------------+  +--------------+  +--------------+  |
|  |   Gateway    |  |   Agent      |  |   Browser    |  |
|  |   Container  |--|   Sandbox    |  |   Sandbox    |  |
|  |  (Hono + WS) |  | (Ephemeral)  |  | (Chromium)   |  |
|  | Port: 18789  |  | (Per-session) |  | Port: 9222   |  |
|  +------+-------+  +--------------+  +--------------+  |
|         |                                               |
|  +------v-------+  +--------------+  +--------------+  |
|  |  Control UI  |  |   Memory     |  |  Monitoring  |  |
|  |   (Web)      |  |   Store      |  |  (OTel +     |  |
|  | Port: 18790  |  | (SQLite-vec) |  |  Prometheus) |  |
|  +--------------+  +--------------+  +--------------+  |
|                                                         |
|  +--------------------------------------------------+  |
|  |              Shared Volumes                       |  |
|  |  config/ | credentials/ | sessions/ | workspace/ |  |
|  +--------------------------------------------------+  |
+---------------------------------------------------------+
```

### 4.4 Opportunities

| Opportunity | Impact | Description |
|------------|--------|-------------|
| **Reproducible deployments** | High | Identical environments across dev, staging, production |
| **Agent isolation** | High | Each agent session runs in an ephemeral container with resource limits |
| **Horizontal scaling** | High | Gateway can be scaled behind a load balancer; agent sandboxes scale independently |
| **Security boundary** | High | Network policies, read-only filesystems, capability restrictions per container |
| **Zero-downtime updates** | Medium | Rolling updates with health checks and graceful shutdown |
| **Multi-tenant support** | Medium | Container-level isolation enables safe multi-user deployment |
| **Observability** | Medium | Centralized logging, metrics, and tracing via OpenTelemetry sidecar |

### 4.5 Blockers & Challenges

| Blocker | Severity | Description | Mitigation |
|---------|----------|-------------|------------|
| **WhatsApp session state** | High | Baileys stores session state locally (auth keys, chat state). Containers are ephemeral - session state must be externalized. | Mount persistent volume for `~/.openclaw/sessions/`; implement session state backup/restore. |
| **Docker-in-Docker** | High | Agent sandbox containers are launched from the gateway container. Requires either Docker socket passthrough (security risk) or DinD sidecar. | Use Sysbox runtime or rootless Docker-in-Docker. Alternatively, use a container runtime like Podman with rootless mode. |
| **Native addon builds** | Moderate | Sharp, sqlite-vec, node-pty require native compilation. Multi-arch builds (amd64/arm64) need cross-compilation. | Use pre-built binaries where available; multi-stage builds with platform-specific base images. |
| **Host device access** | Moderate | iMessage bridge requires macOS host access. Signal requires local D-Bus. USB devices (for mobile bridges) need `--device` passthrough. | These channels cannot run in containers. Document as limitation; keep host-only channel list. |
| **Startup latency** | Low | Cold-starting agent sandbox containers adds latency to first tool invocation. | Implement container pool with pre-warmed sandboxes; use `--rm` for automatic cleanup. |
| **Image size** | Low | Full production image with all dependencies is large (~1.5GB+). Browser sandbox adds Chromium (~500MB+). | Multi-stage builds, distroless base images, separate images per service. |

### 4.6 Implementation Plan

#### Phase 1: Harden Existing Docker Setup (Weeks 1-3)

- [ ] Add health check endpoints to gateway container (`/healthz`, `/readyz`)
- [ ] Implement graceful shutdown handling (SIGTERM, drain connections, exit)
- [ ] Add resource limits (CPU, memory) to docker-compose.yml
- [ ] Enable read-only root filesystem with explicit tmpfs mounts
- [ ] Add seccomp and AppArmor profiles for gateway container
- [ ] Implement log rotation and structured JSON logging for Docker
- [ ] Pin base image digests (not just tags) for reproducibility

#### Phase 2: Agent Sandbox Orchestration (Weeks 4-6)

- [ ] Implement container pool manager for pre-warmed agent sandboxes
- [ ] Add network policies (deny egress by default, allowlist for LLM APIs)
- [ ] Implement per-session resource quotas (CPU, memory, disk, network)
- [ ] Add container lifecycle management (create, start, monitor, cleanup)
- [ ] Implement session state externalization (persistent volume or object store)
- [ ] Add container execution audit logging

#### Phase 3: Full Orchestration (Weeks 7-12)

- [ ] Create Kubernetes manifests (Deployment, Service, ConfigMap, Secret, PVC)
- [ ] Implement Helm chart for parameterized deployment
- [ ] Add horizontal pod autoscaler for gateway and sandbox pools
- [ ] Implement centralized monitoring (Prometheus + Grafana + OpenTelemetry)
- [ ] Add automated backup/restore for session state and configuration
- [ ] Create CI/CD pipeline for automated image builds, scanning, and deployment
- [ ] Implement blue-green or canary deployment strategy

### 4.7 Docker Security Checklist

| Control | Current | Target |
|---------|---------|--------|
| Non-root user | node:1000 | node:1000 |
| Read-only root filesystem | Not enforced | With tmpfs |
| Capability dropping | Documented, not enforced | In compose/k8s |
| Seccomp profile | Not configured | Custom profile |
| Network policies | Not configured | Deny-by-default |
| Image signing | Not implemented | Sigstore/Cosign |
| Vulnerability scanning | Not in CI | Trivy/Grype |
| Resource limits | Not set | CPU + memory caps |
| Secret injection | Env vars | Docker secrets or Vault |
| Health checks | Not implemented | Liveness + readiness |
| Base image pinning | Tag only | SHA digest |

---

## 5. Cross-Cutting Concerns

### 5.1 Testing Strategy Impact

| Change | Test Impact | Mitigation |
|--------|------------|------------|
| Rust core rewrite | All gateway/agent tests must be rewritten | Maintain protocol-level integration tests that work against both implementations during migration |
| Docker hardening | Existing Docker E2E tests need updating | Extend `vitest.gateway.config.ts` with container-specific test suite |
| Security hardening | New security tests needed | Add fuzzing (prompt injection patterns), penetration testing suite, and compliance checks |

### 5.2 Backward Compatibility

| Component | Risk | Mitigation |
|-----------|------|------------|
| Plugin SDK | High (if Rust core changes plugin interface) | Version the plugin SDK; maintain Node.js compatibility layer for existing plugins |
| Gateway protocol | Low (TypeBox schemas are well-defined) | Protocol versioning with backward-compatible evolution |
| Config format | Low | Keep YAML config format; add migration tooling if schema changes |
| Native apps | Medium (Swift/Kotlin communicate via protocol) | Protocol Buffers provide cross-language stability |

### 5.3 Operational Considerations

| Concern | Description | Recommendation |
|---------|-------------|----------------|
| Monitoring | No centralized observability | Implement OpenTelemetry (extension exists: `diagnostics-otel`) |
| Incident response | No automated alerting | Add health check failures to alerting pipeline |
| Disaster recovery | Session state not backed up | Implement automated backup with point-in-time recovery |
| Capacity planning | No resource usage metrics | Add Prometheus metrics for memory, CPU, connection count, message throughput |

---

## 6. Recommendations & Roadmap

### 6.1 Prioritized Recommendation Summary

| Priority | Initiative | Timeline | Impact |
|----------|-----------|----------|--------|
| 1 | **Security hardening** (prompt injection filters, mandatory sandbox, TLS, credential encryption) | Weeks 1-6 | Critical risk reduction |
| 2 | **Docker closed-loop** (health checks, resource limits, network policies, sandbox orchestration) | Weeks 1-12 | Deployment reliability |
| 3 | **Supply chain security** (SBOM, image signing, Action pinning, dependency review) | Weeks 2-4 | Compliance readiness |
| 4 | **Rust hybrid core** (gateway + agent orchestration in Rust, Node.js plugin bridge) | Months 3-12 | Performance + safety |
| 5 | **Full Rust migration** (channel SDKs, plugin system redesign) | Months 12-24 | Long-term architecture |

### 6.2 Go / No-Go Decision Points

| Decision | Criteria | When |
|----------|----------|------|
| Start Rust hybrid | WhatsApp bridge design validated; Tokio WebSocket prototype passes load test | Month 2 |
| Commit to full Rust | Rust gateway running in production for 30 days; plugin WASM interface proven | Month 6 |
| Abandon Rust for Baileys | No viable Rust WhatsApp solution after 3 months of investigation | Month 5 |
| K8s deployment | Gateway running stable in Docker Compose for 30 days with less than 0.1% error rate | Month 4 |

### 6.3 Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Baileys has no Rust equivalent | High | Critical | Maintain Node.js bridge; invest in HTTP API if WhatsApp releases one |
| Plugin ecosystem fragmentation | Medium | High | Dual SDK support (Node.js + WASM) during transition period |
| Docker-in-Docker security issues | Medium | High | Evaluate Sysbox, rootless Podman, or Firecracker micro-VMs |
| Team skill gap (Rust expertise) | Medium | Medium | Training investment; hire Rust-experienced engineers |
| Performance regression during migration | Low | Medium | Benchmark suite with automated regression detection |
| Breaking changes in messaging APIs | Medium | Medium | Version lock SDKs; automated API compatibility tests |

---

## Appendix A: File Structure Reference

```
src/
  agents/              # Agent engine (Pi embedded, auth, tools, sandbox)
  gateway/             # WebSocket server, RPC methods, protocol
  channels/            # Channel routing, allowlists, registry
  cli/                 # CLI commands, onboarding, doctor
  commands/            # Command implementations
  config/              # Config loading, validation
  infra/               # Runtime guards, env, ports, errors
  media/               # Image/PDF processing pipeline
  memory/              # Vector search, embeddings
  providers/           # LLM provider factories
  routing/             # Message routing logic
  sessions/            # Session persistence
  plugins/             # Plugin loader, discovery, SDK

extensions/            # 30+ channel/provider plugins
apps/                  # Native mobile/desktop apps
packages/              # Utility packages
scripts/               # Build, test, infra automation
docs/                  # Mintlify documentation
```

## Appendix B: Technology Mapping (TypeScript to Rust)

| TypeScript | Rust Equivalent | Maturity |
|-----------|----------------|----------|
| Hono (HTTP) | Axum / Actix-web | Mature |
| ws (WebSocket) | tokio-tungstenite | Mature |
| Commander.js (CLI) | clap | Mature |
| AJV (JSON Schema) | jsonschema | Mature |
| Zod (validation) | serde + validator | Mature |
| Sharp (images) | image + imageproc | Moderate |
| sqlite-vec | sqlite-vec (C lib + FFI) | Early |
| node-pty | portable-pty | Moderate |
| Grammy (Telegram) | teloxide | Mature |
| Discord.js types | serenity / twilight | Mature |
| Baileys (WhatsApp) | **None** | N/A |
| @slack/bolt | slack-morphism | Moderate |
| tslog (logging) | tracing | Mature |
| Vitest (testing) | cargo test + nextest | Mature |
| pnpm (package mgr) | cargo | Mature |

## Appendix C: Security Controls Matrix (OWASP LLM Top 10 2025)

| OWASP LLM Top 10 (2025) | Current State | Recommended Action |
|--------------------------|--------------|-------------------|
| LLM01: Prompt Injection | Basic sanitization | Implement systematic input/output filtering with known attack patterns |
| LLM02: Insecure Output Handling | Partial (channel-specific) | Add output validation layer before delivery to all channels |
| LLM03: Training Data Poisoning | N/A (uses external APIs) | Monitor for model behavior anomalies |
| LLM04: Model Denial of Service | No rate limiting | Add per-user/channel rate limits and token budget caps |
| LLM05: Supply Chain Vulnerabilities | detect-secrets + pnpm lock | Add SBOM, image signing, dependency attestation |
| LLM06: Sensitive Information Disclosure | Redaction in logs | Add PII detection, credential scrubbing in agent outputs |
| LLM07: Insecure Plugin Design | Profile-based permissions | Enforce sandbox, add plugin signature verification |
| LLM08: Excessive Agency | Tool profiles (4 tiers) | Enforce mandatory sandbox for coding/full profiles |
| LLM09: Overreliance | N/A (user responsibility) | Add confidence indicators to agent responses |
| LLM10: Model Theft | API key management | Encrypt credentials at rest, add key rotation automation |
